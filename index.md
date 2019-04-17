## XDP
Express Data Path is a programmable fast packet processor in the kernel. Details about XDP can be found [here](https://dl.acm.org/citation.cfm?id=3281443), and [here](https://developers.redhat.com/blog/2018/12/06/achieving-high-performance-low-latency-networking-with-xdp-part-1/). This article contains the steps to setup a development environment for XDP.  

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

## First XDP Program
The _hello world_ equivalent of an XDP program, is a program that just drops all packets. 

We first start with writing a Makefile to build the code we write. This Makefile just makes sure required include files are accessible. 

```markdown
KDIR ?= /lib/modules/$(shell uname -r)/source
CLANG ?= clang
LLC ?= llc
ARCH := $(subst x86_64,x86,$(shell arch))

BIN := drop_packets.o
CLANG_FLAGS = -I. -I$(KDIR)/arch/$(ARCH)/include \
    -I$(KDIR)/arch/$(ARCH)/include/generated \
    -I$(KDIR)/include \
    -I$(KDIR)/arch/$(ARCH)/include/uapi \
    -I$(KDIR)/arch/$(ARCH)/include/generated/uapi \
    -I$(KDIR)/include/uapi \
    -I$(KDIR)/include/generated/uapi \
    -include $(KDIR)/include/linux/kconfig.h \
    -I$(KDIR)/tools/testing/selftests/bpf/ \
    -D__KERNEL__ -D__BPF_TRACING__ -Wno-unused-value -Wno-pointer-sign \
    -D__TARGET_ARCH_$(ARCH) -Wno-compare-distinct-pointer-types \
    -Wno-gnu-variable-sized-type-not-at-end \
    -Wno-address-of-packed-member -Wno-tautological-compare \
    -Wno-unknown-warning-option \
    -O2 -emit-llvm

all: $(BIN)

drop_packets.o: drop_packets.c
    $(CLANG) $(CLANG_FLAGS) -c $< -o - | \
    $(LLC) -march=bpf -mcpu=$(CPU) -filetype=obj -o $@

clean:
    rm -f drop_packets.o
```

The XDP program that drops all packets is as follows _drop_pcakets.c_
```markdown
#define KBUILD_MODNAME "foo"
#define asm_volatile_goto(x...)
#include <uapi/linux/bpf.h>

SEC("prog")
int xdp_drop(struct xdp_md *ctx)
{
    return XDP_DROP;
}

char __license[] __section("license") = "GPL";
```

The first two _#define_ are to be defined and are workarounds for clang not being able to work with _asm_goto_ constructs. __These two definitions must be done in all XDP programs__. For this simple program, the section name in `SEC(" ")` has to be `prog` because clang looks for this section. In the later part of this page, user programs that compile and load kernel programs will be written in which case the section name need not be `prog`. the `xdp_drop` function, recieves the packet in `ctx` at the interface we load the XDP program to. The functions returns the value `XDP_DROP` forcing all packets at the interface to be dropped. The _license_ section is also necessary to access _GPL_ licensed BPF features. 

Build the program
```
make
```

Load the program to an interface
```
sudo ip link set dev <interface-name> xdp object drop_packets.o verb
```

Send some traffic to the interface from another node. You can use any other traffic generator to send traffic to the interface. Listen at interface node here XDP program was loaded. You should not be able to recieve any packets. 

Unload the program from the interface
```
sudo ip link set dev <interface-name> xdp off
```

## TODO 
### Using XDP maps
### Modifying packets using XDP
### Using XDP tail calls 
