---
title: Faster and More Reliable Hugging Face Downloads Using aria2 and GNU Parallel
description: Discover how to boost your Hugging Face download speeds and reliability with aria2 and GNU Parallel. This guide offers step-by-step techniques, configuration tips, and practical examples to streamline your workflow for downloading machine learning models and datasets.
tags: huggingface, aria2, gnu-parallel, machine-learning, performance
cover_image: ''
canonical_url: null
published: false
---

## Summary

- Speed up and improve the reliability of your Hugging Face downloads using `aria2` and `GNU Parallel`.
- To download Hugging Face models, use `aria2` to download files in parallel. If errors occur during the download, you can resume the download from where it left off.
- To quickly verify the integrity of the downloaded files, use `GNU Parallel` to compute the SHA256 hashes in parallel using multiple CPU cores.

## Introduction

Downloading machine learning models and datasets from Hugging Face is time-consuming and unreliable. It is especially slow when dealing with large files or slow internet connections. Follow this guide to speed up and improve the reliability of your Hugging Face downloads using two powerful command-line tools: `aria2` and `GNU Parallel`.

## Prerequisites

Before we get started, make sure you have the following tools installed on your system:

- [Git Large File Storage (git-lfs)](https://git-lfs.com/): An open source Git extension for versioning large files.
- [aria2](https://aria2.github.io/): A lightweight multi-protocol & multi-source command-line download utility.
- [GNU Parallel](https://www.gnu.org/software/parallel/): A shell tool for executing jobs in parallel using one or more computers.

Conda, Ubuntu, and macOS users can install these tools using the following commands:

### Conda Environment

```bash
source ~/miniconda3/bin/activate
conda create -n hf_dl -y
conda activate hf_dl

conda install conda-forge::git-lfs conda-forge::aria2 conda-forge::parallel -y
```

### Ubuntu

```bash
sudo apt install git-lfs aria2 parallel -y
```

### macOS

```bash
brew install git-lfs aria2 parallel
brew install coreutils  # for sha256sum command
```

## Downloading Hugging Face Models

In this section, we will see how to download Hugging Face models (e.g. `perplexity-ai/r1-1776`) using `aria2`.

First, let's clone a Hugging Face repository using `git`. To avoid downloading the large files, we set the `GIT_LFS_SKIP_SMUDGE` environment variable to `1`.

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone https://huggingface.co/perplexity-ai/r1-1776
cd r1-1776
```

The `git lfs ls-files` command lists the files tracked by `git-lfs`. With the `-l` option it will show the OID (SHA256 hash) and the filename. We will use this information to download the files using `aria2` and compute the SHA256 hashes to verify the integrity of the downloaded files.

```bash
git lfs ls-files -l
# 59e42946865f05124b81da4d864f0a0b453c8e6c838f1cb3771c894c0bdba2d9 - model-00001-of-00252.safetensors
# a602965ac1a2dc7e32c6ecde2cf1bc7550d2edb29e0bcf312c6f452733161c15 - model-00002-of-00252.safetensors
# ...
```

With the `-n` option, `git lfs ls-files` will only show the filenames.

```bash
git lfs ls-files -n
# model-00001-of-00252.safetensors
# model-00002-of-00252.safetensors
# ...

git lfs ls-files -n | wc -l  # 252
```

Next, we create a list of files (`files.txt`) to download with `aria2`. We use `xargs` to generate the download URL and the output filename for the list.

```bash
git lfs ls-files -n | xargs -d "\n" -I {} echo -e "https://huggingface.co/perplexity-ai/r1-1776/resolve/main/{}\n    out={}" >> files.txt

head files.txt
# https://huggingface.co/perplexity-ai/r1-1776/resolve/main/model-00001-of-00252.safetensors
#     out=model-00001-of-00252.safetensors
# https://huggingface.co/perplexity-ai/r1-1776/resolve/main/model-00002-of-00252.safetensors
#     out=model-00002-of-00252.safetensors
# https://huggingface.co/perplexity-ai/r1-1776/resolve/main/model-00003-of-00252.safetensors
#     out=model-00003-of-00252.safetensors
# https://huggingface.co/perplexity-ai/r1-1776/resolve/main/model-00004-of-00252.safetensors
#     out=model-00004-of-00252.safetensors
# https://huggingface.co/perplexity-ai/r1-1776/resolve/main/model-00005-of-00252.safetensors
#     out=model-00005-of-00252.safetensors

wc -l files.txt  # 504 files.txt
```

Before downloading the files, we need to remove the files that are already in the directory. Otherwise, `aria2` will add a suffix to the downloaded files.

```bash
git lfs ls-files -n | xargs -d '\n' rm
```

Finally, we download the files using `aria2`. The `-j` option specifies the number of simultaneous downloads. The appropriate values will depend on your network speed and the server's capabilities, but I recommend starting with 4 to around 12. Be careful not to hit the server's rate limit.

```bash
time aria2c -j 8 -i files.txt
```

## Verifying the SHA256 Hashes

After downloading the files, we need to verify the SHA256 hashes to ensure the integrity of the files. We use the `sha256sum` command to calculate the SHA256 hash of each file and compare it with the expected hash.

Unfortunately, `sha256sum` takes longer time to compute the hash for large files. We can speed up the process by using GNU Parallel (`parallel` command) to compute the hashes in parallel.

First, we create files to store the expected SHA256 hashes for each file.

```bash
git lfs ls-files -l | awk '{print $1 "  " $3 > $3".sha256"}'

wc -l *.sha256  # 252 total

cat model-00001-of-00252.safetensors.sha256
# 59e42946865f05124b81da4d864f0a0b453c8e6c838f1cb3771c894c0bdba2d9  model-00001-of-00252.safetensors
```

Let's compute the SHA256 hash of a first file using the `sha256sum` command and measure how long it takes.

```bash
time sha256sum model-00001-of-00252.safetensors
# 59e42946865f05124b81da4d864f0a0b453c8e6c838f1cb3771c894c0bdba2d9  model-00001-of-00252.safetensors
#
# real    0m4.483s
# user    0m3.735s
# sys     0m0.734s
```

It takes about 4 seconds to compute the SHA256 hash for a single file. We can speed up the process by using GNU Parallel (`parallel` command) to compute the hashes in parallel using multiple CPU cores. The `-j` option specifies the number of parallel jobs to run. You can set it to the number of CPU cores on your system. In Linux, you can use the `nproc` command to find out the number of CPU cores.

```bash
time find . -name "*.sha256" | sort | parallel -j 8 -u "sha256sum -c {} 2>&1" | tee sha256sum.log
# model-00214-of-00252.safetensors: OK
# model-00186-of-00252.safetensors: OK
# ...
#
# real    0m46.358s
# user    19m46.452s
# sys     63m22.175s

wc -l sha256sum.log  # 252 sha256sum.log
sort sha256sum.log | nl | less  # should be all OK
```

## Conclusion

- By using `aria2` to download files in parallel and `GNU Parallel` to compute the SHA256 hashes in parallel, you can speed up and improve the reliability of your Hugging Face downloads.
- These tools are especially useful when dealing with large files or unstable internet connections.
- Remember to adjust the number of parallel downloads and jobs based on your network speed and the server's capabilities.

## Citation

```bibtex
@software{tange_2024_14550073,
      author       = {Tange, Ole},
      title        = {GNU Parallel 20241222 ('Bashar')},
      month        = Dec,
      year         = 2024,
      note         = {{GNU Parallel is a general parallelizer to run
                       multiple serial command line programs in parallel
                       without changing them.}},
      publisher    = {Zenodo},
      doi          = {10.5281/zenodo.14550073},
      url          = {https://doi.org/10.5281/zenodo.14550073}
}
```
