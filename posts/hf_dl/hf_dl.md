---
title: Faster and More Reliable Hugging Face Downloads Using aria2 and GNU Parallel
description: 'This guide offers step-by-step instructions on how to speed up and improve the reliability of your Hugging Face downloads using two powerful command-line tools: aria2 and GNU Parallel.'
tags: 'huggingface, llm, python, ai'
cover_image: ./hf-logo-1000-400.png
canonical_url: null
published: true
id: 2294102
date: '2025-03-23T04:16:34Z'
---

## Summary

- Faster and more reliable hugging face downloads with `aria2` and `GNU Parallel`.
- Use `aria2` to download Hugging Face models and datasets in parallel. If errors occur during the download, you can resume the download from where it left off.
- Use `GNU Parallel` to quickly verify the hashes of the downloaded files in parallel using multiple CPU cores.

## Introduction

Downloading machine learning models and datasets from Hugging Face is time-consuming and unreliable. It is especially slow when dealing with large files or unstable internet connections. Follow this guide to speed up and improve the reliability of your Hugging Face downloads using two powerful command-line tools: `aria2` and `GNU Parallel`.

## Prerequisites

Before we get started, make sure you have the following tools installed on your system:

- [Git Large File Storage (git-lfs)](https://git-lfs.com/): An open source Git extension for versioning large files.
- [aria2](https://aria2.github.io/): A lightweight multi-protocol & multi-source command-line download utility.
- [GNU Parallel](https://www.gnu.org/software/parallel/): A shell tool for executing jobs in parallel using one or more computers.
- [sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html): A command to compute checksums of files using the SHA-256 algorithm. *Note: This command is available on typical Linux distributions. macOS's users can install it using Homebrew.*

Ubuntu, macOS or Conda users can install these tools using the following commands:

### Ubuntu

```bash
sudo apt install git-lfs aria2 parallel -y
```

### macOS

```bash
brew install git-lfs aria2 parallel
brew install coreutils  # for sha256sum command
```

### Conda Environment

```bash
source ~/miniconda3/bin/activate
conda create -n hf_dl -y
conda activate hf_dl

conda install conda-forge::git-lfs conda-forge::aria2 conda-forge::parallel -y
```

## Downloading Hugging Face Models

In this section, we will see how to download Hugging Face models (e.g. `Qwen/Qwen2.5-72B`) using `aria2`.

First, let's clone a Hugging Face repository using `git`. To avoid downloading the large files, we set the `GIT_LFS_SKIP_SMUDGE` environment variable to `1`.

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone https://huggingface.co/Qwen/Qwen2.5-72B
cd Qwen2.5-72B
```

The `git lfs ls-files` command lists the files tracked by `git-lfs`. With the `-l` option it will show the OID (SHA256 hash) and the filename. We will use this information to download the files using `aria2` and to verify the SHA256 hashes.

```bash
git lfs ls-files -l
# f4ee40f3b260372082ee0cb2aff1dcdbbe310322651a6ad36327536cdcdf3f40 - model-00001-of-00037.safetensors
# f597885404ed877706e6b78aa14147002169549c6e6a2e344ca4132365464be0 - model-00002-of-00037.safetensors
# 482d4ff5fca9e5be7bc4285d806fcb31582798f20881be23b02270baa37b1d6d - model-00003-of-00037.safetensors
# 896add67b07b2770692df6ff84eee73ec36599c460cf77069e25c860dad18397 - model-00004-of-00037.safetensors
# b21a29eba64b550a11276653eee792bead634d6d35dd9e42fc8c985b53fd501c - model-00005-of-00037.safetensors
# ...
```

With the `-n` option, `git lfs ls-files` will only show the filenames.

```bash
git lfs ls-files -n
# model-00001-of-00037.safetensors
# model-00002-of-00037.safetensors
# model-00003-of-00037.safetensors
# model-00004-of-00037.safetensors
# model-00005-of-00037.safetensors
# ...

git lfs ls-files -n | wc -l  # 37
```

Next, we create a list of files (`files.txt`) to download with `aria2`. We use `xargs` to generate the download URL and the output filename for the list.

```bash
git lfs ls-files -n | xargs -d "\n" -I {} echo -e "https://huggingface.co/Qwen/Qwen2.5-72B/resolve/main/{}\n    out={}" >> files.txt

head files.txt
# https://huggingface.co/Qwen/Qwen2.5-72B/resolve/main/model-00001-of-00037.safetensors
#     out=model-00001-of-00037.safetensors
# https://huggingface.co/Qwen/Qwen2.5-72B/resolve/main/model-00002-of-00037.safetensors
#     out=model-00002-of-00037.safetensors
# https://huggingface.co/Qwen/Qwen2.5-72B/resolve/main/model-00003-of-00037.safetensors
#     out=model-00003-of-00037.safetensors
# https://huggingface.co/Qwen/Qwen2.5-72B/resolve/main/model-00004-of-00037.safetensors
#     out=model-00004-of-00037.safetensors
# https://huggingface.co/Qwen/Qwen2.5-72B/resolve/main/model-00005-of-00037.safetensors
#     out=model-00005-of-00037.safetensors

wc -l files.txt  # 74 files.txt
```

Before downloading the files, we need to remove the files that are already in the directory. Otherwise, `aria2` will add a suffix to the downloaded files.

```bash
git lfs ls-files -n | xargs -d '\n' rm
```

If the model or dataset requires authentication, you will need to log in to Hugging Face using the `huggingface-cli login` command. This command will store the authentication token in the file `~/.cache/huggingface/token`. We can use this token to download the files using `aria2`.

```bash
huggingface-cli login
```

Finally, we download the files using `aria2`. The `-j` option specifies the number of simultaneous downloads. The appropriate values will depend on your network speed and the server's capabilities, but I recommend starting with 4 to around 12. Be careful not to hit the server's rate limit.

```bash
aria2c -j 8 -i files.txt --header="Authorization: Bearer $(cat ~/.cache/huggingface/token)"
```

## Verifying the SHA256 Hashes

After downloading the files, we need to verify the SHA256 hashes to ensure the integrity of the files. We use the `sha256sum` command to calculate the SHA256 hash of each file and compare it with the expected hash.

Unfortunately, `sha256sum` takes longer time to compute the hash for large files. We can speed up the process by using GNU Parallel (`parallel` command) to compute the hashes in parallel.

First, we create files to store the expected SHA256 hashes for each file.

```bash
git lfs ls-files -l | awk '{print $1 "  " $3 > $3".sha256"}'

find . -name "*.sha256" -print | wc -l  # 37

cat model-00001-of-00037.safetensors.sha256
# f4ee40f3b260372082ee0cb2aff1dcdbbe310322651a6ad36327536cdcdf3f40  model-00001-of-00037.safetensors
```

Let's compute the SHA256 hash of a first file using the `sha256sum` command.

```bash
sha256sum model-00001-of-00037.safetensors
# f4ee40f3b260372082ee0cb2aff1dcdbbe310322651a6ad36327536cdcdf3f40  model-00001-of-00037.safetensors
```

We can speed up the process by using GNU Parallel (`parallel` command) to compute the hashes in parallel using multiple CPU cores. The `-j` option specifies the number of parallel jobs to run. You can set it to the number of CPU cores on your system. In Linux, you can use the `nproc` command to find out the number of CPU cores.

```bash
time find . -name "*.sha256" -print | sort | parallel -j 8 -u "sha256sum -c {} 2>&1" | tee sha256sum.log
# model-00001-of-00037.safetensors: OK
# model-00007-of-00037.safetensors: OK
# model-00003-of-00037.safetensors: OK
# ...
```

Let's check the contents of the file `sha256sum.log`. It should contain the results of the SHA256 hash verification for each file. The `OK` message indicates that the hash verification was successful.

```bash
wc -l sha256sum.log  # 37 sha256sum.log

sort sha256sum.log | nl
#      1  model-00001-of-00037.safetensors: OK
#      2  model-00002-of-00037.safetensors: OK
#      3  model-00003-of-00037.safetensors: OK
# ...
#     35  model-00035-of-00037.safetensors: OK
#     36  model-00036-of-00037.safetensors: OK
#     37  model-00037-of-00037.safetensors: OK
```

OK! All the files have been successfully downloaded and verified!

## Conclusion

- By using `aria2` to download files in parallel and `GNU Parallel` to compute the SHA256 hashes in parallel, you can speed up and improve the reliability of your Hugging Face downloads.
- These tools are particularly useful when dealing with large files and/or unstable internet connections.
- Remember to adjust the number of parallel downloads or jobs based on your network speed and the server capabilities.

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
