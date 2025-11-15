---
layout:         post
title:          Make your life easier with snakemake
date:           2021-12-22
author_id:  jianyu
toc:        true
categories: [Tutorial, Blog]
tags: [snakemake, html]
---

No doubt, doing bioinformatic research requires your creativity, you need to work very hard to find the special pattern in a bunch of meaningless signal tracks, linking them to genes, making your conclusions, then getting the paper on journal. However, in practice, a great amout of time I was just sitting in front of the monitor and playing some pipeline scripts in bash. It's absolutely a headache to use bash to write a neat workflow, and it becomes even more painful when I want to integrate different bash scripts together to reproduce my previous analysis. After spending so much time understanding a variety of error messages, I gave up, and decided to look for better tools for my workflow management.

Thankfully, I did find a nice tool for workflow management, which is [snakemake](https://snakemake.readthedocs.io/en/stable/). My research heavily relies on this now, it saves me a lot of time to reproduce my previous results, especially during the past several years I was moving to different places frequently. I hope this blog can show you the benefit of using snakemake.

Snakemake borrows the idea of `GNU Make`, which is all the tasks are dependent on your output files. You predefine what kind of task (called `rule` in snakemake)you want to run, then Snakemake will find the correct rule for you based on desired output to run. A quick example would be like:

```python
# Snakefile
rule bwa_map:
    input:
        "data/chr1.fa",
        "data/samples/A.fastq"
    output:
        "output/mapped/A.bam"
    shell:
        "bwa mem {input} | samtools view -Sb - > {output}"
```

The basic components of a `rule` consist of rule name, input, output and the command (shell in this case). This rule does a simple task to map an empty fastq file to chromosome 1 using `bwa mem`, the command will be used has been written in `shell` field, `{input}` and `{output}` correspond to the contents in input and output fields. 

Currently, the file structure is like below:

```bash
(smk) ➜  ~[Snakemake_workshop] git:(main) tree
.
├── LICENSE
├── README.md
├── Snakefile
├── bwa.smk
├── data
│   ├── chr1.fa
│   ├── chr1.fa.amb
│   ├── chr1.fa.ann
│   ├── chr1.fa.bwt
│   ├── chr1.fa.pac
│   ├── chr1.fa.sa
│   └── samples
│       ├── A.fastq
│       ├── Alan.fastq
│       └── B.fastq
└── environment.yaml
```

The above rule is defined in Snakefile, which works like `Makefile` for `GNU Make`. This is the defaule file name that snakemake will use to find rules.

I have built the genome index for `chr1.fa`, now let's see what snakemake can do!

```bash
(smk_workshop) ➜  ~[Snakemake_workshop] git:(main) ✗ snakemake --cores 1 output/mapped/A.bam
Building DAG of jobs...
Using shell: /bin/bash
Provided cores: 1 (use --cores to define parallelism)
Rules claiming more threads will be scaled down.
Conda environments: ignored
Job stats:
job        count    min threads    max threads
-------  -------  -------------  -------------
bwa_map        1              1              1
total          1              1              1

Select jobs to execute...

[Wed Dec 22 17:41:27 2021]
rule bwa_map:
    input: data/chr1.fa, data/samples/A.fastq
    output: output/mapped/A.bam
    jobid: 0
    resources: tmpdir=/var/folders/p1/r92y9wf946z4plsbfdy02m780000gs/T

[M::bwa_idx_load_from_disk] read 0 ALT contigs
[M::process] read 2 sequences (72 bp)...
[M::mem_process_seqs] Processed 2 reads in 0.000 CPU sec, 0.000 real sec
[main] Version: 0.7.17-r1188
[main] CMD: bwa mem data/chr1.fa data/samples/A.fastq
[main] Real time: 6.378 sec; CPU: 0.293 sec
[Wed Dec 22 17:41:34 2021]
Finished job 0.
1 of 1 steps (100%) done
Complete log: /Users/yztxwd/Git/Snakemake_workshop/.snakemake/log/2021-12-22T174102.239522.snakemake.log
```

I used the command `snakemake --cores 1 output/mapped/A.bam`, as you already see, `output/mapped/A.bam` is the output file of rule `bwa_map`. Snakemake find the correct rule to execute and get the desired output!

Then we can do more, let's assume this time we have hundreds of fastq files to map, it would be impossible to write a rule for each file, we need to make this rule more "general" that can be used for all input files, here comes the `wildcard`. See below modified rule with wildcard

```python
# example of wildcard (generalized rule)
rule bwa_map_generalize:
    input:
        "data/chr1.fa",
        "data/samples/{sample}.fastq"
    output:
        "output/mapped/{sample}.wildcards.bam"
    shell:
        "bwa mem {input} | samtools view -Sb - > {output}"
```

The difference made is: now both input and output files have `{sample}`, which replaces the original file prefix `A`, this is called wildcard, string wrapped by curly bracket would be treated as wildcard. Everytime when snakemake finds the desired output matches this pattern `output/samples/{sample}.wildcards.bam`, it will assign this value to a special variable `wildcards`, and replace the corresponding placeholder string (in this case, `sample`) in input file to run this `rule`.

It might not be very intuitive for now, let see an example, now we want to generate output `output/mapped/A.wildcards.bam`

```bash

snakemake --cores 1 output/mapped/A.wildcards.bam
Building DAG of jobs...
Using shell: /bin/bash
Provided cores: 1 (use --cores to define parallelism)
Rules claiming more threads will be scaled down.
Conda environments: ignored
Job stats:
job                   count    min threads    max threads
------------------  -------  -------------  -------------
bwa_map_generalize        1              1              1
total                     1              1              1

Select jobs to execute...

[Wed Dec 22 17:52:17 2021]
rule bwa_map_generalize:
    input: data/chr1.fa, data/samples/A.fastq
    output: output/mapped/A.wildcards.bam
    jobid: 0
    wildcards: sample=A
    resources: tmpdir=/var/folders/p1/r92y9wf946z4plsbfdy02m780000gs/T

[M::bwa_idx_load_from_disk] read 0 ALT contigs
[M::process] read 2 sequences (72 bp)...
[M::mem_process_seqs] Processed 2 reads in 0.000 CPU sec, 0.000 real sec
[main] Version: 0.7.17-r1188
[main] CMD: bwa mem data/chr1.fa data/samples/A.fastq
[main] Real time: 4.565 sec; CPU: 0.270 sec
[Wed Dec 22 17:52:22 2021]
Finished job 0.
1 of 1 steps (100%) done
Complete log: /Users/yztxwd/Git/Snakemake_workshop/.snakemake/log/2021-12-22T175214.212067.snakemake.log

```

See, snakemake based on the output `output/mapped/A.wildcards.bam`, it knows `{sample}` should be equal to `A` and filled this into input to get `data/samples/A.fastq`, this specific rule was generated from the "general" rule we just created! I can generate output for `output/mapped/B.wildcards.bam` in thte same way, the corresponding input would be `data/samples/B.fastq`

```bash
(smk_workshop) ➜  ~[Snakemake_workshop] git:(main) ✗ snakemake -np output/mapped/B.wildcards.bam
Building DAG of jobs...
Job stats:
job                   count    min threads    max threads
------------------  -------  -------------  -------------
bwa_map_generalize        1              1              1
total                     1              1              1


[Wed Dec 22 18:09:56 2021]
rule bwa_map_generalize:
    input: data/chr1.fa, data/samples/B.fastq
    output: output/mapped/B.wildcards.bam
    jobid: 0
    wildcards: sample=B
    resources: tmpdir=/var/folders/p1/r92y9wf946z4plsbfdy02m780000gs/T

bwa mem data/chr1.fa data/samples/B.fastq | samtools view -Sb - > output/mapped/B.wildcards.bam
```

A `dry-run` is used this time (`-n`, `-p` to print the command would be executed), this is particularly useful when you just want to see what's going to happen. 

You can have multiple wildcards in output, for example, the output can be `output/mapped/{sample}.{replicate}.bam` to specify two wildcards `sample` and `replicate` to represent the sample and biological replicate. After specifying outputs, these wildcards will be used to find corresponding inputs.

> Note: All wildcards must appear in output, you shouldn't have a wildcard only exists in input. Remember snakemake starts everything from output!

The last useful feature I want to introduce is you can specify parameters for each rule, it's simple, I'll let example speak:

```python
# params:
rule bwa_map_params:
    input:
        "data/chr1.fa",
        "data/samples/{sample}.fastq"
    output:
        "output/mapped/{sample}.params.bam"
    threads: 4
    params: "-v 1"
    shell:
        "bwa mem -t {threads} {params} {input} | samtools view -Sb - > {output}"
```

There are two additional fields here, `threads` and `params`, both are used as arguments for `bwa mem`

`threads` is specifically used for number of cpus, snakemake uses this field to determine how many cpus will be used for this rule. When running many rules in parallel, snakemake decides which job to run based on current available cpus (total cpus is specified by `--cores`) and `threads` field of each rule.

`params` is more flexible, you can specify any kind of parameters, depending on which tool you are using. Here I use `-v 1` to specify the verbose level.

Let's roll! with a `dry-run`

```bash

(smk_workshop) ➜  ~[Snakemake_workshop] git:(main) ✗ snakemake -np output/mapped/A.params.bam
Building DAG of jobs...
Job stats:
job               count    min threads    max threads
--------------  -------  -------------  -------------
bwa_map_params        1              1              1
total                 1              1              1


[Wed Dec 22 18:12:30 2021]
rule bwa_map_params:
    input: data/chr1.fa, data/samples/A.fastq
    output: output/mapped/A.params.bam
    jobid: 0
    wildcards: sample=A
    resources: tmpdir=/var/folders/p1/r92y9wf946z4plsbfdy02m780000gs/T

bwa mem -t 1 -v 1 data/chr1.fa data/samples/A.fastq | samtools view -Sb - > output/mapped/A.params.bam
```

As you can see, `threads` and `params` were filled into the corresponding place in the command.

>An interesting thing you might notice is although I claimed that I need `4` cores to run this job, snakemake only gave me `1` core. This is because the default number of cores is `1`. Snakemake lower the `threads` for us, it's clever, right?

Now I have shown how to write a rule with customized parameters, also writing a rule that can be used for different outputs, I might write another post regarding more useful features of snakemake, including some python functions helping find input files, cluster execution and auto-generated html reports! Hope you already see some benefits of using snakemake by these toy examples.

## Resources

You can find the example used in this blog [here](https://github.com/yztxwd/Snakemake_workshop)

## Reference

[Official tutorial](https://snakemake.readthedocs.io/en/stable/)
