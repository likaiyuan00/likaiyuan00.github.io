---
title: cosyvoice2
date: 2025-09-23 14:45:27
tags: llm
---
# CosyVoice 安装与使用指南

## 环境准备
### 1. 克隆代码仓库
```bash
git clone --recursive https://github.com/FunAudioLLM/CosyVoice.git
cd CosyVoice
# 若子模块克隆失败，重复执行直到成功
git submodule update --init --recursive
```

### 2. 安装Miniconda
```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
~/miniconda3/bin/conda init bash
source ~/.bashrc
```

### 3. 创建虚拟环境
```bash
conda create -n cosyvoice python=3.10
conda activate cosyvoice
```

### 4. 安装系统依赖
```bash
sudo yum install sox sox-devel
sudo yum groupinstall "Development Tools"
sudo yum install python3-devel libsndfile-devel
```

## 安装依赖
```bash
# 如果不使用ttsfrd 就必须下载pynini
conda install -y -c conda-forge pynini==2.1.5
pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host=mirrors.aliyun.com
```

## 模型下载
```bash
mkdir pretrained_models
modelscope download --model=iic/CosyVoice-300M-SFT --local_dir pretrained_models/CosyVoice-300M-SFT
modelscope download --model=iic/CosyVoice-300M --local_dir pretrained_models/CosyVoice-300M
modelscope download --model=iic/CosyVoice-300M-Instruct --local_dir pretrained_models/CosyVoice-300M-Instruct
modelscope download --model=iic/CosyVoice-ttsfrd --local_dir pretrained_models/CosyVoice-ttsfrd

cd pretrained_models/CosyVoice-ttsfrd/
unzip resource.zip -d .
pip install ttsfrd_dependency-0.1-py3-none-any.whl
#对应python3.10版本
pip install ttsfrd-0.4.2-cp310-cp310-linux_x86_64.whl
```

## 验证安装
```python
python -c "import torch;print(torch.cuda.is_available())"  # 应输出True
```

## 功能验证示例
### 零样本推理测试
```python
import time
import torchaudio
#from cosyvoice import CosyVoice2
from cosyvoice.utils.file_utils import load_wav
import sys
sys.path.append('third_party/Matcha-TTS')
from cosyvoice.cli.cosyvoice import CosyVoice, CosyVoice2

# 初始化模型
cosyvoice = CosyVoice2('/mnt/CosyVoice2-0.5B', load_jit=False, load_trt=False, load_vllm=False, fp16=False)

# 准备输入数据
prompt_speech_16k = load_wav('./asset/zero_shot_prompt.wav', 16000)
text = '你好。'
spk_text = '希望你以后'

# 存储 TTFT 结果
ttft_list = []

# 运行10次推理
for run in range(10):
    start_time = time.time()  # 记录开始时间
    first_token_received = False

    for i, j in enumerate(cosyvoice.inference_zero_shot(text, spk_text, prompt_speech_16k, stream=False)):
        if not first_token_received:
            ttft = (time.time() - start_time) * 1000  # 单位：毫秒
            print(f"Run {run+1} TTFT: {ttft:.2f} ms")
            ttft_list.append(ttft)
            first_token_received = True
        torchaudio.save(f'zero_shot_run{run+1}_{i}.wav', j['tts_speech'], cosyvoice.sample_rate)

# 计算并打印平均 TTFT
average_ttft = sum(ttft_list) / len(ttft_list)
print(f"\nAverage TTFT over 10 runs: {average_ttft:.2f} ms")
```

### 跨语言推理
```python
import sys
sys.path.append('third_party/Matcha-TTS')
from cosyvoice.cli.cosyvoice import CosyVoice, CosyVoice2
from cosyvoice.utils.file_utils import load_wav
import torchaudio

cosyvoice = CosyVoice2('/mnt/CosyVoice2-0.5B', load_jit=False, load_trt=False, load_vllm=False, fp16=False)
prompt_speech_16k = load_wav('./asset/zero_shot_prompt.wav', 16000)

for i, j in enumerate(cosyvoice.inference_cross_lingual('在他讲述那个荒诞故事的过程中，他突然[laughter]停下来，因为他自己也被逗笑了[laughter]。', prompt_speech_16k, stream=False)):
    torchaudio.save('fine_grained_control_{}.wav'.format(i), j['tts_speech'], cosyvoice.sample_rate)
```

### 音色选择（SFT模型）
```python
#当前支持的音色包括：[‘中文女’, ‘中文男’, ‘日语男’, ‘粤语女’, ‘英文女’, ‘英文男’, ‘韩语女’]
import sys
sys.path.append('third_party/Matcha-TTS')
from cosyvoice.cli.cosyvoice import CosyVoice, CosyVoice2
from cosyvoice.utils.file_utils import load_wav
import torchaudio

cosyvoice = CosyVoice('pretrained_models/CosyVoice-300M-SFT', load_jit=False, load_trt=False, fp16=False)
# sft usage
print(cosyvoice.list_available_spks())
# change stream=True for chunk stream inference
for i, j in enumerate(cosyvoice.inference_sft('你好，我是通义生成式语音大模型，请问有什么可以帮您的吗？', '中文女', stream=False)):
    torchaudio.save('sft_{}.wav'.format(i), j['tts_speech'], cosyvoice.sample_rate)
```

## Web界面部署
```bash
#必须使用CosyVoice-300M-SFT才有音色选择其他的没有
#需要下载 ffmpeg 不然无法合成
# 使用Docker运行
docker run -it --gpus all  \
-v /mnt/CosyVoice-300M-SFT:/CosyVoice/pretrained_models/CosyVoice-300M-SFT \
--ipc=host --cap-add=SYS_PTRACE --network=host \
--name CosyVoice-web --privileged \
-d registry.cn-hangzhou.aliyuncs.com/lky-deploy/cosyvoice:v2-cu12-py3.10 \
/bin/bash -c 'sleep inf'

# 安装FFmpeg
conda install -c conda-forge ffmpeg=5.1.2
pip install --force-reinstall pydub==0.25.1

# 启动服务
python webui.py --model_dir pretrained_models/CosyVoice-300M-SFT &> web.log &
```

## 注意事项
1. 音色选择功能仅支持`CosyVoice-300M-SFT`模型
2. 确保GPU可用且CUDA环境正确配置
3. 首次运行需下载约5GB的模型文件
4. 推荐使用阿里云镜像加速下载：`-i https://mirrors.aliyun.com/pypi/simple/`

## 参考资源
- [ModelScope 模型文档](https://www.modelscope.cn/models/iic/CosyVoice2-0.5B/summary)
- [GitHub 项目仓库](https://github.com/FunAudioLLM/CosyVoice)
