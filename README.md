# ELISA: Exit-Less, Isolated, and Shared Access

This document complements the paper "[Exit-Less, Isolated, and Shared Access for Virtual Machines](https://doi.org/10.1145/3582016.3582042)" presented at the International Conference on Architectural Support for Programming Languages and Operating Systems in March 2023 (ASPLOS'23).

## WARNING

- **The authors do not guarantee perfect security of the ELISA-relevant prototype implementations.**
- **The ELISA-relevant prototype implementations can break your entire system.**
- **The authors will not bear any responsibility if the implementations, provided by the authors, cause any problems.**

## Table of contents

- [Materials](#materials)
- [List of the ELISA prototype repositories](#list-of-the-elisa-prototype-repositories)
- [Requirements](#requirements)
- [Setup](#setup)
- [Trying an example application : elisa-app-nop](#trying-an-example-application--elisa-app-nop)
- [Commentary](#commentary)
	- [Section 4.1 : Anywhere Page Table (APT)](#section-41--anywhere-page-table-apt)
	- [Section 4.2 : Gate EPT Context](#section-42--gate-ept-context)
	- [Section 5.1 : ELISA-Specific Hypercalls](#section-51--elisa-specific-hypercalls)
	- [Section 5.2 : libelisa](#section-52--libelisa)
	- [Section 5.3 : Negotiation Steps](#section-53--negotiation-steps)
	- [Section 5.4 : Code for the Sub EPT Context](#section-54--code-for-the-sub-ept-context)
	- [Section 5.5 : Shared Memory Management](#section-55--shared-memory-management)
	- [Section 5.6 : Interrupt Setting](#section-56--interrupt-setting)
	- [Section 6.1 : Context Switch Overhead](#section-61--context-switch-overhead)
	- [Section 7.1 : VM Networking System](#section-71--vm-networking-system)

## Materials

- Paper: [https://doi.org/10.1145/3582016.3582042](https://doi.org/10.1145/3582016.3582042)
- Slides
- Lightning talk slides
- Lightning talk video: [https://www.youtube.com/watch?v=oLatL6TIIa4](https://www.youtube.com/watch?v=oLatL6TIIa4)
- Poster

## List of the ELISA prototype repositories

### Core components

- [libelisa](https://github.com/yasukata/libelisa): the core library for ELISA
- [libelisa-extra](https://github.com/yasukata/libelisa-extra): supplemental utilities of libelisa
- [kvm-elisa](https://github.com/yasukata/kvm-elisa): KVM modification for ELISA

### Utilities

- [elisa-util-exec](https://github.com/yasukata/elisa-util-exec): a simple ELISA application launcher

### Applications

- [elisa-app-nop](https://github.com/yasukata/elisa-app-nop): a minimal ELISA application
- [elisa-app-vmnet](https://github.com/yasukata/elisa-app-vmnet): an ELISA-based VM networking system
	- [rvs](https://github.com/yasukata/rvs): a virtual switch implementation employed for the ELISA-based VM networking system
	- [librte_pmd_rvif](https://github.com/yasukata/librte_pmd_rvif): a DPDK poll mode driver for rvif that is the vNIC of rvs

## Requirements

The ELISA prototype implementations assume the following platform.

- CPU: VMFUNC-capable Intel CPU
- OS: Linux
- Hypervisor: QEMU

The authors use Ubuntu 20.04/22.04, and recommend using Ubuntu for a relatively easy setup.

If you do not have QEMU on your machine, you can install it by the following command (on Ubuntu).

```
sudo apt install qemu-system-x86
```

## Setup

**As mentioned in [the WARNING section](#warning), the ELISA prototype implementations can break your entire system. Therefore, we strongly recommend trying the ELISA prototype implementations on a computer that is OK to get damaged.**

### General information

To use ELISA, we need to install a modified KVM kernel module.

Please find the ELISA patch for KVM [here](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch); this patch is generated for linux-source-5.15.0 which is distributed by the apt repository of Ubuntu.

We are primarily assuming this patch for KVM is applied **by hand** along with manual source code changes for a Linux version installed in the user's environment; this is because:
- the definitions of data structures and functions, used in the original KVM implementation, are changed quite often, and minor version changes lead to incompatibility between the KVM module and the Linux kernel, therefore, we thought it is difficult for a single repository to cover a large set of Linux versions.
- we do not want developers, who think of trying ELISA, to compile the full Linux kernel code base just for ELISA because kernel compilation is troublesome; we assume to use a Linux kernel provided by a distributor (e.g., Ubuntu apt repository).

Patching by hand sounds complicated, but we believe it is not too much because we have minimized the size of the patch: the code that needs to be added is just around 120 lines of a single code block, and a single line has to be replaced.

The goal of the patch is to add the hypercalls, newly implemented for ELISA.

- [line 137 ~ 138](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L137-L138) applies the newly implemented hypercall handler as the callback function for VMCALL.
- [line 8 ~ 128](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L8-L128) is the hypercall implementation.

### Example on Ubuntu 22.04 with Linux 5.15

While ELISA does not depend on a specific Linux kernel version, we do not want to compile the full Linux kernel source code because it is troublesome.

This example shows how to use ELISA on Ubuntu 22.04 without fully replacing the Linux kernel provided by Ubuntu. 

#### 1. download the Linux source code

First, please download the Linux kernel source code by the following command; you will get it at ```/usr/src/linux-source-5.15.0.tar.bz2```.

```
sudo apt install linux-source-5.15.0
```

#### 2. install the Linux kernel for the downloaded source

To install a self-compiled KVM module, the source code of it has to be compatible with the Linux kernel running on the machine, and the versions of the Linux source code and the running Linux kernel have to be the same or very close; otherwise, a compilation of the KVM source code often fails because of undefined symbols.

To cope with this issue, let's first check which version of the Linux source is installed in the previous step [1. download the Linux source code](1-download-the-linux-source-code).

```
$ sudo apt install linux-source-5.15.0
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
linux-source-5.15.0 is already the newest version (5.15.0-67.74).
```

As seen above, in this case, we have the Linux source for 5.15.0-67. (**in the following steps, please change the version number according to the linux-source version you got on your environment.**)

To minimize the version mismatch between the source code and the installed Linux kernel, we install Linux 5.15.0-67 (offered by Ubuntu) by the following command. After finishing the following procedures, you need to reboot (and you may need to explicitly specify it in the GRUB menu) to load Linux 5.15.0-67.

```
KVER=5.15.0-67-generic sudo apt install linux-image-$KVER linux-modules-$KVER linux-modules-extra-$KVER linux-headers-$KVER
```

After reboot, you can confirm Linux 5.15.0-67 is properly loaded by the ```uname``` command.

```
$ uname -a
Linux ubuntu 5.15.0-67-generic #74-Ubuntu SMP Wed Feb 22 14:14:39 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

#### 3. extract the KVM module files

Then, let's make a new working directory; this time, we name it ```./kvm-elisa-patch```.

```
mkdir kvm-elisa-patch
```

Please extract the downloaded Linux source in ```./kvm-elisa-patch``` by the following command.

```
tar xvf /usr/src/linux-source-5.15.0.tar.bz2 -C ./kvm-elisa-patch
```

For easy source code maintenance, let's copy KVM-relevant files to ```./kvm-elisa-patch/kvm```. (It is also fine to directly use the source in ```./kvm-elisa-patch/linux-source-5.15.0```)

```
mkdir -p kvm-elisa-patch/kvm/virt
```

```
mkdir -p kvm-elisa-patch/kvm/arch/x86
```

```
cp -r kvm-elisa-patch/linux-source-5.15.0/virt/kvm kvm-elisa-patch/kvm/virt
```

```
cp -r kvm-elisa-patch/linux-source-5.15.0/arch/x86/kvm kvm-elisa-patch/kvm/arch/x86
```

Now, we have the default KVM implementation in ```./kvm-elisa-patch/kvm```.

It may be a good time to save the status of ```./kvm-elisa-patch/kvm``` by git so that we can easily roll back to the default KVM implementation. If you have no preference for gitting, you can use the following commands.

```
cd kvm-elisa-patch/kvm
```

```
git init
```

```
git add .
```

```
git commit -m "first commit"
```

#### 4. test compilation for the default KVM module

Before starting to apply the ELISA patch, let's confirm the KVM source code can be properly compiled.

Please save the following as ```./kvm-elisa-patch/kvm/arch/x86/kvm/MyMakefile```.

```Makefile
KDIR := /lib/modules/$(shell uname -r)/build

srctree := $(dir $(abspath $(lastword $(MAKEFILE_LIST))))../../..

ccflags-y += -I$(srctree)/arch/x86 # for trace-relevant header

include $(dir $(abspath $(lastword $(MAKEFILE_LIST))))Makefile

EXTRA_CFLAGS += $(ccflags-y)

SUBDIRS := $(dir $(abspath $(lastword $(MAKEFILE_LIST))))
COMMON_OPS = -C $(KDIR) M='$(SUBDIRS)' EXTRA_CFLAGS='$(EXTRA_CFLAGS)'

CLEANITEMS := *.ur-safe .cache.mk *.o *.ko *.cmd *.mod.c *.mod .*.cmd .tmp_versions modules.order Module.symvers *~
CLEANOBJS = $(addprefix $(dir $(abspath $(lastword $(MAKEFILE_LIST)))),$(CLEANITEMS)) \
			$(addprefix $(dir $(abspath $(lastword $(MAKEFILE_LIST))))mmu/,$(CLEANITEMS)) \
			$(addprefix $(dir $(abspath $(lastword $(MAKEFILE_LIST))))vmx/,$(CLEANITEMS)) \
			$(addprefix $(dir $(abspath $(lastword $(MAKEFILE_LIST))))svm/,$(CLEANITEMS)) \
			$(addprefix $(dir $(abspath $(lastword $(MAKEFILE_LIST))))../../../virt/kvm/,$(CLEANITEMS)) \

default:
	$(MAKE) $(COMMON_OPS) modules

clean:
	@rm -f $(CLEANOBJS)
```

Then, please enter ```./kvm-elisa-patch/kvm/arch/x86/kvm```.

```
cd kvm-elisa-patch/kvm/arch/x86/kvm 
```

Please type the following command to compile the KVM module.

```
make -f MyMakefile
```

If the compilation fails here and the compiler complains, for example, something is not declared, it would be because of the version mismatch between the source code and the Linux kernel you are currently using; in this case, please look back [2. install the Linux kernel for the downloaded source](#2-install-the-linux-kernel-for-the-downloaded-source).

#### 5. install the KVM module for testing

Once you could compile the KVM module, you can install it by the following commands.

First, uninstall the current kvm_intel (vmx) kernel module.

```
sudo rmmod kvm_intel
```

Afterward, uninstall the current kvm kernel module.

```
sudo rmmod kvm
```

Then, install the newly compiled kvm kernel module.

```
sudo insmod ./kvm.ko
```

Finally, install the modified kvm_intel (vmx) kernel module.

```
sudo insmod ./kvm-intel.ko
```

Up to here, if you do not encounter any issue, it means that the installation of the default KVM module works properly, and if you find a problem after proceeding with the subsequent steps, it means that the ELISA patch adaptation causes the problem.

#### 6. apply the ELISA patch for KVM

Finally, let's start applying the patch for ELISA to KVM.

Here, we assume that we are in ```./kvm-elisa-patch```.

Please download the ELISA patch for KVM.

```
wget https://raw.githubusercontent.com/yasukata/kvm-elisa/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch
```

First, please try to apply it by the following git command; if you are lucky (maybe using Linux 5.15), the command above successfully applies the ELISA patch to the KVM module implementation, otherwise, you need to modify the KVM source code manually.

```
git -C kvm apply ../elisa.patch
```

If the patch adaptation by the command above fails, please edit ```kvm-elisa-patch/kvm/arch/x86/kvm/vmx/vmx.c``` by yourself to:

- set [```handle_vmx_hypercall```](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L113) to the VMCALL handler (done in [line 137 ~ 138 in the patch](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L137-L138)).
- add the code block which is [line 8 ~ 128 in the patch](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L8-L128); it may be better to put it just above the part setting ```handle_vmx_hypercall``` to the VMCALL handler.

Afterward, please try to compile the KVM module that includes the ELISA patch by the following commands.

```
cd kvm/arch/x86/kvm 
```

```
make -f MyMakefile
```

If the compilation by the command above fails, it would be because the definition of the data structures and functions, used in the ELISA patch, would be different from the ones defined in the Linux kernel version that you use.

In this case, please change the definitions in the ELISA patch accordingly by your hand; hopefully, it is not too complicated because the size of the ELISA patch is not too big.

#### 7. install the modified KVM module

After you could compile the modified KVM module, you can install it using the same commands shown in [5. install the KVM module for testing](#5-install-the-kvm-module-for-testing).

