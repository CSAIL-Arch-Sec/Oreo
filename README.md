# Oreo Artifact

This repository is the artifact of **Oreo: Protecting ASLR Against Microarchitectural Attacks**.

Please check our paper for details:
```
@inproceedings{song-oreo-ndss-2025,
  author = {Song, Shixin and Zhang, Joseph and Yan, Mengjia},
  title = {Oreo: Protecting ASLR Against Microarchitectural Attacks},
  booktitle = {The Network and Distributed System Security (NDSS) Symposium 2025},
  series = {NDSS 2025},
  year = {2025}
}
```

## Supported Platforms

The artifact should work on most modern Linux distributions and CPUs. 
It has been currently tested on the following platforms:

### CPUs

* Intel(R) Xeon(R) Gold 5220R 
* Intel(R) Core(TM) i9-14900K
* AMD Ryzen Threadripper 2970WX 


### Linux distributions

* Ubuntu 22.04 (Linux kernel 5.15)
* Manjaro Linux 24.0 (Linux kernel 6.1 and 6.6)


## Dependencies

The following software dependencies should be correctly installed and configured
before evaluating the artifact:

* Docker v24+ (with `docker-compose` v1.29+)
* Vagrant v2.4
* VirtualBox v7.0
* SPEC 2017 Benchmarks (`cpu2017-1.1.9.iso` is not included in the artifact because of copyright issues)

## Artifact Structure 

The artifact consists of three main parts, which are managed by git submodules:

