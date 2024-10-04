<!--Copyright 2024 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
-->
# CogVideoX
CogVideoX is an open-source version of the video generation model originating from QingYing. The table below displays the list of video generation models we currently offer, along with their foundational information.


## Load model checkpoints
Model weights may be stored in separate subfolders on the Hub or locally, in which case, you should use the from_pretrained() method:


```
from diffusers import CogVideoXPipeline, CogVideoXImageToVideoPipeline
pipe = CogVideoXPipeline.from_pretrained(
    "THUDM/CogVideoX-2b",
    torch_dtype=torch.float16
)

pipe = CogVideoXImageToVideoPipeline.from_pretrained(
    "THUDM/CogVideoX-5b-I2V",
    torch_dtype=torch.bfloat16
)

```

## Text-to-Video
For text-to-Video, pass a text prompt. By default, CogVideoX generates a 720 x 480 Video for the best results

```
import torch
from diffusers import CogVideoXPipeline
from diffusers.utils import export_to_video

prompt = "An elderly gentleman, with a serene expression, sits at the water's edge, a steaming cup of tea by his side. He is engrossed in his artwork, brush in hand, as he renders an oil painting on a canvas that's propped up against a small, weathered table. The sea breeze whispers through his silver hair, gently billowing his loose-fitting white shirt, while the salty air adds an intangible element to his masterpiece in progress. The scene is one of tranquility and inspiration, with the artist's canvas capturing the vibrant hues of the setting sun reflecting off the tranquil sea."

pipe = CogVideoXPipeline.from_pretrained(
    "THUDM/CogVideoX-5b",
    torch_dtype=torch.bfloat16
)

pipe.enable_model_cpu_offload()
pipe.vae.enable_tiling()

video = pipe(
    prompt=prompt,
    num_videos_per_prompt=1,
    num_inference_steps=50,
    num_frames=49,
    guidance_scale=6,
    generator=torch.Generator(device="cuda").manual_seed(42),
).frames[0]

export_to_video(video, "output.mp4", fps=8)

```


<div class="flex justify-center">
    <img src="docs/source/en/imgs/cogvideox_out.gif" alt="generated image of an astronaut in a jungle"/>
</div>


## Image-to-Video


The are two variants of this model,[THUDM/CogVideoX-5b-I2V](https://huggingface.co/THUDM/CogVideoX-5b-I2V). The CogVideoX checkpoint is trained to generate 60 frames  

You'll use the CogVideoX-5b-I2V  checkpoint for this guide.

```py
import torch
from diffusers import CogVideoXImageToVideoPipeline
from diffusers.utils import export_to_video, load_image

prompt = "A vast, shimmering ocean flows gracefully under a twilight sky, its waves undulating in a mesmerizing dance of blues and greens. The surface glints with the last rays of the setting sun, casting golden highlights that ripple across the water. Seagulls soar above, their cries blending with the gentle roar of the waves. The horizon stretches infinitely, where the ocean meets the sky in a seamless blend of hues. Close-ups reveal the intricate patterns of the waves, capturing the fluidity and dynamic beauty of the sea in motion."
image = load_image(image="cogvideox_rocket.png")
pipe = CogVideoXImageToVideoPipeline.from_pretrained(
    "THUDM/CogVideoX-5b-I2V",
    torch_dtype=torch.bfloat16
)
 
pipe.vae.enable_tiling()
pipe.vae.enable_slicing()

video = pipe(
    prompt=prompt,
    image=image,
    num_videos_per_prompt=1,
    num_inference_steps=50,
    num_frames=49,
    guidance_scale=6,
    generator=torch.Generator(device="cuda").manual_seed(42),
).frames[0]

export_to_video(video, "output.mp4", fps=8)
```

<div class="flex gap-4">
  <div>
    <img class="rounded-xl" src="docs/source/en/imgs/cogvideox_rocket.png"/>
    <figcaption class="mt-2 text-center text-sm text-gray-500">initial image</figcaption>
  </div>
  <div>
    <img class="rounded-xl" src="docs/source/en/imgs/cogvideox_outrocket.gif"/>
    <figcaption class="mt-2 text-center text-sm text-gray-500">generated video</figcaption>
  </div>
</div>


## Reduce memory usage

While testing using the diffusers library, all optimizations included in the diffusers library were enabled. This
scheme has not been tested for actual memory usage on devices outside of **NVIDIA A100 / H100** architectures.
Generally, this scheme can be adapted to all **NVIDIA Ampere architecture** and above devices. If optimizations are
disabled, memory consumption will multiply, with peak memory usage being about 3 times the value in the table.
However, speed will increase by about 3-4 times. You can selectively disable some optimizations, including:

```
pipe.enable_sequential_cpu_offload()
pipe.vae.enable_slicing()
pipe.vae.enable_tiling()
```

+ For multi-GPU inference, the `enable_sequential_cpu_offload()` optimization needs to be disabled.
+ Using INT8 models will slow down inference, which is done to accommodate lower-memory GPUs while maintaining minimal
  video quality loss, though inference speed will significantly decrease.
+ The CogVideoX-2B model was trained in `FP16` precision, and all CogVideoX-5B models were trained in `BF16` precision.
  We recommend using the precision in which the model was trained for inference.
+ [PytorchAO](https://github.com/pytorch/ao) and [Optimum-quanto](https://github.com/huggingface/optimum-quanto/) can be
  used to quantize the text encoder, transformer, and VAE modules to reduce the memory requirements of CogVideoX. This
  allows the model to run on free T4 Colabs or GPUs with smaller memory! Also, note that TorchAO quantization is fully
  compatible with `torch.compile`, which can significantly improve inference speed. FP8 precision must be used on
  devices with NVIDIA H100 and above, requiring source installation of `torch`, `torchao`, `diffusers`, and `accelerate`
  Python packages. CUDA 12.4 is recommended.
+ The inference speed tests also used the above memory optimization scheme. Without memory optimization, inference speed
  increases by about 10%. Only the `diffusers` version of the model supports quantization.
+ The model only supports English input; other languages can be translated into English for use via large model
  refinement.
+ The memory usage of model fine-tuning is tested in an `8 * H100` environment, and the program automatically
  uses `Zero 2` optimization. If a specific number of GPUs is marked in the table, that number or more GPUs must be used
  for fine-tuning.