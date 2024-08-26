# CogVideo

# Diffusers

## Model

| T2V-Models   | Resolution | Checkpoints                                                         |
|--------------|------------|---------------------------------------------------------------------|
| CogVideoX-2b | 720x480    | [Hugging Face](https://huggingface.co/THUDM/CogVideoX-2b/tree/main) |
|

## Get started

## Set up environment

### For CUDA 12.x
```
conda create -n cogvideo python=3.10
conda activate cogvideo
```


### Install dependencies
```
pip install -r requirements.txt
pip install --upgrade opencv-python transformers diffusers # Must using diffusers>=0.30.0
```

## Prepare checkpoints

Use the following command to clone the repository and download the checkpoints. 
Or access the [Hugging Face](https://huggingface.co/THUDM/CogVideoX-2b) to download the checkpoints.
```
git lfs install
git clone https://huggingface.co/THUDM/CogVideoX-2b
```

### Remarks
When downloading the checkpoints, the file /text_encoder/model-00001-of-00002.safetensors and
/text_encoder/model-00002-of-00002.safetensors may not be successfully downloaded. You need to remove the
files and download them again using the following command:
```
cd CogVideoX-2b
cd text_encoder
rm model-00001-of-00002.safetensors
wget https://huggingface.co/THUDM/CogVideoX-2b/resolve/main/text_encoder/model-00001-of-00002.safetensors
rm model-00002-of-00002.safetensors
wget https://huggingface.co/THUDM/CogVideoX-2b/resolve/main/text_encoder/model-00002-of-00002.safetensors
```

## Generate video
Generates a video based on the given prompt and saves it to the specified path.

Parameters:
- prompt (str): The description of the video to be generated.
- model_path (str): The path of the pre-trained model to be used.
- output_path (str): The path where the generated video will be saved.
- num_inference_steps (int): Number of steps for the inference process. More steps can result in better quality.
- guidance_scale (float): The scale for classifier-free guidance. Higher values can lead to better alignment with the prompt.
- num_videos_per_prompt (int): Number of videos to generate per prompt.
- dtype (torch.dtype): The data type for computation (default is torch.float16).


### Example
```
python inference_diffusers.py --prompt "A video of a cat playing with a ball" --model_path /path/to/CogVideoX-2b --output_path output.mp4
```
It will generate a video of a cat playing with a ball and save it to the file /cogVideo/output.mp4.

# SAT Inference
* Single GPU Inference (FP16), the model need 18GB GPU memory. \
* Multi GPUs Inference (FP16), the model need 20GB minimum per GPU using diffusers .

## Get started

### Set up environment
```
conda create -n CogVideo python=3.10
conda activate CogVideo
```

### Install dependencies
```
cd cogVideo
pip install -r requirements.txt
cd sat
pip install -r requirements.txt
```

### Prepare checkpoints
```
mkdir CogVideoX-2b-sat
cd CogVideoX-2b-sat
wget https://cloud.tsinghua.edu.cn/f/fdba7608a49c463ba754/?dl=1
mv 'index.html?dl=1' vae.zip
unzip vae.zip
wget https://cloud.tsinghua.edu.cn/f/556a3e1329e74f1bac45/?dl=1
mv 'index.html?dl=1' transformer.zip
unzip transformer.zip
```
### Clone the T5 model
```
git clone https://huggingface.co/THUDM/CogVideoX-2b.git
mkdir t5-v1_1-xxl
mv CogVideoX-2b/text_encoder/* CogVideoX-2b/tokenizer/* t5-v1_1-xxl
```

### Modify the file in ```/path/to/cogVideo/sat/configs/cogvideox_2b.yaml```.
```
model:
  scale_factor: 1.15258426
  disable_first_stage_autocast: true
  log_keys:
    - txt

  denoiser_config:
    target: sgm.modules.diffusionmodules.denoiser.DiscreteDenoiser
    params:
      num_idx: 1000
      quantize_c_noise: False

      weighting_config:
        target: sgm.modules.diffusionmodules.denoiser_weighting.EpsWeighting
      scaling_config:
        target: sgm.modules.diffusionmodules.denoiser_scaling.VideoScaling
      discretization_config:
        target: sgm.modules.diffusionmodules.discretizer.ZeroSNRDDPMDiscretization
        params:
          shift_scale: 3.0

  network_config:
    target: dit_video_concat.DiffusionTransformer
    params:
      time_embed_dim: 512
      elementwise_affine: True
      num_frames: 49
      time_compressed_rate: 4
      latent_width: 90
      latent_height: 60
      num_layers: 30
      patch_size: 2
      in_channels: 16
      out_channels: 16
      hidden_size: 1920
      adm_in_channels: 256
      num_attention_heads: 30

      transformer_args:
        checkpoint_activations: True ## using gradient checkpointing
        vocab_size: 1
        max_sequence_length: 64
        layernorm_order: pre
        skip_init: false
        model_parallel_size: 1
        is_decoder: false

      modules:
        pos_embed_config:
          target: dit_video_concat.Basic3DPositionEmbeddingMixin
          params:
            text_length: 226
            height_interpolation: 1.875
            width_interpolation: 1.875

        patch_embed_config:
          target: dit_video_concat.ImagePatchEmbeddingMixin
          params:
            text_hidden_size: 4096

        adaln_layer_config:
          target: dit_video_concat.AdaLNMixin
          params:
            qk_ln: True

        final_layer_config:
          target: dit_video_concat.FinalLayerMixin

  conditioner_config:
    target: sgm.modules.GeneralConditioner
    params:
      emb_models:
        - is_trainable: false
          input_key: txt
          ucg_rate: 0.1
          target: sgm.modules.encoders.modules.FrozenT5Embedder
          params:
            model_dir: "{absolute_path/to/your/t5-v1_1-xxl}/t5-v1_1-xxl" # Absolute path to the t5-v1_1-xxl weights folder
            max_length: 226

  first_stage_config:
    target: vae_modules.autoencoder.VideoAutoencoderInferenceWrapper
    params:
      cp_size: 1
      ckpt_path: "{absolute_path/to/your/t5-v1_1-xxl}/CogVideoX-2b-sat/vae/3d-vae.pt" # Absolute path to the CogVideoX-2b-sat/vae/3d-vae.pt folder
      ignore_keys: [ 'loss' ]

      loss_config:
        target: torch.nn.Identity

      regularizer_config:
        target: vae_modules.regularizers.DiagonalGaussianRegularizer

      encoder_config:
        target: vae_modules.cp_enc_dec.ContextParallelEncoder3D
        params:
          double_z: true
          z_channels: 16
          resolution: 256
          in_channels: 3
          out_ch: 3
          ch: 128
          ch_mult: [ 1, 2, 2, 4 ]
          attn_resolutions: [ ]
          num_res_blocks: 3
          dropout: 0.0
          gather_norm: True

      decoder_config:
        target: vae_modules.cp_enc_dec.ContextParallelDecoder3D
        params:
          double_z: True
          z_channels: 16
          resolution: 256
          in_channels: 3
          out_ch: 3
          ch: 128
          ch_mult: [ 1, 2, 2, 4 ]
          attn_resolutions: [ ]
          num_res_blocks: 3
          dropout: 0.0
          gather_norm: False

  loss_fn_config:
    target: sgm.modules.diffusionmodules.loss.VideoDiffusionLoss
    params:
      offset_noise_level: 0
      sigma_sampler_config:
        target: sgm.modules.diffusionmodules.sigma_sampling.DiscreteSampling
        params:
          uniform_sampling: True
          num_idx: 1000
          discretization_config:
            target: sgm.modules.diffusionmodules.discretizer.ZeroSNRDDPMDiscretization
            params:
              shift_scale: 3.0

  sampler_config:
    target: sgm.modules.diffusionmodules.sampling.VPSDEDPMPP2MSampler
    params:
      num_steps: 50
      verbose: True

      discretization_config:
        target: sgm.modules.diffusionmodules.discretizer.ZeroSNRDDPMDiscretization
        params:
          shift_scale: 3.0

      guider_config:
        target: sgm.modules.diffusionmodules.guiders.DynamicCFG
        params:
          scale: 6
          exp: 5
          num_steps: 50
```

### Modify the file in ```/path/to/cogVideo/sat/configs/inference.yaml```.
```
args:
  latent_channels: 16
  mode: inference
  load: "{absolute_path/to/your}/transformer" # Absolute path to the CogVideoX-2b-sat/transformer folder
  # load: "{your lora folder} such as zRzRzRzRzRzRzR/lora-disney-08-20-13-28" # This is for Full model without lora adapter

  batch_size: 1
  input_type: txt # You can choose txt for pure text input, or change to cli for command line input
  input_file: configs/test.txt # Pure text file, which can be edited. If use command line as prompt iuput, please change it to input_type: cli
  sampling_num_frames: 13  # Must be 13, 11 or 9
  sampling_fps: 8
  fp16: True # For CogVideoX-2B
#  bf16: True # For CogVideoX-5B
  output_dir: outputs/ # Change the output_dir if you want to save the results to your own directory
  force_inference: True
```

#### Remarks
If you want to use your own prompt, you can modify the file in ```/path/to/CogVideo/sat/configs/test.txt```.
If multiple prompts is required, in which each line makes a prompt.

### Generate video
Go to file ```/path/to/cogVideo/sat```, and run the following command:
```
bash inference.sh
```