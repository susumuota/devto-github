---
title: Faster and More Reliable Hugging Face Downloads Using aria2 and GNU Parallel
description: 'This guide offers step-by-step techniques, configuration tips, and practical examples to streamline your workflow for downloading machine learning models and datasets.'
tags: 'huggingface, llm, machinelearning, python, ai'
cover_image: ''
canonical_url: null
published: false
id: 2294102
---

## Summary

- Faster and more reliable hugging face downloads with `aria2` and `GNU Parallel`.
- Use `aria2` to download Hugging Face models and datasets in parallel. If errors occur during the download, you can resume the download from where it left off.
- Use `GNU Parallel` to quickly verify the hashes of the downloaded files in parallel using multiple CPU cores.

## Introduction

Downloading machine learning models and datasets from Hugging Face is time-consuming and unreliable. It is especially slow when dealing with large files or slow internet connections. Follow this guide to speed up and improve the reliability of your Hugging Face downloads using two powerful command-line tools: `aria2` and `GNU Parallel`.

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

In this section, we will see how to download Hugging Face models (e.g. `mistralai/Mistral-Small-24B-Instruct-2501`) using `aria2`.

First, let's clone a Hugging Face repository using `git`. To avoid downloading the large files, we set the `GIT_LFS_SKIP_SMUDGE` environment variable to `1`.

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone https://huggingface.co/mistralai/Mistral-Small-24B-Instruct-2501
cd Mistral-Small-24B-Instruct-2501
```

The `git lfs ls-files` command lists the files tracked by `git-lfs`. With the `-l` option it will show the OID (SHA256 hash) and the filename. We will use this information to download the files using `aria2` and to verify the SHA256 hashes.

```bash
git lfs ls-files -l
# 82718244a767b5b3545a46736801a0dfbdc4aacb581f7cfd9c08a2bdfdd4333e - consolidated.safetensors
# 75a14c708eea501700a723dc74bc886cf36a1393686a3fb098ee106b160da32f - model-00001-of-00010.safetensors
# 1ff40fbfd9e042b7dab3f3c9442f870a4701f53e394dda769807a160ba40f32a - model-00002-of-00010.safetensors
# 4cc2d059fded71efd2947a414f32053b4ed3fa84383edf97b6d91fd9f04e4235 - model-00003-of-00010.safetensors
# ...
```

With the `-n` option, `git lfs ls-files` will only show the filenames.

```bash
git lfs ls-files -n
# consolidated.safetensors
# model-00001-of-00010.safetensors
# model-00002-of-00010.safetensors
# model-00003-of-00010.safetensors
# ...

git lfs ls-files -n | wc -l  # 13
```

Next, we create a list of files (`files.txt`) to download with `aria2`. We use `xargs` to generate the download URL and the output filename for the list.

```bash
git lfs ls-files -n | xargs -d "\n" -I {} echo -e "https://huggingface.co/mistralai/Mistral-Small-24B-Instruct-2501/resolve/main/{}\n    out={}" >> files.txt

head files.txt
# https://huggingface.co/mistralai/Mistral-Small-24B-Instruct-2501/resolve/main/consolidated.safetensors
#     out=consolidated.safetensors
# https://huggingface.co/mistralai/Mistral-Small-24B-Instruct-2501/resolve/main/model-00001-of-00010.safetensors
#     out=model-00001-of-00010.safetensors
# https://huggingface.co/mistralai/Mistral-Small-24B-Instruct-2501/resolve/main/model-00002-of-00010.safetensors
#     out=model-00002-of-00010.safetensors
# https://huggingface.co/mistralai/Mistral-Small-24B-Instruct-2501/resolve/main/model-00003-of-00010.safetensors
#     out=model-00003-of-00010.safetensors
# https://huggingface.co/mistralai/Mistral-Small-24B-Instruct-2501/resolve/main/model-00004-of-00010.safetensors
#     out=model-00004-of-00010.safetensors

wc -l files.txt  # 26 files.txt
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

wc -l *.sha256  # 13 total

cat model-00001-of-00010.safetensors.sha256
# 75a14c708eea501700a723dc74bc886cf36a1393686a3fb098ee106b160da32f  model-00001-of-00010.safetensors
```

Let's compute the SHA256 hash of a first file using the `sha256sum` command and measure how long it takes.

```bash
time sha256sum model-00001-of-00010.safetensors
# 75a14c708eea501700a723dc74bc886cf36a1393686a3fb098ee106b160da32f  model-00001-of-00010.safetensors
#
# real    0m4.073s
# user    0m3.373s
# sys     0m0.692s
```

It takes about 4 seconds to compute the SHA256 hash for a single file. We can speed up the process by using GNU Parallel (`parallel` command) to compute the hashes in parallel using multiple CPU cores. The `-j` option specifies the number of parallel jobs to run. You can set it to the number of CPU cores on your system. In Linux, you can use the `nproc` command to find out the number of CPU cores.

```bash
time find . -name "*.sha256" | sort | parallel -j 8 -u "sha256sum -c {} 2>&1" | tee sha256sum.log
# model-00005-of-00010.safetensors: OK
# model-00003-of-00010.safetensors: OK
# model-00002-of-00010.safetensors: OK
# ...

wc -l sha256sum.log  # 13 sha256sum.log
sort sha256sum.log | nl | less  # should be all OK
```

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
