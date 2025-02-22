---
layout: blog_detail
title: "Accelerated Stable Diffusion with PyTorch 2"
author: Grigory Sizov, Michael Gschwind, Hamid Shojanazeri, Driss Guessous, Daniel Haziza, Christian Puhrsch
featured-img: "assets/images/stable-diffusion/summary_n_samples_1_n_iter_2_sd2.png"
---

**TL;DR:** PyTorch 2.0 nightly offers out-of-the-box performance improvement for Stable Diffusion 2.1 by using the new `torch.compile()` compiler and optimized implementations of Multihead Attention integrated with PyTorch 2.

## Introduction

Stable Diffusion (SD) is a great example of Generative AI, producing high quality images from text prompts. However, as well as for other diffusion-based models, its generation is rather slow, due to the iterative nature of the sampling process by which the images are produced. This makes it important to optimize the code running inside the sampling loop. 

We took [SD 2.1 from Stability AI](https://github.com/Stability-AI/stablediffusion) as a starting point and accelerated its text-to-image generation using two optimizations available in PyTorch 2: compilation and fast attention implementation. Together with a few minor memory processing improvements in the code these optimizations give up to 49% inference speedup relative to the original SD implementation without [xFormers](https://github.com/facebookresearch/xformers), and 39% inference speedup relative to using SD with xFormers (excluding the compilation time), depending on the GPU architecture and batch size. Importantly, the speedup comes without a need to install xFormers or any other extra dependencies. 

The table below shows the improvement in runtime between the original implementation with xFormers installed and our optimized version with PyTorch-integrated memory efficient attention (originally developed for and released in the [xFormers](https://github.com/facebookresearch/xformers) library)  and PyTorch compilation. The compilation time is excluded.

**Runtime improvement in % compared to original+xFormers**

See the absolute runtime numbers in section [“Benchmarking setup and results summary”](#benchmarking-setup-and-results-summary)


<table class="table">
<thead>
  <tr>
   <td><strong>GPU</strong>
   </td>
   <td><strong>Batch size 1</strong>
   </td>
   <td><strong>Batch size 2</strong>
   </td>
   <td><strong>Batch size 4</strong>
   </td>
  </tr>
</thead>
<tbody>
  <tr>
   <td><strong>P100 (no compilation)</strong>
   </td>
   <td>-3.8
   </td>
   <td>0.44
   </td>
   <td>5.47
   </td>
  </tr>
  <tr>
   <td><strong>T4</strong>
   </td>
   <td>2.12
   </td>
   <td>10.51
   </td>
   <td>14.2
   </td>
  </tr>
  <tr>
   <td><strong>A10</strong>
   </td>
   <td>-2.34
   </td>
   <td>8.99
   </td>
   <td>10.57
   </td>
  </tr>
  <tr>
   <td><strong>V100</strong>
   </td>
   <td>18.63
   </td>
   <td>6.39
   </td>
   <td>10.43
   </td>
  </tr>
  <tr>
   <td><strong>A100</strong>
   </td>
   <td>38.5
   </td>
   <td>20.33
   </td>
   <td>12.17
   </td>
  </tr>
</tbody>
</table>


One can notice the following:



* The improvements are significant for powerful GPUs like A100 and V100. For those GPUs the improvement is most pronounced for batch size 1
* For less powerful GPUs we observe smaller speedups (or in two cases slight regressions). The batch size trend is reversed here: improvement is larger for larger batches

In the following sections we describe the applied optimizations and provide detailed benchmarking data, comparing SD performance with various optimization features on/off.

Specifically, we benchmark 5 configurations and the plots below compare their absolute performance for different GPUs and batch sizes. For definitions of these configurations see section [“Benchmarking setup and results”](#benchmarking-setup-and-results-summary).


![Benchmark of Stable Diffusion 2 versions across GPU architectures, batch size 1](/assets/images/stable-diffusion/summary_n_samples_1_n_iter_2_sd2.png){:width="100%"}

![Benchmark of Stable Diffusion 2 versions across GPU architectures, batch size 2](/assets/images/stable-diffusion/summary_n_samples_2_n_iter_2_sd2.png){:width="100%"}

![Benchmark of Stable Diffusion 2 versions across GPU architectures, batch size 4](/assets/images/stable-diffusion/summary_n_samples_4_n_iter_2_sd2.png){:width="100%"}



If you prefer looking directly at the code, see the [Google Colab](https://colab.research.google.com/drive/1cSP5HoRZCbjH55MdYiRtxC_Q0obQQ5ZD?usp=sharing) which runs the benchmark on T4.

			


## Optimizations 

Here we’ll go into more detail about the optimizations introduced into the SD code. At the moment they rely on features only available in the nightlies, so we pinned the PyTorch version to a recent nightly (see [here](https://github.com/sgrigory/stablediffusion2/blob/0f6d17cb2602302bc0f5c7dee6825e4b49a85518/environment.yaml#L13-L15)). Once the PyTorch 2.0 release comes out, these optimizations won’t have to rely on nightlies any more. 


### Optimized Attention

One part of the code which we optimized was the scaled dot-product attention. Attention is known to be a heavy operation: naive implementation materializes the attention matrix, leading to time and memory complexity quadratic in sequence length. In Stable Diffusion attention (`CrossAttention`) appears as part of Transformer blocks in multiple parts of the U-Net. Since the U-Net runs at every sampling step, this becomes a critical point to optimize. In PyTorch 2 optimized attention implementation is integrated into `torch.nn.MultiheadAttention`, and so we used it to replace the [custom attention implementation](https://github.com/Stability-AI/stablediffusion/blob/d55bcd4d31d0316fcbdf552f2fd2628fdc812500/ldm/modules/attention.py#L145-L194) in `CrossAttention`.

The optimized implementation of attention was available already in PyTorch 1.13 (see [here](https://pytorch.org/blog/a-better-transformer-for-fast-transformer-encoder-inference/)) and widely adopted (see e.g. [HuggingFace transformers library example](https://medium.com/pytorch/bettertransformer-out-of-the-box-performance-for-huggingface-transformers-3fbe27d50ab2)). In particular, it integrates memory-efficient attention from the [xFormers](https://github.com/facebookresearch/xformers) library and flash attention from [https://arxiv.org/abs/2205.14135](https://arxiv.org/abs/2205.14135). PyTorch 2.0 expands this to additional attention functions such as cross attention and custom kernels for further acceleration, making it applicable to SD.

Flash attention is available on GPUs with compute capability SM 7.5 or SM 8.x - for example, on T4, A10, and A100, which are included in our benchmark (you can check compute capability of each NVIDIA GPU [here](https://developer.nvidia.com/cuda-gpus#compute)). However, in our tests on A100 the memory efficient attention performed better than flash attention for the particular case of SD, due to the small number of attention heads and small batch size.  PyTorch understands this and chooses memory efficient attention over flash attention for SD when both are available (see the logic [here](https://github.com/pytorch/pytorch/blob/d8e795ecd53670682bd3b2e5ff1f378402b147d5/aten/src/ATen/native/transformers/cuda/sdp_utils.h#L33-L71)). For full control over the attention backends (memory-efficient attention, flash attention, “vanilla math”, or any future ones), power users can enable and disable them manually with the help of the context manager [torch.backends.cuda.sdp_kernel](https://pytorch.org/docs/master/backends.html#torch.backends.cuda.sdp_kernel). 


### Compilation

Compilation is a [new feature of PyTorch 2.0](https://pytorch.org/get-started/pytorch-2.0/#user-experience), enabling significant speedups with a very simple user experience. To invoke the default behavior, simply wrap a PyTorch module or a function into `torch.compile`:


```
model = torch.compile(model)
```


PyTorch compiler then turns Python code into a set of instructions which can be executed efficiently without Python overhead. The compilation happens dynamically the first time the code is executed. With the default behavior, under the hood PyTorch utilized [TorchDynamo](https://pytorch.org/docs/master/dynamo/index.html) to compile the code and [TorchInductor](https://dev-discuss.pytorch.org/t/torchinductor-a-pytorch-native-compiler-with-define-by-run-ir-and-symbolic-shapes/747) to further optimize it. See [this tutorial](https://pytorch.org/tutorials/intermediate/dynamo_tutorial.html) for more details.

Although the one-liner above is enough for compilation, certain modifications in the code can squeeze a larger speedup. In particular, one should avoid so-called graph breaks - places in the code which PyTorch can’t compile. As opposed to previous PyTorch compilation approaches (like TorchScript), PyTorch 2 compiler doesn’t break in this case. Instead it falls back on eager execution - so the code runs, but with reduced performance. We introduced a few minor changes to the SD code to eliminate graph breaks ([here](https://github.com/Stability-AI/stablediffusion/compare/main...sgrigory:stablediffusion2:optimize-w-compile?expand=1#diff-db5d837c282869a3588a17885e0baec3e29bf0701af6f4f34774d7b94503f7d4R34) and [here](https://github.com/Stability-AI/stablediffusion/compare/main...sgrigory:stablediffusion2:optimize-w-compile?expand=1#diff-db5d837c282869a3588a17885e0baec3e29bf0701af6f4f34774d7b94503f7d4R336-R343)). See this [doc](https://pytorch.org/docs/master/dynamo/faq.html#identifying-the-cause-of-a-graph-break) to learn more about graph breaks and how to eliminate them.

Note that compilation [requires GPU compute capability >= SM 7.0](https://github.com/openai/triton/blob/b5d32896b1f89fc44a82f8df3bb010934c53f4f5/README.md?plain=1#L66-L68) to run in non-eager mode. This covers all GPUs in our benchmarks -  T4, V100, A10, A100 - except for P100 (see the [full list](https://developer.nvidia.com/cuda-gpus#compute)). 


### Other optimizations

In addition, we have improved efficiency of some memory operations - e.g. creating a tensor on GPU directly rather than creating it on CPU and later moving to GPU (see [here](https://github.com/Stability-AI/stablediffusion/compare/main...sgrigory:stablediffusion2:optimize-w-compile?expand=1#diff-7f9fb1cdee2602845e0a4ad2a62dfcf86d4a868490e0ff126e8a1d045106d065R166-R167) and [here](https://github.com/Stability-AI/stablediffusion/compare/main...sgrigory:stablediffusion2:optimize-w-compile?expand=1#diff-1cd43c874cb6fb8799d24ba5b9e9c2f8a9e058b976b93a09e409c9d5853884f2R150-R220)). The places where such optimizations were necessary were determined by line-profiling and looking at CPU/GPU traces and [Flame Graphs](https://github.com/brendangregg/FlameGraph).


## Benchmarking setup and results summary

We have two versions of SD code to compare: _original_ and _optimized_. On top of this, several optimization features (xFormers, PyTorch memory efficient attention, compilation) can be turned on/off. Overall, as mentioned in the introduction, we will be benchmarking 5 configurations:



* _Original code without xFormers_
* _Original code with xFormers_
* _Optimized code with vanilla math attention backend and no compilation_
* _Optimized code with memory-efficient attention backend and no compilation_
* _Optimized code with memory-efficient attention backend and compilation_

As the _original version_ we took the SD 2.1 release, and placed it [here](https://github.com/sgrigory/stablediffusion2/tree/cee9b9f057eeef4b481e138da9dbc4fe8ecb0cba) with minimal modifications necessary for benchmarking. It uses PyTorch 1.12 and a custom implementation of attention.

The _optimized version_ is the code living [here](https://github.com/sgrigory/stablediffusion2/tree/0f6d17cb2602302bc0f5c7dee6825e4b49a85518). It uses `nn.MultiheadAttention` in `CrossAttention` and PyTorch 2.0.0.dev20230111+cu117. It also has a few other minor optimizations in PyTorch-related code. 

Please see the appendix “Benchmarked versions definition” in [the companion page](/blog/performance-experiments-stable-diffusion/) for the precise definition of the 5 configurations and prompts triggering each of them.

The table below shows runtime of each version of the code in seconds, and the percentage improvement compared to the _original with xFormers_. The compilation time is excluded.

**Runtimes for batch size 1. In parenthesis - relative improvement with respect to the “Original with xFormers” row**


<table class="table">
  <tr>
   <td><strong>Configuration</strong>
   </td>
   <td><strong>P100</strong>
   </td>
   <td><strong>T4</strong>
   </td>
   <td><strong>A10</strong>
   </td>
   <td><strong>V100</strong>
   </td>
   <td><strong>A100</strong>
   </td>
  </tr>
  <tr>
   <td><strong>Original without xFormers</strong>
   </td>
   <td>30.4s (-19.3%)
   </td>
   <td>29.8s (-77.3%)
   </td>
   <td>13.0s (-83.9%)
   </td>
   <td>10.9s (-33.1%)
   </td>
   <td>8.0s (-19.3%)
   </td>
  </tr>
  <tr>
   <td><strong>Original with xFormers</strong>
   </td>
   <td><strong>25.5s</strong> (0.0%)
   </td>
   <td>16.8s (0.0%)
   </td>
   <td><strong>7.1s</strong> (0.0%)
   </td>
   <td>8.2s (0.0%)
   </td>
   <td>6.7s (0.0%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with vanilla math attention, no compilation</strong>
   </td>
   <td>27.3s (-7.0%)
   </td>
   <td>19.9s (-18.7%)
   </td>
   <td>13.2s (-87.2%)
   </td>
   <td>7.5s (8.7%)
   </td>
   <td>5.7s (15.1%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with mem. efficient attention, no compilation</strong>
   </td>
   <td>26.5s (-3.8%)
   </td>
   <td>16.8s (0.2%)
   </td>
   <td><strong>7.1s</strong> (-0.8%)
   </td>
   <td>6.9s (16.0%)
   </td>
   <td>5.3s (20.6%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with mem. efficient attention and compilation</strong>
   </td>
   <td>-
   </td>
   <td><strong>16.4s </strong>(2.1%)
   </td>
   <td>7.2s (-2.3%)
   </td>
   <td><strong>6.6s</strong> (18.6%)
   </td>
   <td><strong>4.1s</strong> (38.5%)
   </td>
  </tr>
</table>


**Runtimes for batch size 2**


<table class="table">
  <tr>
   <td><strong>Configuration</strong>
   </td>
   <td><strong>P100</strong>
   </td>
   <td><strong>T4</strong>
   </td>
   <td><strong>A10</strong>
   </td>
   <td><strong>V100</strong>
   </td>
   <td><strong>A100</strong>
   </td>
  </tr>
  <tr>
   <td><strong>Original without xFormers</strong>
   </td>
   <td>58.0s (-21.6%)
   </td>
   <td>57.6s (-84.0%)
   </td>
   <td>24.4s (-95.2%)
   </td>
   <td>18.6s (-63.0%)
   </td>
   <td>12.0s (-50.6%)
   </td>
  </tr>
  <tr>
   <td><strong>Original with xFormers</strong>
   </td>
   <td>47.7s (0.0%)
   </td>
   <td>31.3s (0.0%)
   </td>
   <td>12.5s (0.0%)
   </td>
   <td>11.4s (0.0%)
   </td>
   <td>8.0s (0.0%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with vanilla math attention, no compilation</strong>
   </td>
   <td>49.3s (-3.5%)
   </td>
   <td>37.9s (-21.0%)
   </td>
   <td>17.8s (-42.2%)
   </td>
   <td>12.7s (-10.7%)
   </td>
   <td>7.8s (1.8%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with mem. efficient attention, no compilation</strong>
   </td>
   <td><strong>47.5s </strong>(0.4%)
   </td>
   <td>31.2s (0.5%)
   </td>
   <td>12.2s (2.6%)
   </td>
   <td>11.5s (-0.7%)
   </td>
   <td>7.0s (12.6%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with mem. efficient attention and compilation</strong>
   </td>
   <td>-
   </td>
   <td><strong>28.0s</strong> (10.5%)
   </td>
   <td><strong>11.4s</strong> (9.0%)
   </td>
   <td><strong>10.7s </strong>(6.4%)
   </td>
   <td><strong>6.4s</strong> (20.3%)
   </td>
  </tr>
</table>


**Runtimes for batch size 4**


<table class="table">
  <tr>
   <td><strong>Configuration</strong>
   </td>
   <td><strong>P100</strong>
   </td>
   <td><strong>T4</strong>
   </td>
   <td><strong>A10</strong>
   </td>
   <td><strong>V100</strong>
   </td>
   <td><strong>A100</strong>
   </td>
  </tr>
  <tr>
   <td><strong>Original without xFormers</strong>
   </td>
   <td>117.9s (-20.0%)
   </td>
   <td>112.4s (-81.8%)
   </td>
   <td>47.2s (-101.7%)
   </td>
   <td>35.8s (-71.9%)
   </td>
   <td>22.8s (-78.9%)
   </td>
  </tr>
  <tr>
   <td><strong>Original with xFormers</strong>
   </td>
   <td>98.3s (0.0%)
   </td>
   <td>61.8s (0.0%)
   </td>
   <td>23.4s (0.0%)
   </td>
   <td>20.8s (0.0%)
   </td>
   <td>12.7s (0.0%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with vanilla math attention, no compilation</strong>
   </td>
   <td>101.1s (-2.9%)
   </td>
   <td>73.0s (-18.0%)
   </td>
   <td>28.3s (-21.0%)
   </td>
   <td>23.3s (-11.9%)
   </td>
   <td>14.5s (-13.9%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with mem. efficient attention, no compilation</strong>
   </td>
   <td><strong>92.9s </strong>(5.5%)
   </td>
   <td>61.1s (1.2%)
   </td>
   <td>23.9s (-1.9%)
   </td>
   <td>20.8s (-0.1%)
   </td>
   <td>12.8s (-0.9%)
   </td>
  </tr>
  <tr>
   <td><strong>Optimized with mem. efficient attention and compilation</strong>
   </td>
   <td>-
   </td>
   <td><strong>53.1s </strong>(14.2%)
   </td>
   <td><strong>20.9s</strong> (10.6%)
   </td>
   <td><strong>18.6s</strong> (10.4%)
   </td>
   <td><strong>11.2s</strong> (12.2%)
   </td>
  </tr>
</table>


To minimize fluctuations and external influence on the performance of the benchmarked code, we ran each version of the code one after another, and then repeated this sequence 10 times: A, B, C, D, E,  A, B, … So the results of a typical run would look like the one in the picture below. For results of all runs please see appendix “Per-run data” in [the companion page](/blog/performance-experiments-stable-diffusion/). Note that one shouldn’t rely on comparison of absolute run times between different graphs, but comparison of run times _inside_ one graph is pretty reliable, thanks to our benchmarking setup.

![Stable Diffusion 2.1 benchmarks](/assets/images/stable-diffusion/original_vs_optimized_a100_n_samples_1_n_iter_2_sd2.png){:width="80%"}


Each run of `txt2img.py` generates several batches, which is regulated by the CLI parameter `--n_iter`. In the benchmarks we used `n_iter = 2`, but introduced an additional “warm-up” iteration, which doesn’t contribute to the run time. This was necessary for the runs with compilation, because compilation happens the first time the code runs, and so the first iteration is much longer than all subsequent. To make comparison fair, we also introduced this additional “warm-up” iteration to all other runs, which is turned on by CLI option `--skip_first` provided to the modified `txt2img.py`.

The numbers in the table above are for number of iterations 2 (plus a “warm-up one”), prompt ”A photo”, seed 1, PLMS sampler, and autocast turned on. See [the companion page](/blog/performance-experiments-stable-diffusion/) for precise CLI commands in appendix “Benchmarked versions definition” and detailed results of individual runs in appendix “Per-run data”.

The P100, V100, and A100 benchmarks were done on Meta internal infrastructure. The T4 benchmarks were done in Google Colab Pro (see the [Google Colab notebook](https://colab.research.google.com/drive/1cSP5HoRZCbjH55MdYiRtxC_Q0obQQ5ZD?authuser=1#scrollTo=0d793Fus6RBY)). The A10 benchmarks were done on g5.4xlarge AWS instances with 1 GPU.


## Conclusions and next steps

We have shown that new features of PyTorch 2 - compiler and optimized attention implementation - give performance improvements exceeding or comparable with what previously required installation of an external dependency (xFormers). PyTorch achieved this, in particular, by integrating memory efficient attention from xFormers into its codebase. This is a significant improvement for user experience, given that xFormers, being a state-of-the-art library, in many scenarios requires custom installation process and long builds.

There are a few natural directions in which this work can be continued:	



* There are new implementations of SD, including a port to [HuggingFace diffusers library](https://github.com/huggingface/diffusers). It would be interesting to benchmark against them. Note that diffusers also require installing xFormers in order to use memory efficient attention
* The optimizations we implemented and described here are only benchmarked for text-to-image inference so far. It would be interesting to see how they affect training. PyTorch compilation can be directly applied to training; enabling training with PyTorch optimized attention is on the roadmap
* We intentionally minimized changes to the original SD code. Further profiling and optimization can probably bring more improvements
* At the moment compilation is applied only to the U-Net model inside the sampler. Since there is a lot happening outside of U-Net (e.g. operations directly in the sampling loop), it would be beneficial to compile the whole sampler. However, this would require analysis of the compilation process to avoid recompilation at every sampling step
* Current code only applies compilation within the PLMS sampler, but it should be trivial to extend it to other samplers
* Besides text-to-image generation, SD 2.1 has other pipelines - image-to-image and inpainting. It would be interesting to measure how their performance improves from PyTorch 2 optimizations 

Try some of this in the [Colab](https://colab.research.google.com/drive/1cSP5HoRZCbjH55MdYiRtxC_Q0obQQ5ZD?usp=sharing) or on a GPU of your choice. See if you can further increase the performance of SD, and share the results! This is your chance to get a preview of PyTorch 2.0 and experience the features coming in the next release. 

As a note, if you want access to new PyTorch features which come after this post is published, just tweak the PyTorch and TorchVision versions in [environment.yaml](https://github.com/sgrigory/stablediffusion2/blob/0f6d17cb2602302bc0f5c7dee6825e4b49a85518/environment.yaml#L14-L15).


## Resources



* PyTorch 2.0 overview, which has a lot of information on `torch.compile`: [https://pytorch.org/get-started/pytorch-2.0/](https://pytorch.org/get-started/pytorch-2.0/)
* Tutorial on `torch.compile`: [https://pytorch.org/tutorials/intermediate/torch_compile_tutorial.html](https://pytorch.org/tutorials/intermediate/torch_compile_tutorial.html)
* General compilation troubleshooting: [https://pytorch.org/docs/master/dynamo/troubleshooting.html](https://pytorch.org/docs/master/dynamo/troubleshooting.html)
* Details on graph breaks: [https://pytorch.org/docs/master/dynamo/faq.html#identifying-the-cause-of-a-graph-break](https://pytorch.org/docs/master/dynamo/faq.html#identifying-the-cause-of-a-graph-break)
* Details on guards: [https://pytorch.org/docs/master/dynamo/guards-overview.html](https://pytorch.org/docs/master/dynamo/guards-overview.html)
* Video deep dive on TorchDynamo [https://www.youtube.com/watch?v=egZB5Uxki0I](https://www.youtube.com/watch?v=egZB5Uxki0I) 
* Benchmark of SD1 based on HuggingFace implementation: [https://lambdalabs.com/blog/inference-benchmark-stable-diffusion](https://lambdalabs.com/blog/inference-benchmark-stable-diffusion)
* Tutorial on optimized attention in PyTorch 1.12: [https://pytorch.org/tutorials/beginner/bettertransformer_tutorial.html](https://pytorch.org/tutorials/beginner/bettertransformer_tutorial.html) 


## Acknowledgements

We would like to thank Geeta Chauhan, Natalia Gimelshein, Patrick Labatut, Bert Maher, Mark Saroufim, Michael Voznesensky and Francisco Massa for their valuable advice and early feedback on the text.

Special thanks to Yudong Tao for creating the first version of Stable Diffusion with PyTorch native attention.

For more information, [visit this page with additional resources](/blog/performance-experiments-stable-diffusion/).

