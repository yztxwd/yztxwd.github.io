---
layout:         post
title:          BPnet environment setup
date:           2022-3-18
author_id:  jianyu
toc:        true
categories: [Tutorial, Blog]
tags: [deep learning, bpnet]
---

{:toc}

Recently I want to get [BPnet](https://github.com/kundajelab/bpnet) running, but setting up environment for it is a bit tricky, so I wrote down the efforts I spent on this for future reference or anyone having trouble like me.

> The BPnet version I used in this post is https://github.com/kundajelab/bpnet/tree/6b846b65e9c0930ea798eddfed68e460018397be (Jul 7, 2021)

The main trouble I met is BPnet uses `tensorflow=1.7`, while the cudnn corresponding this version is not available on conda. The steps of solving this are described below:

## Installing tensorflow 1.7 by conda

### Set up everything except cuDNN

Clone the [BPnet](https://github.com/kundajelab/bpnet) repository first, then enter the root directory of repo.

```bash
$ git clone https://github.com/kundajelab/bpnet.git
$ cd bpnet
```

I used the following yaml file to create conda environment, there is some difference between this and the one available on bpnet repo, because I found there are some requirements of package not listed there.

```yaml
name: bpnet-gpu
channels:
- bioconda
- pytorch
- conda-forge
- defaults
dependencies:
- python=3.6.13
- click=8

# genomics
- pybedtools>=0.7.10
- bedtools>=2.27.1
- pybigwig>=0.3.10
- pysam>=0.14.0
- genomelake>=0.1.4

- pytorch  # optional for data-loading
- cython
# h5py 3.X doesn't work
- h5py>=2.7.0, <=2.10.0
- numpy
# has to specify this to avoid pytorch mess up requirements for tensorflow
- cudatoolkit=9.0
# newer sklearn doesn't work
- scikit-learn=0.19.2

- pandas>=0.23.0
- fastparquet
- python-snappy

- nb_conda=2.2.1
- pip
- pip:
  - git+https://github.com/kundajelab/DeepExplain.git

  #Synchronized packages for Jupyter Notebooks
  - ipykernel==5.3.0
  - ipython==7.15.0
  - ipywidgets==7.5.1
  - jupyter_client==6.1.3
  - jupyter_core==4.6.3
  - markdown==3.2.2
  - nbconvert==5.6.1
  - nbformat==5.0.6
  - notebook==6.0.3
  - papermill==2.1.1
  - nbclient==0.4.0

  # ML & numerics
  - tensorflow-gpu==1.7 #for gpu


  # bpnet package
  - .[dev,extras]  # install the local basepair package. All the other required pip packages are specified in the setup.py
```

Create the environment by the above yaml file (assume you already have conda and mamba installed):

```bash
$ mamba env create -f bpnet-gpu_new.yaml -n bpnet-gpu
```

You might notice that I didn't specify cudnn here because `cudnn 7.0.X` compiled by `cudatoolkit=9.0` is not available on anaconda and conda-forge channel

### cuDNN

Download the cuDNN compiled by `cudatoolkit=9.0` from [NVIDIA](https://developer.nvidia.com/rdp/cudnn-archive). You need to register a NVIDIA developer account for this.

![cuDNN 7.0.5](/assets/img/cudnn_version.png)

decompress

```bash
$ tar -zxvf cudnn-9.0-linux-x64-v7.tgz
```

Copy files into the **conda envrionment folder** we just created (replace `~/group/lab/jianyu/miniconda3` with your conda installation directory)

```bash
$ cp cuda/include/cudnn.h ~/group/lab/jianyu/miniconda3/envs/bpnet-gpu/include/
$ cp -P cuda/lib64/libcudnn* ~/group/lab/jianyu/miniconda3/envs/bpnet-gpu/lib/
```

Now test if `tensorflow 1.7` works

```bash
(bpnet-gpu) -bash-4.2$ python
Python 3.6.13 | packaged by conda-forge | (default, Sep 23 2021, 07:56:31) 
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> tf.__version__
'1.7.0'
>>> tf.test.is_gpu_available()
2022-03-17 15:58:50.014491: I tensorflow/core/platform/cpu_feature_guard.cc:140] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 AVX512F FMA
2022-03-17 15:58:50.390332: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1344] Found device 0 with properties: 
name: NVIDIA TITAN V major: 7 minor: 0 memoryClockRate(GHz): 1.455
pciBusID: 0000:a6:00.0
totalMemory: 11.78GiB freeMemory: 11.48GiB
2022-03-17 15:58:50.390491: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1423] Adding visible gpu devices: 0
2022-03-17 15:58:51.000570: I tensorflow/core/common_runtime/gpu/gpu_device.cc:911] Device interconnect StreamExecutor with strength 1 edge matrix:
2022-03-17 15:58:51.000645: I tensorflow/core/common_runtime/gpu/gpu_device.cc:917]      0 
2022-03-17 15:58:51.000659: I tensorflow/core/common_runtime/gpu/gpu_device.cc:930] 0:   N 
2022-03-17 15:58:51.000850: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1041] Created TensorFlow device (/device:GPU:0 with 11103 MB memory) -> physical GPU (device: 0, name: NVIDIA TITAN V, pci bus id: 0000:a6:00.0, compute capability: 7.0)
True
```

## Prepare inputs

I didn't find description of what kind of "pos/neg" bigwig files are expected in bpnet repo, on another associated [repo](https://github.com/kundajelab/basepairmodels) that includes bpnet model, there is a tutorial on how to prepare input files. Input files generated according to that works on my side.

## Train

During training, I found although before running evaluation step (which creates a new Tensorflow session) `K.clear_session()` is called to release GPU memory, the GPU memory was not released actually based on `nvidia-smi`, which leads to the program crash. Setting `--memfrac-gpu 0.4` solves this issue.

NOTE: I found another way of resolving it is to use `numba` to release GPU memory, but it needs some modification on the source code, refer [https://github.com/yztxwd/bpnet](https://github.com/yztxwd/bpnet) if you're interested. Might create a pull request later.
