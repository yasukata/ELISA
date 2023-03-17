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

## Trying an example application : elisa-app-nop

The [elisa-app-nop](https://github.com/yasukata/elisa-app-nop) application would be a good starting point.

**As mentioned in [the WARNING section](#warning), the ELISA prototype implementations can break your entire system. Therefore, we strongly recommend trying the ELISA prototype implementations on a computer that is OK to get damaged.**

### Preparation for running an ELISA-based application

The example application runs on VMs that are hosted by the host installing [the KVM module including the ELISA patch](#setup).

Please prepare at least one VM image that QEMU can run, in a manner you wish (e.g., virt-manager); we also recommend using Ubuntu for the VM.

### Download the source code needed for elisa-app-nop

Please type the following to download the source files; it is recommended to download and build the them on a VM which executes them.

```
git clone https://github.com/yasukata/elisa-app-nop.git
```

Then, please enter ```./elisa-app-nop```.

```
cd ./elisa-app-nop
```

First, please create directories in ```./elisa-app-nop``` to place the core library and utility for ELISA.

```
mkdir -p deps/elisa/lib
```

```
mkdir -p deps/elisa/util
```

Please type the following commands to download the source code of libelisa, libelisa-extra, and elisa-util-exec.

```
git -C deps/elisa/lib clone https://github.com/yasukata/libelisa.git
```

```
git -C deps/elisa/lib clone https://github.com/yasukata/libelisa-extra.git
```

```
git -C deps/elisa/util clone https://github.com/yasukata/elisa-util-exec.git
```

Now, we have all assets needed to build the elisa-app-nop application.

### Compilation of elisa-app-nop

Let's start compilation.

The following produces ```./deps/elisa/lib/libelisa/libelisa.a```.

```
make -C deps/elisa/lib/libelisa
```

The following generates ```./deps/elisa/util/elisa-util-exec/a.out```.

```
make -C deps/elisa/util/elisa-util-exec
```

The following command generates ```./lib/libelisa-applib-nop/lib.so```.

```
make -C lib/libelisa-applib-nop
```

The following command generates ```./libelisa-app-nop.so```.

```
make
```

### Run elisa-app-nop

#### Note on the simplified example for elisa-app-nop

- For the following example commands, to simplify the testing, we assume to use a single VM as the manager VM and the guest VM, meaning that the program made for the manager VM and the program for guest VMs run on the same VM.
- If you have multiple VMs and wish to use one as the manager VM and the others as the guest VMs, please change the IP address passed to ```./deps/elisa/util/elisa-util-exec/a.out``` executed on a guest VM through the ```-s``` option, to specify the IP address of the manager VM. No change is needed for the command to be executed on the manager VM.

#### The command for the manager VM

The following command is for the manager VM. **WARNING: Do not terminate the process executing the following command while an ELISA-based application runs on a guest VM. The memory for the gate/sub EPT contexts of a guest VM is allocated from the process running the command on the manager VM; if you stop it, the memory for the gate/sub EPT contexts of the guest VM is automatically released and it results in a system failure because the guest VM accesses the released memory.**

```
sudo ELISA_APPLIB_FILE=./lib/libelisa-applib-nop/lib.so ./deps/elisa/util/elisa-util-exec/a.out -f ./libelisa-app-nop.so -p 10000
```

#### The command for a guest VM

The following command is for a guest VM.

```
taskset -c 0 ./deps/elisa/util/elisa-util-exec/a.out -f ./libelisa-app-nop.so -p 10000 -s 127.0.0.1
```

#### Example output of elisa-app-nop

The following is the example output seen on the guest VM.

```
$ taskset -c 0 ./deps/elisa/util/elisa-util-exec/a.out -f ./libelisa-app-nop.so -p 10000 -s 127.0.0.1
enter sub EPT context and get a return value
return value is 1
```

### Explanation on elisa-app-nop

Up to here, we have quickly run the elisa-app-nop application.

Here, we explain what happened in the steps above.

#### Binary files

We have generated four binary files.

##### ./deps/elisa/lib/libelisa/libelisa.a

```./deps/elisa/lib/libelisa/libelisa.a``` is involved by ```./deps/elisa/util/elisa-util-exec/a.out``` and ```./libelisa-app-nop.so```.

##### ./lib/libelisa-applib-nop/lib.so

```./lib/libelisa-applib-nop/lib.so``` contains the code executed in the sub EPT context.

##### ./libelisa-app-nop.so

```./libelisa-app-nop.so``` implements the application-specific logic which is nop this time, and it includes the procedure for both the manager VM and guest VMs:
- in the manager VM side, it [loads the code](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L42-L59) specified by an environment variable [```ELISA_APPLIB_FILE```](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L29) to the sub EPT context of a guest VM, and
- in the guest VM side, it [enters the sub EPT context and executes the code in it](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L69-L75).

##### ./deps/elisa/util/elisa-util-exec/a.out

```./deps/elisa/util/elisa-util-exec/a.out``` is a generic launcher application and used for executing both the commands for the manager VM and guest VMs; we note that we made ```./deps/elisa/util/elisa-util-exec/a.out``` just for reducing redundant implementations and it is not mandatory to use. 

```./deps/elisa/util/elisa-util-exec/a.out``` works either in the manager VM mode or the guest VM mode; if it has an argument for ```-s``` which specifies the IP address of the manager VM, it will be the guest VM mode, and otherwise, the manager VM mode.

```./deps/elisa/util/elisa-util-exec/a.out``` loads a library file specified through its ```-f``` option passes a pointer to a function named ```elisa__server_cb``` to libelisa if it is in the manager VM mode, and ```elisa__client_cb``` will be passed to libelisa if it is in the guest VM mode. The option ```-p``` specifies the port to listen on for the manager VM or connect to for a guest VM.

#### Explanation on the commands

The meaning of the command for the manager VM is:
- ```sudo```: we need sudo for translating from GVA to GPA
- ```ELISA_APPLIB_FILE=./lib/libelisa-applib-nop/lib.so```: requesting ```./libelisa-app-nop.so``` to load ```./lib/libelisa-applib-nop/lib.so``` to the sub EPT context of a guest VM.
- ```./deps/elisa/util/elisa-util-exec/a.out```: the binary executed
- ```-f ./libelisa-app-nop.so```: requesting ```./deps/elisa/util/elisa-util-exec/a.out``` to load ```./libelisa-app-nop.so```
- ```-p 10000```: requesting ```./deps/elisa/util/elisa-util-exec/a.out``` to listen on port 10000

The meaning of the command for the guest VM is:
- ```taskset -c 0```: we specify the CPU affinity of the process to ensure that the negotiation with the manager VM and the execution of the code in the sub EPT context are done on the same vCPU. (The negotiation (explained in Section 5.3 of the paper) with the manager VM has to be done for each vCPU, but this implementation, particularly on the guest VM side, only performs a single negotiation with the manager VM. By modifying the implementation for the guest VM to conduct negotiation for each vCPU, we can remove this restriction.)
- ```./deps/elisa/util/elisa-util-exec/a.out```: the binary executed
- ```-f ./libelisa-app-nop.so```: requesting ```./deps/elisa/util/elisa-util-exec/a.out``` to load ```./libelisa-app-nop.so```
- ```-p 10000```: requesting ```./deps/elisa/util/elisa-util-exec/a.out``` to connect to port 10000 of the manager VM
- ```-s 127.0.0.1```: telling ```./deps/elisa/util/elisa-util-exec/a.out``` that the IP address of the manager VM is 127.0.0.1. (If you use different VMs for the manager VM and the guest VM, please change this part to specify the IP address of the manager VM.)

#### Explanation of the implementations

The output by printf on the guest VM comes from [line 70](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L70) and [line 74](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L74).

The guest VM prints [the value obtained from the code executed in the sub EPT context](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L72).

The implementation of the code for the sub EPT context is found in [elisa-app-nop/lib/libelisa-applib-nop/main.c](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/lib/libelisa-applib-nop/main.c); especially, the current implementation of it [returns 1](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/lib/libelisa-applib-nop/main.c#L26), therefore, we have got the output ```return value is 1```.

In ```elisa-app-nop/lib/libelisa-applib-nop/main.c```, [line 71](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L71) disables interrupt and [line 73](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L73) enables interrupt; to find the reason why we need this, please refer to Section 5.6 of the paper.

#### Quick experiment

For a quick experiment, if you replace the return value in ```elisa-app-nop/lib/libelisa-applib-nop/main.c``` with a value that you like and recompile it, the next execution will print the value that you put.

## Commentary

This commentary bridges the descriptions in the paper and the source code of the ELISA prototype.

### List of the sections complemented by the commentary

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

### Section 4.1 : Anywhere Page Table (APT)

The implementation of APT is found at [```configure_apt```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L304-L348), which assumes to be called at [the last step](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L383) of [an EPT context setup](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L356-L384); it fills all empty GPA entries in the EPT with the pointer to the HPA of the created page table root.

The page table root for APT is allocated as [```uint64_t apt_mem[512]```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L299) which is part of [```struct vcpu_ept_ctx```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L298-L302) that represents an EPT context; we allocate ```struct vcpu_ept_ctx``` [on the stack](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L427-L439), and [```__attribute__((aligned(0x1000)))```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L302) automatically instantiates ```uint64_t apt_mem[512]``` at a 4KB aligned memory address.

```configure_apt``` first [creates the mapping between the last GPA in the GPA range and HPA of ```uint64_t apt_mem[512]```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L328-L330); at the same time, and this will add an entry to each level of the EPT tree.

Then, we [collect the added entry value at each layer of the EPT tree](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L336-L344), in a 64-bit integer array.

Afterward, we walk through the EPT tree, and if there is an empty entry at a specific level, we [fill this entry with the collected entry value of the same level](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L315).

### Section 4.2 : Gate EPT Context

The implementation of the EPT context transition point (shown by Figure 5 in the paper) is divided across the default, gate, and sub EPT contexts.  

#### The default EPT context

A guest VM only needs to load a [12 KB code block](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L22-L53) (4 KB x 3 : top, middle, and bottom) in its program to make an entry point.

The [top 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L23-L38) has the entry point to the gate EPT context, named ```elisa_gate_entry```, at the end of it.

The [middle 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L42-L43) is empty. (The shaded part in Figure 5.)

When the execution returns from the sub EPT context, it lands at the top of the [bottom 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L47-L53).

#### The gate EPT context

The guest VM enters the gate EPT context by executing [VMFUNC at the end of the top 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L37), and lands at the top of the [middle 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L48-L92) in the gate EPT context; the code shown in Figure 6 is [in this middle 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L48-L60).

The manager VM maps the EPTP list for the vCPU at the [bottom 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L511-L518).

The manager VM conducts the page table and EPT settings for the gate EPT context [here](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L507-L532).

#### The sub EPT context

The middle 4 KB page of the gate EPT context contains VMFUNC and it brings the execution at [line 116](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L116) in the [middle 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L116-L147) of the sub EPT context.

The manager VM maps the EPTP list for the vCPU at the [bottom 4 KB page](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L471).

### Section 5.1 : ELISA-Specific Hypercalls

The ELISA-specific hypercalls are implemented in [the patch for KVM](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L113-L128).

As mentioned in [general information](#general-information) for the setup, we tried to minimize the size of the patch for KVM as much as possible; for simplicity, in the current implementation, there is no clear separation between the manager and guest hypercalls, and all guest VMs can execute all ELISA-specific hypercalls.

Therefore, if you wish to separate manager and guest hypercalls, you need to add some implementations for checking whether a VM, attempting a hypercall, is privileged or not.

The implementations of the manager hypercall functionalities are:
- [address translation from GPA to HPA](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L42-L44); this is used by the manager VM to [translate from GVA to HPA](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/vmcall.c#L104-L110).
- to [associate a created EPTP list with a vCPU of a guest VM](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L84-L89).

The implementations of the guest hypercall functionalities are:
- to [obtain a unique identifier of its vCPU which is used in the negotiation with the manager VM](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L94-L96).
- to [trigger the activation of the EPTP list configured by the manager VM](https://github.com/yasukata/kvm-elisa/blob/0fadc257ca8a99365ba8db09d03eed431881cdd8/elisa.patch#L97-L104).

### Section 5.2 : libelisa

In the current implementation, libelisa offers two primary functions: [```elisa_server```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L577-L579) for the manager VM programming and [```elisa_client```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L59) for the guest VM programming; the primary [task of elisa-util-exec described in the example section](#run-elisa-app-nop) is to execute either [```elisa_server```](https://github.com/yasukata/elisa-util-exec/blob/eb8a14a132fe7f9b0e7980c7a4ab3a30ef8a465f/main.c#L67) or [```elisa_client```](https://github.com/yasukata/elisa-util-exec/blob/eb8a14a132fe7f9b0e7980c7a4ab3a30ef8a465f/main.c#L72).

```elisa_server``` enters an infinite server loop, and ```elisa_client``` returns after the setup of the gate/sub EPT contexts has been completed.

#### The arguments of [```elisa_server```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L577-L579)

- ```const int server_port```: the port that the manager VM listens on.
- ```int (*server_cb)(int, uint64_t *, struct elisa_map_req **, int *, uint64_t *)```: the callback function [called during the sub EPT context setup](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L488).
- ```int (*server_exit_cb)(uint64_t *)```: the callback function [called when a guest VM, that has a sub EPT context, exits](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L572), and this is primarily preserved to implement application-specific destructors.

[```server_cb```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L488) is the point where programmers can customize how a sub EPT context is configured, and it takes the following arguments:
- ```int```: a socket file descriptor connected to a guest VM.
- ```uint64_t *```: a pointer to a 64-bit variable that a programmer can use for any purpose, and passed to ```server_exit_cb```; primarily assuming to assign an identifier in ```server_cb```.
- ```struct elisa_map_req **```: a pointer to an array of [```struct elisa_map_req```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/include/libelisa.h) page mapping requests that libelisa assumes to be prepared by ```server_cb```; subsequently, libelisa makes page table and EPT mappings according to the requests put in this array.
- ```int *```: number of mapping requests in the array of ```struct elisa_map_req```, set by ```server_cb```.
- ```uint64_t *```: a virtual address of the entry point of the core function executed in the sub EPT context; this value is [embedded](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L490) to [the code of the sub EPT context](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/server.c#L129).

Programmers can map pages allocated in a user-space process to a sub EPT context of a guest VM, by creating an array of [```struct elisa_map_req```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/include/libelisa.h#L61-L69).

libelisa-extra has [```elisa_create_program_map_req```](https://github.com/yasukata/libelisa-extra/blob/a3ccab11ffb65c7cb5be741bdd5c0f041c40c343/include/libelisa_extra/map.h#L35-L40) which helps programmers to load a shared library file to a sub EPT context of a guest VM in ```server_cb```; elisa-app-nop [uses this](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L48-L53) to load the program to a sub EPT context.

```elisa_create_program_map_req``` takes the following arguments:
- ```const char *program_filename```: the name of the shared library file to be loaded in the sub EPT context.
- ```uint64_t base_gpa```: the lowest GPA where the code will be located in the sub EPT context; GVA is the same as the one allocated in the process executing this on the manager VM. 
- ```void **handle```: the handle [opened by ```dlmopen```](https://github.com/yasukata/libelisa-extra/blob/a3ccab11ffb65c7cb5be741bdd5c0f041c40c343/include/libelisa_extra/map.h#L88); this is mainly used for finding the address of the entry point of the core function. elisa-app-nop [uses this](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/main.c#L57) to find the address of the entry function named [```entry_function``` in the code for the sub EPT context](https://github.com/yasukata/elisa-app-nop/blob/bc932930c0dd07fbee61a41c968f5476a8bde981/lib/libelisa-applib-nop/main.c#L19).
- ```struct elisa_map_req **map_req```: array of the memory mapping request, particularly for the code, and [manipulated in 
```elisa_create_program_map_req```](https://github.com/yasukata/libelisa-extra/blob/a3ccab11ffb65c7cb5be741bdd5c0f041c40c343/include/libelisa_extra/map.h#L70-L77).
- ```int *map_req_cnt```: number of the mapping request made by ```elisa_create_program_map_req```. 
- ```int *map_req_num```: the current length of the ```map_req``` array, that is [dynamically extended in ```elisa_create_program_map_req```](https://github.com/yasukata/libelisa-extra/blob/a3ccab11ffb65c7cb5be741bdd5c0f041c40c343/include/libelisa_extra/map.h#L63-L68).

#### The arguments of [```elisa_client```](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L59)
- ```const char *server_addr_str```: the string specifying the address of the manager VM.
- ```const int server_port```: the port the manager VM listens on.
- ```int client_cb(int)```: a callback function [called during the negotiation with the manager VM](https://github.com/yasukata/libelisa/blob/e46242e5ecd854a807f9ea1816dae3f292d5a250/client.c#L75); this is preserved so that the negotiation steps can be extended for passing application-specific information through the socket file descriptor passed as the argument (int).

