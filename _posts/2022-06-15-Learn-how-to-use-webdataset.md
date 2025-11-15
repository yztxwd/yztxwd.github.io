---
layout:         post
title:          Webdataset for PyTorch
date:           2022-6-15
author_id:  jianyu
toc:        true
categories: [Tutorial, Blog]
tags: [deep learning, webdataset, pytorch]
---

After switching to Pytorch for deep learning projects, I kept looking for a dataset format that can give me as good performance as TFRecord. 

First I tried using tensorflow to save and load TFRecord, but keeping both tensorflow and pytorch in the same environment doesn't seem very elegant. Then I switched to a nice repo [tfrecord](https://github.com/vahidk/tfrecord) enabling me to get rid of the clumsy tensorflow library, however, TPU training requires dataset to have `__len__` method while there is no such option in [tfrecord](https://github.com/vahidk/tfrecord).

Finally I decide to use [webdataset](https://github.com/webdataset/webdataset), which is the recommended dataset format by PyTorch official team. I tried learning this several months ago, while I didn't figure out how exactly I should use it at that time.

## A brief introduction

It seems webdataset's idea is very straightforward, each sample is represented as a collection of files, e.g., suppose a sample consists of `sequence` and `target`, each corresponds to a numpy array, saving them in npy format:

```
sample1 sequence1.npy target.npy
sample2 sequence2.npy target.npy
sample3 sequence3.npy target.npy
...
```

Then using `tar` command to pack them together, the packed tar file can be used by webdataset.

A nice thing comes with this straightforward idea is, you can use any format you like to save the data, including pickle, npy, coded image... This flexibility is very nice. Also webdataset is compatibile with **NVIDIA DALI**, which is another high performance tool for loading and manipulating data during training.

## Creating WebDataset in python

Create such a tar file in python is also not a heavy task, suppose we already have a dataset. Just looping through the dataset and putting each sample in a dictionary, then feeding those dictionaries into the writer you will be fine. As the example shows below.

```python
sink = wds.TarWriter("dest.tar")
dataset = open_my_dataset()
for index, (input, output) in dataset:
    sink.write({
        "__key__": "sample%06d" % index,
        "input.npy": input,
        "output.npy": output,
    })
sink.close()
```

Be noticed: This is the part I didn't understand during the last time, the suffix `.npy` means it would be in numpy format, you can also choose `.pyd` to save in pickle format. There are also other pre-defined suffix (e.g., some compression formats for images) you can use.

## Load it back

When loading the dataset, to retrieve the original data back, you need to decode the data back to things you want. As the example below uses the pickled format. Loading pickled object would be:

```python
def npy_decoder(key, value):
    if not key.endswith(".npy"):
        return None
    assert isinstance(value, bytes)
    return np.load(io.BytesIO(value))

dataset = DataPipeline(
    wds.SimpleShardList("dest.tar"),
    wds.tarfile_to_samples(),
    wds.decode(npy_decoder),
    wds.to_tuple("seq.npy", "target.npy"),
)
```

As of I wrote this blog, the document available on github includes a lot of examples not working on my side, only this `pipeline` style interface works for me. I guess it's a little bit outdated, or maybe they expect you to read the source code to understand how it works

## NVIDIA DALI

NVIDIA DALI has reader function for Webdataset, which is very nice, probably I would build a standard input pipeline combing Webdataset + NVIDIA DALI. This solution seems both flexible (variable formats by Webdataset) and high performance
