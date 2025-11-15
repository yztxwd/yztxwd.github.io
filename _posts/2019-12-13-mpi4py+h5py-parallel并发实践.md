---
title:      通过h5py-parallel多进程并发优化Chiptoolkit读取速度
subtitle:   多进程读取HDF5文件
date:       2019-12-13
author_id:  jianyu
toc:        true
categories: [Tutorial, Blog]
tags: [python, chiptoolkit]
---

## 前言

之前我写了一个简单的模块numpyArrayDict.py来保存genome coverage信息。通过这个方法可以将reads在genome上的per base coverage计算完后保存在同一个HDF5文件内，而且一些其他类型的per base info（例如DNA methylation data处理后得到的每个C上甲基化的比例）也可以存储。由于h5py支持直接从文件中读取出给定范围的slice并且输出为numpy array，我们不用将整个genome的数据读取到内存，这样节省了内存，还省去了每次处理的时候都要重新计算coverage的麻烦。但是当时写的脚本同一时间只能有一个reader读取数据，在需要读取大量source的时候会等很长时间。最近看到了parallel hdf5这个功能，正好我也要做大量数据的correlation analysis，就来试试手啦。

## 安装h5py-parallel

之前完全没想到只是重新build一下会这么麻烦，各种环境问题搞了我两天。。。记录一下正确流程，主要步骤是根据[Build and Install MPI, parallel HDF5, and h5py from Source on Linux](https://drtiresome.com/2016/08/23/build-and-install-mpi-parallel-hdf5-and-h5py-from-source-on-linux/)来安装的

**在conda环境下安装h5py-parallel**
conda不推荐在环境内使用非conda命令安装package，而conda源提供的h5py-parallel安装后没有用（提示找不到h5py，不知道为什么），所以创建一个单独的环境来安装h5py-parallel
&nbsp;
conda版本：4.6.14
&nbsp;
1. 首先是创建一个新的环境phdf5
```
(base) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ conda create -n phdf5
```
2. 安装python3.7/gcc/mpi4py/numpy
```
(base) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ conda install -n phdf5 python=3.7 gcc_linux-64 mpi4py numpy
```
3. 启动环境
```
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/hdf5-1.10.5$ conda activate phdf5
```
4. 下载[HDF5 source code](https://www.hdfgroup.org/downloads/hdf5/source-code/)，解压，我的版本是1.10.5
```
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ wget https://s3.amazonaws.com/hdf-wordpress-1/wp-content/uploads/manual/HDF5/HDF5_1_10_5/source/hdf5-1.10.5.tar.gz
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ tar -zxvf hdf5-1.10.5.tar.gz
```
5. 进入HDF5解压的目录，安装HDF5，这里要指定环境变量CC为conda环境下的mpicc
```
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ cd hdf5-1.10.5
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/hdf5-1.10.5$ which mpicc
/data1/yztxwd/miniconda3/envs/phdf5/bin/mpicc
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/hdf5-1.10.5$ CC=mpicc ./configure --disable-fortran --prefix=/data1/yztxwd/Software/test/hdf5 --enable-shared --enable-parallel
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/hdf5-1.10.5$ make -j4 && make check && make install
```
6. 回到test目录下，下载[h5py source code](https://pypi.org/project/h5py/#files)，解压，我的版本是2.10.0
```
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ cd ..
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ wget https://files.pythonhosted.org/packages/5f/97/a58afbcf40e8abecededd9512978b4e4915374e5b80049af082f49cebe9a/h5py-2.10.0.tar.gz
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ tar -zxvf h5py-2.10.0.tar.gz
```
7. 安装h5py-parallel,注意configure --hdf5选项后的目录为刚刚编译安装的hdf5 source code
```
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test$ cd h5py-2.10.0
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/h5py-2.10.0$ export CC=mpicc
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/h5py-2.10.0$ export HDF5_MPI="ON"
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/h5py-2.10.0$ which python
/data1/yztxwd/miniconda3/envs/phdf5/bin/python
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/h5py-2.10.0$ python setup.py configure --mpi --hdf5=/data1/yztxwd/Software/test/hdf5
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/h5py-2.10.0$ python setup.py build
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/h5py-2.10.0$ python setup.py install
```
8. 安装完测试一下，import成功
```
(phdf5) yztxwd@YenlabMain:/data1/yztxwd/Software/test/h5py-2.10.0$ cd
(phdf5) yztxwd@YenlabMain:/data1/yztxwd$ python
>>> import h5py
```

## 使用h5py-parallel实现多进程读取单个文件
h5py-parallel的使用还是很简单的，根据[docs](https://docs.h5py.org/en/stable/mpi.html)唯一比较需要注意的是任何改变文件strcutre和metadata（比如创建一个新的group）都必须由所有的进程集体操作（就是都执行命令），例子：
```
>>> grp = f.create_group('x')  # right

>>> if rank == 1:
...     grp = f.create_group('x')   # wrong; all processes must do this
```

## 效果
之前的script：[loadHDF_pearson.py](https://github.com/yztxwd/chiptoolkit/blob/master/loadHDF5_pearson.py)
 25个source，16000个region在1个进程下耗时：
```
real	5m23.990s
user	5m26.284s
sys	0m53.624s
```
使用h5py-parallel修改后：[loadHDF5_pearson_mpi.py](https://github.com/yztxwd/chiptoolkit/blob/master/loadHDF5_pearson_mpi.py)
 25个source，16000个region在25个进程下耗时：
```
real	1m52.002s
user	23m30.316s
sys	2m52.412s
```

## 后记
虽然并行读取速度虽然有所提升但与使用的进程数相比不是特别令人满意，可能是进程间传递对象耗时较长。而且我也不清楚hdf5-parallel实现原理，之后还需要深入分析优化。

    
