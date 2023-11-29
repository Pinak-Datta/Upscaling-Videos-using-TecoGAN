# Upscaling-Videos-using-TecoGAN

This readme provides instructions on how to run the TecoGAN model for video enhancement, as well as details on evaluating its performance.

[Google Colab Notebook Link](https://colab.research.google.com/drive/1PeyhUNNX7OcL0KywTnjnCh70TSsKF8uC?usp=sharing)

## Requirements

* Python
* Jupyter Notebook
* nnabla library
* ffmpeg
* nnabla-ext-cuda116
* Pre-trained TecoGAN model

## Setup
Install required packages by running the following commands in your Jupyter Notebook:
```
pip install ffmpeg
pip install nnabla-ext-cuda116
git clone https://github.com/sony/nnabla-examples.git
```

Navigate to the TecoGAN directory:

```bash
cd nnabla-examples/video-superresolution/tecogan
```

Download pre-trained parameters and a subset of the test dataset:
```bash
wget https://nnabla.org/pretrained-models/nnabla-examples/GANs/tecogan/tecogan_model.h5
wget https://ge.in.tum.de/download/data/TecoGAN/vid3_LR.zip
unzip vid3_LR.zip
```

## Super-Resolution
Apply TecoGAN to a low-resolution scene from the downloaded dataset (for testing the model):

```bash
python generate.py --model tecogan_model.h5 --input-dir-lr walk --output-dir results/walk
ffmpeg -i results/walk/output_%04d.png -r 24/1 -y hr.mp4
ffmpeg -i walk/%04d.png -r 24/1 -y lr.mp4
```

## Evaluation
Check the results by playing the low and high-resolution versions:

```python
from IPython.display import HTML
from base64 import b64encode

def play_video(filename, height=512, width=512):
    mp4 = open(filename, 'rb').read()
    data_url = "data:video/mp4;base64," + b64encode(mp4).decode()
    return HTML(f"""
    <video width={width} height={height} controls>
          <source src={data_url} type="video/mp4">
    </video>""")

# Play low resolution version
play_video("lr.mp4")

# Play high resolution version
play_video("hr.mp4")
```

## Testing Against Custom Video
You can use your own video by uploading it:

```python
from google.colab import files

video = files.upload()
```

Rename the input video for convenience:

```python
import os

ext = os.path.splitext(list(video.keys())[-1])[-1]
os.rename(list(video.keys())[-1], "input_video{}".format(ext))
input_video = "input_video" + ext
```

Convert the video to a resolution aspect ratio of width 200:

```python
width=!ffprobe -v error -select_streams v:0 -show_entries stream=width -of csv=p=0 input_video.mp4
height=!ffprobe -v error -select_streams v:0 -show_entries stream=height -of csv=p=0 input_video.mp4
new_height = int(height[0])*200 // int(width[0])
new_height = 4*round(new_height/4)
!mkdir -p frames/input_video
!ffmpeg -i $input_video -vf "fps=30, scale=200:$new_height" frames/input_video/frame_%04d.png
!ffmpeg -i frames/input_video/frame_%04d.png -r 30/1 -y user_video_lr.mp4
```

## Apply super-resolution with TecoGAN to the custom video:

```bash
python generate.py --model tecogan_model.h5 --input-dir-lr frames/input_video/ --output-dir results/user
ffmpeg -i results/user/output_frame_%04d.png -i input_video.mp4 -r 30/1
```

## Evaluation Metrics:
For the model evaluation, I have used both SSIM and PSNR metrics.

### For SSIM:

```python
!ffmpeg -i user_video_hr.mp4 -i user_video_lr.mp4 -lavfi "[0:v]scale=iw*min(800/iw\,800/ih):ih*min(800/iw\,800/ih)[a];[1:v]scale=iw*min(800/iw\,800/ih):ih*min(800/iw\,800/ih)[b];[a][b]ssim" -f null -
```

Output:
```bash
[Parsed_ssim_2 @ 0x5c735fc7e480] SSIM Y:0.750084 (6.022060) U:0.981342 (17.291398) V:0.976813 (16.347605) All:0.902747 (10.120948)
```

### For PSNR:

```python
!ffmpeg -i user_video_lr.mp4 -i user_video_hr.mp4 -lavfi "[0:v]scale=iw*min(800/iw\,800/ih):ih*min(800/iw\,800/ih)[a];[1:v]scale=iw*min(800/iw\,800/ih):ih*min(800/iw\,800/ih)[b];[a][b]psnr" -f null -
```

Output:
```bash
[Parsed_psnr_2 @ 0x5ad49a41c640] PSNR y:22.419331 u:44.076441 v:40.353784 average:27.092135 min:26.469507 max:27.641027
```
