## Setup

The following contains the XDP development environment setup information for an Ubuntu machine.

#### Native XDP support
Todo Punith

#### Prerequisites
First, certain packages need to be installed.
```markdown
sudo apt-get install -y make gcc libssl-dev bc libelf-dev libcap-dev clang gcc-multilib llvm libncurses5-dev git pkg-config libmnl-dev bison flex graphviz git exuberant-ctags binutils iproute2 
```

#### Compiling an XDP enabled kernel
Development of BPF and XDP features happens in the `net-next` tree. Obtain the kernel source.
```markdown
git clone git://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git
```

To create a configuration file for compilation, copy the existing config file.
```markdown
cd net-next
cp /boot/config-`uname -r`* .config
```

Make sure that the following options in .config have the right values set.
```markdown
CONFIG_CGROUP_BPF=y
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_NET_SCH_INGRESS=m
CONFIG_NET_CLS_BPF=m
CONFIG_NET_CLS_ACT=y
CONFIG_BPF_JIT=y
CONFIG_LWTUNNEL_BPF=y
CONFIG_HAVE_EBPF_JIT=y
CONFIG_BPF_EVENTS=y
CONFIG_TEST_BPF=m
```

Build the configuration.
```markdown
make menuconfig
```

Compile the kernel. If you have multiple CPUs, use the j option. For example, for 4 CPUs do. 
```markdown
make -j4
```

When the make completes, compile modules
```markdown
sudo make modules_install install
```

I got an error `Error! Bad return status for module build on kernel: 5.0.0-rc5+ (x86_64)` at this stage. But, i went ahead and completed the rest of the steps. This did not cuse any problems for XDP development later.

Reboot and select the newly built kernel at GRUB. 
```markdown
sudo reboot
```

Verify the setup
```markdown
cd tools/testing/selftests/bpf/
make
./test_verifier
```

No tests should fail. Now your XDP development environment is ready!
