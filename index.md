Written May 2019
## XDP
Express Data Path is a programmable fast packet processor in the kernel. Details about XDP can be found [here](https://dl.acm.org/citation.cfm?id=3281443), and [here](https://developers.redhat.com/blog/2018/12/06/achieving-high-performance-low-latency-networking-with-xdp-part-1/). This article contains the steps to setup a development environment for XDP.  

## Other Articles
### [Using XDP Maps](https://priyankaselvan.github.io/eXpress-Data-Path--Maps)
### [Using XDP tail calls](https://priyankaselvan.github.io/eXpress-Data-Path--TailCalls)
### [Modifying packets using xdp](https://priyaselvan.github.io/eXpress-Data-Path--Packet-Inspection/)
### [Cloudlab Setup for XDP experimentation](https://punithpatil.github.io/eXpress-Data-Path--Cloudlab-Setup/)

## Hardware - Native XDP support

An approximate list of drivers supporting XDP programs was found in the [iovisor/bcc](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#xdp) project. The machine used contained Dual-port Mellanox ConnectX-4 25 GB NIC (PCIe v3.0, 8 lanes). This matches the supporting driver Mellanox `mlx4` driver.


To verify that the correct NIC exists, PCIe connected hardware can be checked. Our setup provides 2 Mellanox NICs with 2 interfaces each. They will show up as 4 Ethernet controllers. Other supporting drivers should work too.

```console
variable@xdp-node:~$ lspci -v | grep -A 5 Mellanox
03:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
	Subsystem: Hewlett Packard Enterprise MT27710 Family [ConnectX-4 Lx]
	Physical Slot: 1
	Flags: bus master, fast devsel, latency 0, IRQ 16, NUMA node 0
	Memory at 94000000 (64-bit, prefetchable) [size=32M]
	Capabilities: <access denied>
--
03:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
	Subsystem: Hewlett Packard Enterprise MT27710 Family [ConnectX-4 Lx]
	Physical Slot: 1
	Flags: bus master, fast devsel, latency 0, IRQ 17, NUMA node 0
	Memory at 96000000 (64-bit, prefetchable) [size=32M]
	Capabilities: <access denied>
--
07:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
	Subsystem: Hewlett Packard Enterprise MT27710 Family [ConnectX-4 Lx]
	Flags: bus master, fast devsel, latency 0, IRQ 16, NUMA node 0
	Memory at 98000000 (64-bit, prefetchable) [size=32M]
	Capabilities: <access denied>
	Kernel driver in use: mlx5_core
--
07:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
	Subsystem: Hewlett Packard Enterprise MT27710 Family [ConnectX-4 Lx]
	Flags: bus master, fast devsel, latency 0, IRQ 17, NUMA node 0
	Memory at 9a000000 (64-bit, prefetchable) [size=32M]
	Capabilities: <access denied>
	Kernel driver in use: mlx5_core
```

## Setup

The following contains the XDP development environment setup information for an Ubuntu 18.04 LTS machine.

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

Make sure that the following options in _.config_ have the right values set.
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

I got an error `Error! Bad return status for module build on kernel: 5.0.0-rc5+ (x86_64)` at this stage. But, i went ahead and completed the rest of the steps. This did not cause any problems for XDP development later.

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

We first start with writing a Makefile to build the kernel code we write.  Makefile just makes sure required header files are accessible. The Makefile compiles the code we write using _clang_. This is due to the fact that only _clang_ provides an option of specifying a _bpf_ target required for XDP. 

```markdown
KDIR ?= /lib/modules/$(shell uname -r)/source
CLANG ?= clangThis
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

The XDP program that drops all packets is as follows _drop_packets.c_
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

The first two _#define_ are to be defined and are workarounds for clang not being able to work with _asm_goto_ constructs. __These two definitions must be done in all XDP kernel programs__. For this simple program, the section name in `SEC(" ")` has to be `prog` because _iproute2_ - the tool we use to load the program to the interface looks for this section. In other articles linked to this one, user programs that compile and load kernel programs will be written in which case the section name need not be `prog`. The `xdp_drop` function, recieves the packet in `ctx` at the interface we load the XDP program to. The functions returns the value `XDP_DROP` forcing all packets at the interface to be dropped. The _license_ section is also necessary to access _GPL_ licensed BPF features. 

Build the program
```
make
```

Load the program to an interface
```
sudo ip link set dev <interface-name> xdp object drop_packets.o verb
```

Send some traffic to the interface from another node. You can use any other traffic generator to send traffic to the interface. Listen at interface node here XDP program was loaded. You should not be able to receive any packets. 

Unload the program from the interface
```
sudo ip link set dev <interface-name> xdp off
```

## Debugging XDP programs
In order to print debugs from the kernel XDP program, _printk_ must be used. A standard way of using this in most XDP sample programs is defining a macro _bpf_debug_ as follows. 
```
#define bpf_debug(fmt, ...)                     \
        ({                          \
            char ____fmt[] = fmt;               \
            bpf_trace_printk(____fmt, sizeof(____fmt),  \
                     ##__VA_ARGS__);            \
        })
```

This can be used like a formatted output statement anywhere in the kernel XDP program.
```
bpf_debug("XDP: Killroy was here! %d\n", 42);
```

In order to view these debugs as traffic arrives at the interface, the following steps need to be taken. 
First, set the debug level to show all debugs. _7_ means all levels. 
```
sudo sh -c 'echo 7 > /proc/sys/kernel/printk'
```
Viewing the debugs. Needs to be run as root. Run `sudo su` before the following. 
```
tc exec bpf dbg
```

Setup and the first program is done. To experiment with other XDP features, you may take a look at the articles listed below. 

### [Using XDP Maps](https://priyankaselvan.github.io/eXpress-Data-Path--Maps)
### [Using XDP tail calls](https://priyankaselvan.github.io/eXpress-Data-Path--TailCalls)
### [Modifying packets using xdp](https://priyaselvan.github.io/eXpress-Data-Path--Packet-Inspection/)
### [Cloudlab Setup for XDP experimentation](https://punithpatil.github.io/eXpress-Data-Path--Cloudlab-Setup/)


#### References
# [A walkthrough](https://netdevconf.org/2.1/slides/apr7/gospodarek-Netdev2.1-XDP-for-the-Rest-of-Us_Final.pdf)
# [Cilium XDP documentation](https://cilium.readthedocs.io/en/latest/bpf/#development-environment)
# [XDP maps example](https://www.linuxplumbersconf.org/event/2/contributions/71/attachments/17/9/presentation-lpc2018-xdp-tutorial.pdf)