* **gem5-new** is a modified version of [gem5 v24.0](https://github.com/gem5/gem5/releases/tag/v24.0.0.0) and our major changes are in the following subdirectories:
  * `configs/example/gem5_library/gem5-configs`: specialized gem5 config files for the artifact
  * `scripts`: helper scripts to build gem5 and run the experiments
  * `src`: implementation of Oreo's microarchitecture changes
* **linux-new** is a modified version of [Linux Kernel v6.6](https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux/+/refs/tags/v6.6) and our major changes are in the following subdirectories:
  * `compile_scripts`: helper scripts to build the Linux kernel
  * other subdirectories: implementation of Oreo's kernel changes
* **experiments** contains a `Vagrantfile` that automatically setups an Ubuntu 24.04 virtual machine with Vagrant and VirtualBox. The disk image of the VM is then converted to `raw` format with `qemu-img` for gem5 simulation. The experiment scripts are included in the VM.

Before evaluating the artifact, run the following commands to clone the git submodules from our public repositories:

```shell
git submodule update --init --recursive
```

**_NOTE:_** `cpu2017-1.1.9.iso` is the official image for SPEC 2017 benchmarks.
It is not released with the artifact since it is not open-source.
You should obtain it by yourself and place it in the root directory of the artifact.

The file structure should be as follows:
```
artifact
    | experiments
    | gem5-new
    | linux-new
    | cpu2017-1.1.9.iso
    | ...
```

### Start docker
In the artifact root directory, setup and enter the docker by running:
```shell
cd linux-new
docker-compose up -d
docker-compose exec x86_fs /bin/bash
```
If you are using a newer version of Docker and `docker-compose`, you may need to replace `docker-compose` with `docker compose`.

### Build Linux
In the docker's shell, build Linux by running:
```shell
cd /root/linux
python3 compile_scripts/compile.py --num-cores=80
```

### Build gem5
In the docker's shell, build gem5 by running:
```shell
cd /root/gem5
python3 scripts/compile.py --num-cores=80
```

### Build disk image
We use gem5 full system mode simulation, so we need to put all experiment files into a disk image.
We use `Vagrant` and `VirtualBox` to generate the disk image. 
Note that in this part, we run all commands on the host machine (not in the docker).

In the artifact root directory, pack Linux source files for faster disk image build.
```bash
tar -cvf linux-new.tar linux-new
```
Run the following commands to generate the disk image, which takes about 1 hour on an Intel(R) Xeon(R) Gold 5220R CPU.
```bash
cd experiments/disk-image
vagrant up
vagrant halt # To shut down the current VM
vagrant global-status --prune # To shut down other VMs
```
Next, we convert VirtualBox image to the raw image for gem5
```bash
ls ~/VirtualBox\ VMs
# Find vm directory denoted as <vm-directory>
cd experiments/disk-image
qemu-img convert -f vmdk -O raw ~/VirtualBox\ VMs/<vm-directory>/ubuntu-jammy-22.04-cloudimg.vmdk experiments.img
```
The disk image is then generated at `experiments/disk-image/experiments.img`.

## Experiments
### Basic Workflow
We run our experiments in the docker in the working directory `/root/gem5`. 
Each experiment follows the following workflow:
1. Boot Linux with gem5 KVM with different setups (baseline and Oreo) and generate a checkpoint.
2. Resume simulation using gem5 O3 to simulate the baseline and Oreo with a script specifying which program/benchmark to run.

### Run Example Programs
Run the following command to simulate two example programs `hello` and `hello_invalid`.
Their source code can be found at `/root/experiments/disk-image/experiments/experiments/src`.
```shell
python3 scripts/gen_checkpoint.py && python3 scripts/run_example.py
```
`hello` is valid program, so gem5 should finish simulation without any exceptions. 
Check `result/restore_ko_111_0c0c00/hello_/board.pc.com_1.device` for its output.
`hello_invalid` is a malicious program that accesses an invalid address. 
In the directory `result/restore_ko_111_0c0c00/hello_invalid_`, 
check `board.pc.com_1.device` for its output; 
check `stderr.log` for gem5 exception message on committing invalid memory accesses.
The exception message can be located by running:
```bash
cat result/restore_ko_111_0c0c00/hello_invalid_/stderr.log | grep "ASLR violation"
```

### Run Performance Evaluation with SPEC2017 IntRate
In the paper we use the following commands to run SPEC2017 benchmarks and parse the results.
With the following parameters, we warm up each benchmark with 10 billion user instructions
and measure the performance of the following 1 billion instructions.
```shell
python3 scripts/run_spec.py --gen-cpt --begin-cpt=1 --num-cpt=1 --num-cores=80 --user-delta=32 --spec-inst-warmup-step=10
python3 scripts/parse_spec.py --parse-raw --begin-cpt=1 --num-cpt=1 --roi-idx=11 --expected-stats=12
python3 scripts/parse_spec.py --begin-cpt=1 --num-cpt=1 --roi-idx=11 --expected-stats=12
```
The above full experiments takes several days to finish.
For artifact evaluation, we offer a scaled-down option to warm up with 1 billion user instructions as follows.
```shell
python3 scripts/run_spec.py --gen-cpt --begin-cpt=1 --num-cpt=1 --num-cores=80 --user-delta=32 --spec-inst-warmup-step=1
python3 scripts/parse_spec.py --parse-raw --begin-cpt=1 --num-cpt=1 --roi-idx=2 --expected-stats=3
python3 scripts/parse_spec.py --begin-cpt=1 --num-cpt=1 --roi-idx=2 --expected-stats=3
```
Check `scripts/spec_output/merge_input_mean_user_cpi_1_2.pdf` for CPI overhead introduced by Oreo.

### Run Performance Evaluation with LEBench
```shell
# Run the experiments
python3 scripts/run_perf.py --gen-cpt --num-cpt=8 --begin-cpt=0 --num-cores=80
# Parse the output
python3 scripts/parse_perf.py --suffix-range=0,8 --plot --parse
```
Check output files at `/root/gem5/scripts/plot`.

### Run Security Evaluation
```shell
# Run the prefetch attack, and attack gadgets for leakge paths 1 and 2
python3 scripts/gen_checkpoint.py && python3 scripts/run_sec.py
# Parse output of the prefetch attack
python3 scripts/parse_prefetch.py
# Parse the output trace of attack gadgets for leakage paths 1 and 2
python3 scripts/parse_trace.py
```
Check output files at `/root/gem5/scripts/plot`.

### Ratio of Masked Memory Accesses
The masked ratio of SPEC benchmarks can be found in file `scripts/spec_output/merge_input_oreo_user_cpi_1_2.csv`.
Run the following commands, and the masked ratio of LEBench can be found in file `scripts/lebench_output/test_mask_ratio.csv`.
```bash
python3 scripts/parse_perf_stats.py --begin-cpt=0 --num-cpt=8
```
