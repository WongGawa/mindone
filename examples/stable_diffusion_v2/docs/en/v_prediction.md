# v-prediction

> [Progressive Distillation for Fast Sampling of Diffusion Models (Section2.4)](https://arxiv.org/pdf/2202.00512.pdf)

## Introduction

V-prediction refers to a type of loss objective in diffusion model. It was first introduced by the Brain team of Google Research for fast sampling diffusion models in 2022. Instead of estimating the noise $\varepsilon$, v-prediction estimates the latent variable's velocity relative to signal-to-nose(SNR) phase, namely,

$$
v\equiv \alpha _t\epsilon -\sigma _tx
$$

V-prediction re-parameterizes the diffusion models in a way that the implied prediction $x_\theta$ remains stable as SNR varies.

## Preparation

#### Dependency

Please refer to the [Installation](../../README.md#installation) section.


#### Pretrained Models

Please download the pretrained [SD2.0-768-v checkpoint](https://download.mindspore.cn/toolkits/mindone/stable_diffusion/sd_v2_768_v-e12e3a9b.ckpt) and put it under `stable_diffusion_v2/models` folder.

#### Text-image Dataset Preparation

The text-image pair dataset for finetuning should follow the file structure below

```text
dir
├── img1.jpg
├── img2.jpg
├── img3.jpg
└── img_txt.csv
```

img_txt.csv is the annotation file in the following format

```text
dir,text
img1.jpg,a cartoon character with a potted plant on his head
img2.jpg,a drawing of a green pokemon with red eyes
img3.jpg,a red and white ball with an angry look on its face
```

For convenience, we have prepared two public text-image datasets obeying the above format.

- [pokemon-blip-caption dataset](https://openi.pcl.ac.cn/jasonhuang/mindone/datasets), containing 833 pokemon-style images with BLIP-generated captions.
- [Chinese-art blip caption dataset](https://openi.pcl.ac.cn/jasonhuang/mindone/datasets), containing 100 chinese art-style images with BLIP-generated captions.

To use them, please download `pokemon_blip.zip` and `chinese_art_blip.zip` from the [openi dataset website](https://openi.pcl.ac.cn/jasonhuang/mindone/datasets). Then unzip them on your local directory, e.g. `./datasets/pokemon_blip`.

## Finetune

We will use the `train_text_to_image.py` script for v-prediciton finetuning.

Before running, please make sure the `image_size` is set to `768` and please modify the following arguments to your
local path in the shell or in the config file `train_config_vanilla_v2_vpred.yaml`:

* `--data_path=/path/to/data`
* `--output_path=/path/to/save/output_data`
* `--pretrained_model_path=/path/to/pretrained_model`

Then, execute the script to launch finetuning:

```shell
python train_text_to_image.py \
    --train_config "configs/train/train_config_vanilla_v2_vpred.yaml" \
    --data_path "datasets/pokemon_blip/train" \
    --output_path "output/vpred_vanilla_finetune_pokemon/txt2img" \
    --pretrained_model_path "models/sd_v2_768_v-e12e3a9b.ckpt"
```

After training, the finetuned checkpoint will be saved in `{output_path}/ckpt/txt2img/ckpt/rank_0/sd-72.ckpt`.

Below are some arguments that you may want to tune for a better performance on your dataset:

- train_batch_size: the number of batch size for training.
- start_learning_rate and end_learning_rate: the initial and end learning rates for training.
- epochs: the number of epochs for training.
- use_ema: whether use EMA for model smoothing

For more argument illustration, please run `python train_text_to_image.py -h`.

## Inference

To perform text-to-image generation with the finetuned v-prediction checkpoint, fisrst modify `configs/v2-inference.yaml` as follows to switch from `eps-prediction` to `v-prediction`,

```yaml
model:
  prediction_type: "v"
```

Then run

```shell
python text_to_image.py \
    --prompt "A drawing of a fox with a red tail" \
    --config configs/v2-inference.yaml \
    --output_path ./output/ \
    --W 768 \
    --H 768 \
    --ckpt_path {path/to/v_prediction_checkpoint_after_finetune}
```

Please update `ckpt_path` according to your finetune settings.

Here are the example results.

<div align="center">
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/pokemon_00012.png" width="160" height="160" />
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/pokemon_00021.png" width="160" height="160" />
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/pokemon_00140.png" width="160" height="160" />
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/pokemon_00161.png" width="160" height="160" />
</div>
<p align="center">
  <em> Images generated by Stable Diffusion 2.0 v-prediction fine-tuned on pokemon-blip dataset</em>
</p>

<div align="center">
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/chinese_art_00000.png" width="160" height="160" />
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/chinese_art_00003.png" width="160" height="160" />
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/chinese_art_00009.png" width="160" height="160" />
<img src="https://github.com/Mark-ZhouWX/resources/blob/main/stable_diffusion/chinese_art_00019.png" width="160" height="160" />
</div>
<p align="center">
  <em> Images generated by Stable Diffusion 2.0 v-prediction fine-tuned on chinese-art-blip dataset</em>
</p>

### Evaluation

We will evaluate the finetuned model on the split test set in `pokemon_blip.zip` and `chinese_art_blip.zip`.

Let us run text-to-image generation conditioned on the prompts in test set then evaluate the quality of the generated images by the following steps.

1. Before running, please modify the following arguments to your local path:

* `--data_path=/path/to/prompts.txt`
* `--output_path=/path/to/save/output_data`
* `--ckpt_path=/path/to/model_checkpoint`

`prompts.txt` is a file which contains multiple prompts, and each line is the caption for a real image in test set, for example

```text
a drawing of a spider on a white background
a drawing of a pokemon with blue eyes
a drawing of a pokemon pokemon with its mouth open
...
```

2. Run multiple-prompt inference on the test set

```shell
python text_to_image.py \
    --version "2.0" \
    --prompt "a wolf in winter" \
    --config configs/v2-inference.yaml \
    --output_path output/ \
    --seed 42 \
    --n_iter 4 \
    --n_samples 1 \
    --W 768 \
    --H 768 \
    --sampling_steps 15 \
    --dpm_solver \
    --scale 9 \
    --ckpt_path "models/sd_v2_768_v-e12e3a9b.ckpt"
```

The generated images will be saved in the `{output_path}/samples` folder.

Note that the following hyper-param configuration will affect the generation and evaluation results.

* sampler: the diffusion sampler
* sampling\_steps: the sampling steps
* scale: unconditional guidance scale

For more details, please run

3. Evaluate the generated images

```shell
python eval/eval_fid.py --real_dir {path/to/test_images} --gen_dir {path/to/generated_images}
python eval/eval_clip_score.py --image_path {path/to/test_images} --prompt {path/to/prompts_file} --load_checkpoint {path/to/checkpoint}
```

For details, please refer to the guideline [Diffusion Evaluation](file:///C:/01Code/mindone/examples/stable_diffusion_v2/eval/README.md).

## Reference

[1] [Progressive Distillation for Fast Sampling of Diffusion Models (Section2.4)](https://arxiv.org/pdf/2202.00512.pdf)