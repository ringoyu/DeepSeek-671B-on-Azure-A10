# Deploying DeepSeek-R1 671B on Azure A10

## Overview
Deploying the **DeepSeek-R1 671B** model on **Azure A10** provides a highly cost-effective and high-performance solution, particularly when leveraging CPU offload optimization strategies.

## Hardware Specifications
- **Azure Instance**: NV72ads A10 v5
- **Operating System**: Linux 6.8.0-1020-Azure x86_64
- **GPU**: 48GB VRAM
- **CUDA Version**: 12.1+

## Performance Insights
- **Current Throughput**: ~8 tokens per second at only **10% GPU utilization**
- **Optimization in Progress**: Further deployment optimizations are underway to enhance performance. More information and documentation will be shared soon.

Stay tuned for more updates on optimizations and deployment strategies!

![image](https://github.com/user-attachments/assets/0d18299b-4837-4e32-ade6-e55ab7c8eb70)



## Quick Start

**Preparation**
- **Virtual Machine**: Standard NV72ads A10 v5 (72 vcpus, 880 GiB memory)
- **OS**:Ubuntu 22.04.5 LTS, Linux 6.8.0-1020-Azure x86_64
- **Nvidia Driver**:535.161.08 \
![image](https://github.com/user-attachments/assets/7a568c2f-c262-4a29-9a96-8291f5005dea)

- **CUDA**: release 12.2, V12.2.140 \
![image](https://github.com/user-attachments/assets/b40e0b50-cddc-40fd-8138-67bff067426c)



**Installation**
- **Add CUDA to Path**
```bash
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
export CUDA_PATH=/usr/local/cuda
```

- **Install packages: gcc, g++, build-essential, cmake, ninji-build**
```bash
sudo apt-get update
sudo apt-get install gcc g++ cmake ninja-build
dpkg -l | grep -E 'gcc|g\+\+|cmake|ninja-build'
```

- **Create conda environment and install python libraries**
```bash
conda create --name KTdeepseek python=3.11
sudo apt-get install gcc g++ cmake ninja-build
dpkg -l | grep -E 'gcc|g\+\+|cmake|ninja-build'
```



**Model Preparation and Running**



**Reference resources**\
https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#cuda-major-component-versions__table-cuda-toolkit-driver-versions \
https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#azure-installation \
https://github.com/kvcache-ai/ktransformers \
https://huggingface.co/deepseek-ai/DeepSeek-R1/tree/main \
https://docs.nvidia.com/deploy/cuda-compatibility/ \
https://github.com/modelscope/evalscope?tab=readme-ov-file#%EF%B8%8F-installation \
https://ollama.com/library/deepseek-r1:671b-q4_K_M \
https://github.com/ubergarm/r1-ktransformers-guide/blob/main/README.zh.md \
https://techcommunity.microsoft.com/blog/AzureHighPerformanceComputingBlog/running-deepseek-r1-on-a-single-ndv5-mi300x-vm/4372726 \
https://learn.microsoft.com/en-us/azure/virtual-machines/linux/n-series-driver-setup#nvidia-grid-drivers \
https://azure.microsoft.com/en-us/pricing/details/virtual-machines/linux/#pricing \
