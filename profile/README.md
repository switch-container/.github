# TrEnv: Transparently Share Serverless Execution Environments Across Different Functions and Nodes

This organization contains the source codes of our paper @ SOSP: TrEnv: Transparently Share Serverless Execution Environments Across Different Functions and Nodes. In the following, I will give instructions on how to depoly TrEnv.

It contains a modified Linux kernel, the modified container runtime (runc, containerd and faasd), the modified CRIU and test scripts.

For the details of TrEnv, please refer to our paper.

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

We recommend the local DRAM capacity is >= 128 GB. There is no constraint of the ratio between local DRAM and CXL memory. As long as the CXL memory can store the memory snapshots (16 GB is enough for our entire evaluated functions).

Finally, TrEnv supports RDMA through RXE (soft roce), so there is no need for hardware RDMA NIC.

*Note that the RDMA support is based on Fastswap @ EuroSys'20, with some modification to accommodate with mm-template. It is easy to adapt TrEnv for the hardware RDMA.*

### Our Hardware Setup

The hardware used for the evaluation in paper is:

- A two-socket 32-core Intel XeonGold 6454S CPU, enable the hyper-threading.

- 256 GB Local DRAM.

- An 128 GB early-stage Samsung CXL device. Its access latency is 600 ns, which is far slower than current CXL device (around 300-400 ns), and even slower than PMem.

- A local-to-local RXE, whose 4 KB latency is around 6 us, which is comparable with hardware RDMA environment.

## Software

In the following sections, I will give you instructions on how to install and run TrEnv.

The instructions I give is **based on a Ubuntu 22.04 as root user**. We also tested it on the CentOS-based environment (but the actual dependency to install has some differences with Ubuntu). If you have any questions on installation on CentOS-based environment, you can contact with me to see if I can help.

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
cd criu
apt install libprotobuf-dev libprotobuf-c-dev protobuf-c-compiler protobuf-compiler python3-protobuf iproute2 libcap-dev libnl-3-dev libnet-dev
make -j16 install-criu
# the output binary is located at /root/criu/criu/criu and /usr/local/sbin/criu
```

Note that CentOS-based system may lack some of the packages. The dependencies listed here is the same as CRIU, so please refer to homepage of CRIU to check the instructions on how to install CRIU on your systems.

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
cd /root
git clone https://github.com/switch-container/utils.git
git clone --branch huang https://github.com/switch-container/faasd-testdriver.git
mkdir -p /root/test && cd /root/test
ln -s /root/faasd-testdriver/ /root/test/faasd-testdriver && \
 ln -s /root/rdma-server/pseudo-mm-rdma-server /root/test/pseudo-mm-rdma-server && \
 ln -s /root/utils/stack.yml /root/test/stack.yml && \
 ln -s /root/faasd-testdriver/functions/template/ /root/test/template

# download two dataset (Azure and Huawei traces)
mkdir -p /root/downloads && cd /root/downloads && \
  wget https://azurepublicdatasettraces.blob.core.windows.net/azurepublicdatasetv2/azurefunctions_dataset2019/azurefunctions-dataset2019.tar.xz && \
  wget https://sir-dataset.obs.cn-east-3.myhuaweicloud.com/datasets/public_dataset/public_dataset.zip
# unzip the dataset
mkdir azurefunction-dataset2019 && tar xf azurefunctions-dataset2019.tar.xz -C azurefunction-dataset2019 && unzip public_dataset.zip
```

The third is the python environment, we recommend to use python version 3.10 (default version for Ubuntu 22.04).

```bash
apt install python3.10-venv
cd /root && mkdir venv && cd venv && python3 -m venv faasd-test
source /root/venv/faasd-test/bin/activate && pip install pyyaml gevent requests pandas numpy matplotlib
```

The last is the function-specific dependencies. In TrEnv, we will dynamically mount the function-specific dependencies into the mntns at runtime. To achieve that, we first need to extract and put each function's dependencies on hosts and mount them as overlayfs during running.
