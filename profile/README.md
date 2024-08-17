# TrEnv: Transparently Share Serverless Execution Environments Across Different Functions and Nodes

This organization contains the source codes of our paper @ SOSP: TrEnv: Transparently Share Serverless Execution Environments Across Different Functions and Nodes. In the following, I will give instructions on how to depoly TrEnv.

It contains a modified Linux kernel, the modified container runtime (runc, containerd and faasd), the modified CRIU and test scripts.

For the details of TrEnv, please refer to our paper.

Table of Contents
=================

* [Hardware Requirement](#hardware-requirement)
  * [CPU](#cpu)
  * [Memory device](#memory-device)
  * [Our Hardware Setup](#our-hardware-setup)
* [Software Installation](#software-installation)
  * [Kernel](#kernel)
  * [Container Runtime](#container-runtime)
  * [CRIU](#criu)
  * [function-specific dependencies](#function-specific-dependencies)
  * [Baselines](#baselines)
  * [Others](#others)
* [Run TrEnv](#run-trenv)
  * [Step 1](#step-1)
  * [Step 2](#step-2)
  * [Step 3](#step-3)
* [Benchmark in Paper](#benchmark-in-paper)

## Hardware Requirement

A single server-level machine is enough to run TrEnv and the evaluation in our paper.

### CPU

First, TrEnv has only been implemented on x86_64 and has only been tested on Intel, so make sure you are using Intel CPU.

Since we has not considered the virtualization of the CXL devices, we recommend that you run TrEnv on a **bare metal** machine (i.e., without running inside a VM).

We recommend using a machine with **as many cores as possible** (in our setup, there are dual 32-core CPUs, which are 128 threads in total). The workloads used in our evaluation will execute many function requests concurrently. If your CPUs become the bottleneck, you will get a poor result.

### Memory device

TrEnv needs a special memory device, which is CXL. For test purpose, a CXL 1.1 device is enough for running on the single machine. If you would like to run TrEnv on multiple machines, you need a special CXL device which has multiple CXL interfaces (like multi-headed device), so that can be connected to multiple machines simultaneously. Besides, TrEnv **does not** need read-write cache coherence supported in CXL 3.0.

The multiple machines are almost independent from each other. They just needs (1) put the memory images on CXL memory by any one of servers and (2) setup their mm-templates to point to the same locations on CXL memory. After that they can run independently, as there is no more write or synchronization to CXL memory

However, if you do not have Type-3 CXL devices, **Intel Optane Persistent Memory (PMem) is also fine for a simulated execution**, and the software setup is the same.

We recommend the local DRAM capacity is >= 128 GB. There is no constraint of the ratio between local DRAM and CXL memory. As long as the CXL memory can store the memory snapshots (4 GB is enough for our entire evaluated functions). However, the REAP and FaaSnap needs more capacity for CXL memory to store the memory images (which needs around 55 GB for FaaSnap and round 27 GB for REAP).

Finally, TrEnv supports RDMA through RXE (soft roce), so there is no need for hardware RDMA NIC.

*Note that the RDMA support is based on Fastswap @ EuroSys'20, with some modifications to accommodate with mm-template. It is easy to adapt TrEnv for the hardware RDMA.*

### Our Hardware Setup

The hardware used for the evaluation in paper is:

- A two-socket 32-core Intel XeonGold 6454S CPU, enable the hyper-threading.

- 256 GB Local DRAM.

- An 128 GB early-stage Samsung CXL device. Its access latency is 600 ns, which is far slower than current CXL device (around 200-300 ns), and even slower than PMem.

- A local-to-local RXE, whose 4 KB latency is around 6 us, which is comparable with hardware RDMA environment.

## Software Installation

In the following sections, I will give you instructions on how to install and run TrEnv.

The instructions I give are **based on Ubuntu 22.04 as root user**. We also tested it on the CentOS-based environment (but the actual dependency to install has some differences with Ubuntu). If you have any questions on installation on CentOS-based environment, you can contact with me to see if I can help.

### Kernel

TrEnv has modified the Linux kernel, so the first step is to compile and install our kernel.

Note that all DAX-related option (including `CONFIG_DAX | CONFIG_NVDIMM_DAX | CONFIG_DEV_DAX | CONFIG_DEV_DAX_PMEM | ...`) in the kernel config must be `y` (built-in) instead of just `m` (kernel module)

```bash
sudo apt install libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm
# clone our kernel source code
git clone https://github.com/switch-container/linux.git
cd linux
# compile the kernel, make sure you have enable the standard server-level server kernel options, plus the following:
# 1. the DAX driver such as DEV_DAX
# 2. CXL (if you are using CXL device) or PMEM driver
# 3. RXE driver

# zcat /proc/config.gz > .config
cp /boot/config-5.15.0-113-generic .config
make menuconfig
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
make -j32
make modules_install
make install
```

Next configure the grub to boot our kernel.

```bash
grep submenu /boot/grub/grub.cfg
# example output:
# submenu 'Advanced options for Ubuntu' $menuentry_id_option 'gnulinux-advanced-...
grep menuentry /boot/grub/grub.cfg
# example output:
# menuentry 'Ubuntu, with Linux 6.1.0-rc8+' --class ubuntu --class gnu-linux --class gnu ...

# Note the actual argument should match the output of the two above commands
sed -i 's/GRUB_DEFAULT=.*$/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.1.0-rc8+"/' /etc/default/grub
update-grub
# reboot the machine
reboot

# confirm our new kernel is running
uname -a
# should output string contains: 6.1.0-rc8
ls /dev | grep pseudo_mm
# should output string pseudo_mm
```

### Container Runtime

TrEnv has modified the container runtime (including runc, containerd and faasd).

```bash
# Install golang
wget https://go.dev/dl/go1.21.13.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.21.13.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
# verify that golang has been installed
source ~/.bashrc && go version

# install dependency
apt install pkg-config libseccomp-dev 

# clone ctr runtime repo and compile
mkdir -p ~/go/src/github.com && cd ~/go/src/github.com
git clone https://github.com/switch-container/runc.git opencontainers/runc
git clone https://github.com/switch-container/containerd.git containerd/containerd
git clone https://github.com/switch-container/go-criu.git checkpoint-restore/go-criu
git clone https://github.com/switch-container/faasd.git openfaas/faasd
git clone https://github.com/switch-container/faas-provider.git openfaas/faas-provider
git clone https://github.com/switch-container/faas-cli.git openfaas/faas-cli

cd opencontainers/runc && make runc && make install && cd -
cd containerd/containerd && make BUILDTAGS=no_btrfs && make install && cd -
cd openfaas/faasd && make local && make install && cd -
cd openfaas/faas-cli && make local-install && cp /root/go/bin/faas-cli /usr/local/bin/faas-cli && cd -
```

Apart from container runtime, we need install CNI (Container Network Interface) to help to configure the network of our containers.

```bash
ARCH=amd64
CNI_VERSION=v1.3.0
mkdir -p /opt/cni/bin
curl -sSL https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-${ARCH}-${CNI_VERSION}.tgz | tar -xz -C /opt/cni/bin
mkdir -p /etc/cni/net.d/
cat >/etc/cni/net.d/10-openfaas.conflist <<EOF
{
    "cniVersion": "0.4.0",
    "name": "openfaas-cni-bridge",
    "plugins": [
      {
        "type": "bridge",
        "bridge": "openfaas0",
        "isGateway": true,
        "ipMasq": true,
        "ipam": {
            "type": "host-local",
            "subnet": "10.62.0.0/16",
            "dataDir": "/var/run/cni",
            "routes": [
                { "dst": "0.0.0.0/0" }
            ]
        }
      },
      {
        "type": "firewall"
      }
    ]
}
EOF

cat >/etc/cni/net.d/99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
```

### CRIU

TrEnv has modified the CRIU to utilize mm-template and reuse isolated environment.

```bash
cd /root
git clone https://github.com/switch-container/criu.git
# make sure that the branch is on `switch`
cd criu
apt install libprotobuf-dev libprotobuf-c-dev protobuf-c-compiler protobuf-compiler python3-protobuf iproute2 libcap-dev libnl-3-dev libnet-dev
make -j16 install-criu
# the output binary is located at /root/criu/criu/criu and /usr/local/sbin/criu
mkdir -p /root/downloads && cp /root/criu/criu/criu /root/downloads/switch-criu
```

Note that CentOS-based system may lack some of the packages. The dependencies listed here is the same as CRIU, so please refer to homepage of CRIU to check the instructions on how to install CRIU on your systems.

### function-specific dependencies

Download the function-specific dependencies and unzip it into `/var/lib/faasd/pkgs`:

```bash
cd /root/downloads
wget -O pkgs.tar.gz https://cloud.tsinghua.edu.cn/f/4644dbac9e3a4b309c50/?dl=1
# Note than you cannot change this directory, unless you modify the faasd
mkdir -p /var/lib/faasd/ && tar xf pkgs.tar.gz -C /var/lib/faasd
```

> If you cannot access the above url, consider download from [Google driver here](https://drive.google.com/file/d/14fgijRcjZAXh3yQKN0feskuHXPA7bMB0/view?usp=drive_link)

The following text in this section is about principles instead of instructions on installation. You can skip them and jump to the next Baselines Section if you are not interested.

In TrEnv, we will dynamically mount (and umount) the function-specific dependencies into the mntns during repurposing. To achieve that, we 

1. first need to extract and store each function's dependencies on host

2. mount them as multiple overlayfs during running.

3. When repurposing, we will dynamically bind mount them into the mntns

Besides, to purge the modification of previous function instance in the overlayfs, you must:

1. Remove all files under upper dir and work dir.

2. Remount the overlayfs, i.e., `mount -o remount /path/to/overlay` or `ret = mount(NULL, /path/to/overlay, NULL, MS_REMOUNT, NULL);`

Note that remount is necessary, as it will flush the (inode) cache of previous overlayfs to make the clean upper dir work.

> It will be much easier for another union filesystem, which is AUFS (another unionfs). AUFS support dynamically change the lower directory by remount. However, AUFS has not been merged into Linux mainline. Thus, the Docker has already deprecated its support. To match with the trend of the container runtime, we still choose to reconfigure with overlayfs to demonstrate the flexibility of TrEnv.

### Baselines

To run the baseline in our paper, including CRIU, FaaSnap and REAP, you need to install additional software.

Please refer to the [FaaSnap](https://github.com/ucsdsysnet/faasnap) on how to build its kernel and firecracker.

We also provide the `vmlinux` and `firecracker` build on our system to you. They are able to run directly on Intel Platforms. Note that `vmlinux` should be copied to `/root/faasnap/vmlinux` and `firecracker` should be copied to `/root/faasnap/firecracker`

```bash
mkdir /root/faasnap && cd /root/faasnap && git clone https://github.com/switch-container/faasnap.git
cd /root/faasnap && \
wget -O vmlinux https://cloud.tsinghua.edu.cn/f/ef649f94564e4b40a1c2/?dl=1 && \
wget -O firecracker https://cloud.tsinghua.edu.cn/f/fa90c80489c842608a51/?dl=1 && \
chmod +x vmlinux firecracker

# build the faasnap daemon
go install github.com/go-swagger/go-swagger/cmd/swagger@latest
cd /root/faasnap/faasnap && /root/go/bin/swagger generate server -f api/swagger.yaml
go get ./... && go build cmd/faasnap-server/main.go
```

Next you should build the rootfs of python and nodejs environments.

Note that both `debian-nodejs-rootfs.ext4`  and `debian-python-rootfs.ext4`should be move to `/root/faasnap/rootfs`

```bash
# download image recognition model
cd /root/faasnap/faasnap/rootfs/guest/python/image_recognition
mkdir -p model && cd model && wget https://raw.githubusercontent.com/fregu856/deeplabv3/master/pretrained_models/resnet/resnet50-19c8e357.pth
# build rootfs
apt install debootstrap acl
# If you are not using proxy, you should uncomment at line 8 and 28 of rootfs/scripts/setup-debian-python-rootfs.sh
# and line 11 of rootfs/scripts/setup-debian-nodejs-rootfs.sh
cd /root/faasnap/faasnap && cd rootfs && make debian-rootfs
cd /root/faasnap && mkdir -p rootfs && cd rootfs && mv /root/faasnap/faasnap/rootfs/debian-{python,nodejs}-rootfs.ext4 .
```

If you do not want to build rootfs yourself, we also provide the rootfs build on our system to you. They are able to run directly on Intel Platforms. 

```bash
cd /root/faasnap && mkdir -p rootfs && cd rootfs \
wget -O debian-nodejs-rootfs.ext4.zip https://cloud.tsinghua.edu.cn/f/0b2144137441475495a3/?dl=1 && \
wget -O debian-python-rootfs.ext4.zip https://cloud.tsinghua.edu.cn/f/72ba9d8cdaac4abf8856/?dl=1
apt install unzip && unzip debian-nodejs-rootfs.ext4.zip && unzip debian-python-rootfs.ext4.zip
```

Finally, we need to build the vanilla CRIU (which is on `master` branch) if we want to run baseline of CRIU.

```bash
cd /root/criu && git checkout master && make clean && make -j16 install-criu && \
 cp /root/criu/criu/criu /root/downloads/raw-criu && git checkout switch && make clean
```

### Others

Apart from the above softwares, there are some other softwares needed to be installed.

The first is rdma-server, which is necessary for TrEnv-RDMA to work.

```bash
cd /root
git clone https://github.com/switch-container/rdma-server.git
apt install libibverbs-dev rdma-core librdmacm-dev
cd rdma-server && make
# the output binary is pseudo-mm-rdma-server
```

The second is the test scripts

```bash
# Dependency to configure CXL (or PMem).
# Note that ndctl is only necessary for PMem, so if you are using CXL you can only install daxctl
apt install numactl daxctl ndctl

cd /root
git clone https://github.com/switch-container/utils.git
git clone --branch huang https://github.com/switch-container/faasd-testdriver.git
mkdir -p /root/test && cd /root/test
ln -s /root/faasd-testdriver/ /root/test/faasd-testdriver && \
 ln -s /root/rdma-server/pseudo-mm-rdma-server /root/test/pseudo-mm-rdma-server && \
 ln -s /root/utils/stack.yml /root/test/stack.yml && \
 ln -s /root/faasd-testdriver/functions/template/ /root/test/template

# download two datasets (Azure and Huawei traces)
mkdir -p /root/downloads && cd /root/downloads && \
  wget https://azurepublicdatasettraces.blob.core.windows.net/azurepublicdatasetv2/azurefunctions_dataset2019/azurefunctions-dataset2019.tar.xz && \
  wget https://sir-dataset.obs.cn-east-3.myhuaweicloud.com/datasets/public_dataset/public_dataset.zip
# unzip the dataset
mkdir azurefunction-dataset2019 && tar xf azurefunctions-dataset2019.tar.xz -C azurefunction-dataset2019 && unzip public_dataset.zip
```

The last is the python environment, we recommend to use python version 3.10 (default version for Ubuntu 22.04).

```bash
apt install python3.10-venv
# You should use exactly the following path for venv, unless you modified the test-common.sh in utils repo.
cd /root && mkdir venv && cd venv && python3 -m venv faasd-test
source /root/venv/faasd-test/bin/activate && pip install pyyaml gevent requests pandas numpy matplotlib
```

## Run TrEnv

Before running my instruction, first check your proxy settings. In scripts, I use something like `https_proxy=http://127.0.0.1:7890`. If you are not use the same proxy settings (or do not use proxy at all), consider remove them first.

Or if you meet any errors that show something like `127.0.0.1:7890`, you should update it to your proxy or completely delete them in the corresponding scripts if you do not use any proxy.

### Step 1

After install all necessary software of TrEnv, in the following text, I will instruct you how to run TrEnv and our evaluation.

Luckily, most of the operations have been integrated into the scripts in utils repo, you can read them if you want to know what is happening.

First confirm that there is a CXL device configured as devdax mode, and reconfigure its alignment.

**If you are using PMem, please skip the following first block and refer to the second block to configure it.**

As you might noticed, we use the extra space of memory device as system-ram (CPU-less NUMA). That is used to store the memory images of FaaSnap/REAP/vanilla CRIU. **If you do not want to run baselines, you can skip configure the extra space of memory device as CPU-less NUMA.**

```bash
# Only for CXL
daxctl list -u
# example output:
#{
#   "chardev":"dax0.0",
#   "size":"XXX",
#    "target_node":2,
#   "align":2097152,
#   "mode":"devdax"
# }

# Reconfigure the CXL memory, with the alignment as 4K (instead of 2MB)
# Use the region id of your device (here use 0)
daxctl disable-device -r 0 all
daxctl destroy-device -r 0 all || true
# 16 GB is enough for memory images, this will create device dax0.0
daxctl create-device -r 0 -a 4096 -s 16g

# configure the left space as NUMA node, this will create device dax0.1
daxctl create-device -r 0 -a 4096
daxctl reconfigure-device --mode=system-ram --no-online --human dax0.1
# online memory
daxctl online-memory dax0.1
```

```bash
# Only For PMem
ndctl list -u
# example output:
#[
#    {
#    "dev":"namespace0.0",
#    "mode":"devdax",
#    "map":"dev",
#    "size":"744.19 GiB (799.06 GB)",
#    "uuid":"dfd9add7-3b3e-4577-bc22-43061b61134c",
#    "chardev":"dax0.0",
#    "align":2097152
#  }
#]
ndctl disable-namespace namespace0.0
ndctl destroy-namespace namespace0.0
# Note that we cannot specify 16 GB here, it conflict with the alignment.
# We need something like multiply of 6, such as 6 GB, 12 GB or 24 GB
# this will create device dax0.0
ndctl create-namespace --mode=devdax -s 24g --align 4096 -r 0
daxctl list -u
# example output:
# {
#    "chardev":"dax0.0",
#    "size":"23.25 GiB (24.96 GB)",
#    "target_node":2,
#    "align":4096,
#    "mode":"devdax"
#  }

# configure some space as NUMA node, this will create device dax0.1
ndctl create-namespace --mode=devdax -s 120g --align 4096 -r 0
echo offline > /sys/devices/system/memory/auto_online_blocks
daxctl reconfigure-device --mode=system-ram --human dax0.1
```

No matter CXL or PMem, please also remember the `target_node` output of `daxctl list`. **It should match the last argument (i.e., `node=2`) at line 45 in `machine_prepare.sh` in utils repo.**

### Step 2

After configure the memory device, now it is time to prepare memory pool and mm-templates. The following instructions should run only one time before running the TrEnv (e.g., after reboot the system).

However, if you have already tested CXL/PMem mm-template, and start to test RDMA mm-template, you should run this instruction as well.

```bash
# For CXL or PMem
# Note that the dax device specify here should match your configuration above
# you can use `daxctl` and `ls /dev | grep dax` to confirm that this device exists.
bash machine-prepare.sh --mem-pool dax --dax-dev /dev/dax0.0
# Or for RDMA
bash machine-prepare.sh --mem-pool rdma --nic eth0
```

Note that since TrEnv uses local-to-local RXE, so the `--nic` you specify here does not need to be real nic (but of course it can be).

For example, you can specifiy a interface that does not exists (e.g., `--nic eth-not-exist` or `--nic eth5`), the scripts will ask you to create a dummy device and use it as RXE interface.

*However, if you want to run RDMA server on another machine, you need specify a real nic that can send traffic to that RDMA server. Meanwhile, you should modify the line 45 at `machine-prepare.sh`, change the modprobe `sip` argument to the remote RDMA server instead of the local ip address.*

This scripts will make some check, download our docker image for containers, start generate snapshot and build mm-templates.

The snapshot images of CRIU will be located on a CXL-based (or PMem-based) tmpfs and it is expected. (1) The vanilla CRIU need to restore from tmpfs to compare with TrEnv. (2) The mm-template will copy the memory imagesto memory pools, so when TrEnv start running, it will not access memory images through this tmpfs (but it does need to access other non-memory images, which is very small).

### Step 3

Now it is fine to start test. The execution time depends on the workload. W1 and W2 need around 30 min, while Huawei and Azure traces need around 1 hour.

Note if you want to run FaaSnap and REAP baselines, you need mount a CXL-backed (or PMem-backed) tmpfs at `/mnt/cxl-tmp/` and create a directory  at `/mnt/cxl-tmp/faasnap/snapshot`. Their snapshot images will be located on there.

Besides, the output to terminal (e.g., stderr) of FaaSnap and REAP maybe weird, but that is fine. You can check the output during running test by the file at `/run/test.log`.

```bash
mkdir -p /mnt/cxl-tmp && mount -t tmpfs -o size=100g,mpol=bind:2 tmpfs /mnt/cxl-tmp && mkdir -p /mnt/cxl-tmp/faasnap/snapshot
```

Then you can start run TrEnv.

```bash
# First generate workload: w1, w2, huawei or azure traces
source /root/venv/faasd-test/bin/activate && cd /root/test/faasd-testdriver
# w1, will run about 30 m
python gen_trace.py -w 1
# w2, will run about 30 m
python gen_trace.py -w 2
# huawei, will run about 1 hour
python gen_trace.py -w huawei --dataset /root/downloads/public_dataset/csv_files/requests_minute
# azure, will run about 1 hour
python gen_trace.py -w azure --dataset /root/downloads/azurefunction-dataset2019

cd /root/utils
# then start run test for the above generated workload
# for TrEnv
bash test.sh --mem 64 <TEST_NAME>
# cold start
bash test.sh --gc 10 --mem 64 --baseline --start-method cold <TEST_NAME>
# criu start
bash test.sh --gc 10 --mem 64 --baseline --start-method criu <TEST_NAME>
# faasnap test
bash test.sh --gc 10 --mem 64 --baseline --start-method faasnap <TEST_NAME>
# reap test
bash test.sh --gc 10 --mem 64 --baseline --start-method reap <TEST_NAME>
```

The above `<TEST_NAME>` could be any string. After running, the result files will be located in `/root/test/result/<TEST_NAME>`

Note that:

1. The generated workload will be as json files under `/root/test/faasd-testdriver`. So you can generate one workload, and run TrEnv and baseline multiple times on it.

2. **Please cleanup the environment after each run**. We provide a command to do that: `bash test.sh --clean`

3. For FaaSnap and REAP, each run (i.e., `bash test.sh`) will first generate its snapshot images, which might take a long time. Besides, there is some possibility that the generation is failed (we have not figured out why). The solution is re-execute the `bash test.sh`.

4. For FaaSnap and REAP, after each run, make sure the FaaSnap/REAP daemon and all Firecracker hypervisors have been killed (whose process name is `./main` and `firecracker`)

## Benchmark in Paper

To get the result of RDMA, you need repeat the above Step 2 for generate mm-template with RDMA pool (e.g., `bash machine-prepare.sh --mem-pool rdma`). In the following instructions, the instructions to run TrEnv of CXL-backed mm-template is the same as those of RDMA-backed mm-template.

> In another words, to get the result of CXL-backed mm-template, you need specify `--mem-pool dax` in the Step 2 and execute `bash test.sh ...`. 
> 
> To get the result of RDMA-backed mm-template, you need repeat the Step 2 but with `--mem-pool rdma` and execute `bash test.sh ...` with the same args again.

To get the result of Figure 10 (a)

```bash
source /root/venv/faasd-test/bin/activate && cd /root/test/faasd-testdriver
python gen_trace.py -w 1
cd /root/utils
# then start run test for the above generated workload
bash test.sh --clean && bash test.sh --mem 64 cxl-w1 && bash test.sh --clean
bash test.sh --gc 2 --mem 64 --baseline --start-method cold cold-w1 && bash test.sh --clean
bash test.sh --gc 2 --mem 64 --baseline --start-method criu criu-w1 && bash test.sh --clean
# For FaaSnap and REAP, make sure the daemon (i.e., main) has been killed
bash test.sh --gc 2 --mem 64 --baseline --start-method faasnap faasnap-w1 && bash test.sh --clean && pkill main && pkill firecracker
bash test.sh --gc 2 --mem 64 --baseline --start-method reap reap-w1 && bash test.sh --clean && pkill main && pkill firecracker
```

To get the result of Figure 10 (b)

```bash
source /root/venv/faasd-test/bin/activate && cd /root/test/faasd-testdriver
python gen_trace.py -w 2
cd /root/utils
# then start run test for the above generated workload
bash test.sh --clean && bash test.sh --mem 32 cxl-w2 && bash test.sh --clean
bash test.sh --gc 3 --mem 32 --baseline --start-method cold cold-w2 && bash test.sh --clean
bash test.sh --gc 3 --mem 32 --baseline --start-method criu criu-w2 && bash test.sh --clean
# For FaaSnap and REAP, make sure the daemon (i.e., main) has been killed
bash test.sh --gc 3 --mem 32 --baseline --start-method faasnap faasnap-w2 && bash test.sh --clean && pkill main && pkill firecracker
bash test.sh --gc 3 --mem 32 --baseline --start-method reap reap-w2 && bash test.sh --clean && pkill main && pkill firecracker
```

To get the result of Figure 11 (b), you can:

```bash
bash test.sh --clean
# TrEnv
bash test.sh --no-reuse --mem 80 --no-test --idle-num 50

# baseline
# bash test.sh --no-reuse --baseline --mem 80 --no-bgtask --no-test --start-method criu 

# faasnap
# bash test.sh --no-reuse --baseline --mem 80 --no-bgtask --no-test --start-method faasnap

cd /root/test && faas-cli register -f ./stack.yml -g http://127.0.0.1:8081 && cd -

# name=image-flip-rotate
name=image-recognition
for ((i=0;i<50;i++)); do curl http://127.0.0.1:8081/invoke/${name}; sleep 0.5; done

cat /sys/fs/cgroup/openfaas-fn/memory.peak
# for faasnap or reap
# cat /sys/fs/cgroup/faasnap/memory.peak
# cat /sys/fs/cgroup/reap/memory.peak
```

To get the result of Figure 12, you can

```bash
source /root/venv/faasd-test/bin/activate && cd /root/test/faasd-testdriver
python gen_trace.py -w func
cd /root/utils
# then start run test for the above generated workload
bash test.sh --clean && bash test.sh --mem 64 --idle-num 1 --functional 5 cxl-func && bash test.sh --clean
bash test.sh --gc 20 --no-reuse --mem 64 --functional 5 --baseline --start-method cold cold-func && bash test.sh --clean
bash test.sh --gc 20 --no-reuse --mem 64 --functional 5 --baseline --start-method criu criu-func && bash test.sh --clean
# For FaaSnap and REAP, make sure the daemon (i.e., main) has been killed
bash test.sh --gc 20 --no-reuse --mem 64 --functional 5 --baseline --start-method faasnap faasnap-func && bash test.sh --clean && pkill main  && pkill firecracker
bash test.sh --gc 20 --no-reuse --mem 32 --functional 5 --baseline --start-method reap reap-func && bash test.sh --clean && pkill main && pkill firecracker
```

To get the result of Figure 13, you can

```bash
source /root/venv/faasd-test/bin/activate && cd /root/test/faasd-testdriver
# huawei, will run about 1 hour
python gen_trace.py -w huawei --dataset /root/downloads/public_dataset/csv_files/requests_minute
cd /root/utils
# then start run test for the above generated workload
bash test.sh --clean && bash test.sh --mem 64 cxl-huawei && bash test.sh --clean
bash test.sh --gc 10 --mem 64 --baseline --start-method cold cold-huawei && bash test.sh --clean
bash test.sh --gc 10 --mem 64 --baseline --start-method criu criu-huawei && bash test.sh --clean
# For FaaSnap and REAP, make sure the daemon (i.e., main) has been killed
bash test.sh --gc 10 --mem 64 --baseline --start-method faasnap faasnap-huawei && bash test.sh --clean && pkill main && pkill firecracker
bash test.sh --gc 10 --mem 32 --baseline --start-method reap reap-huawei && bash test.sh --clean && pkill main && pkill firecracker

cd /root/test/faasd-testdriver
python gen_trace.py -w azure --dataset /root/downloads/azurefunction-dataset2019
cd /root/utils
# then start run test for the above generated workload
bash test.sh --clean && bash test.sh --mem 64 cxl-azure && bash test.sh --clean
bash test.sh --gc 10 --mem 64 --baseline --start-method cold cold-azure && bash test.sh --clean
bash test.sh --gc 10 --mem 64 --baseline --start-method criu criu-azure && bash test.sh --clean
# For FaaSnap and REAP, make sure the daemon (i.e., main) has been killed
bash test.sh --gc 10 --mem 64 --baseline --start-method faasnap faasnap-azure && bash test.sh --clean && pkill main && pkill firecracker
bash test.sh --gc 10 --mem 32 --baseline --start-method reap reap-azure && bash test.sh --clean && pkill main && pkill firecracker
```

The data of Figure 15 is also from azure trace test.

The code for generating the figures in the paper is very difficult to organize. Thus, we only provided a (relative large) jupyter-notebook for now, along with the raw data from our evaluation. Almost all plotting python codes are in that notebook. You can explore those data and jupyter-notebook. Link:

notebook: https://cloud.tsinghua.edu.cn/f/a4bbd1737a8f4191b069/ or https://drive.google.com/file/d/1qHKpdIonP34ihB8MNzQXO7KiEwKdyIxP/view?usp=sharing

data: https://cloud.tsinghua.edu.cn/f/3ec07ad9c14c4548b3c6/ or https://drive.google.com/file/d/1IY1DOrwFJzUFnogameSD5g8QRyjjbgkI/view?usp=sharing
