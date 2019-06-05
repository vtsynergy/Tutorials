# WIP

# Environment setup

**Prerequisites:**
* A virtual machine hypervisor (we used VirtualBox 5.2.22 for designing the tutorial)
* An internet connection

A pre-configured VM export was provided for the original tutorial. Instructions for how to configure an identical environment can be found [here](ManualInstallation.md).
**Expected time, including downloads and builds: 1-2 hours, depending on your internet connection and number of cores and RAM provided to the VM.**

**Post-conditions:**
* An Ubuntu 18.04.2 VirtualMachine with FLOCL, MetaCL, and all their dependencies installed
* (If using the provided VM export, it will also contain the example application and patch files referenced below)

# Example application

We will use LUD from the OpenDwarfs. (For future reproducibility, we will use commit `fdec39374f7e08d`, which was the current master HEAD at the time of the tutorial).

cd ~
git clone https://github.com/VTSynergy/OpenDwarfs
cd OpenDwarfs
git checkout fdec39374f7e08d
Move the tests back to the homedir so they don't take up too much space

<<Pull this repository for the patches, and copy them to the homedirectory for convenience>>

<<Build and initial test>>
cd OpenDwarfs
./autogen.sh
cd build
../configure --with-apps=lud
make

# Minimal MetaCL-ization

Now that the environment and test appliation are set up, we can proceed with the minimal conversion of the application to use a MetaCL-generated OpenCL host-to-device interface. The detailed instructions can be found [here](MinimalMetaCLization.md).

# Complete MetaCL-ization

The previous step perfomed the bare minimum steps to running OpenCL kernels via MetaCL-generated host wrappers. [These instructions](CompleteMetaCLization.md) detail additional options for minimizing the boilerplate requirements and accessing extra features afforded by MetaCL and MetaMorph.

# FLOCL-ization

As a separate step, the application's OpenCL kernel files can be passed through the FLOCL (clang-tidy with added OpenCL- and OpenCL-on-FPGA-specific linters) tool to analyze them for semantic or performance faults. This tutorial and associated patch file presume this step is performed after the "Complete MetaCL-ization" step. [Detailed instructions and commentary.](FLOCLization.md)