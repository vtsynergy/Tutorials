This tutorial provides a walkthrough of applying the [MetaCL+MetaMorph](https://github.com/VTSynergy/MetaMorph) and [FLOCL](https://github.com/VTSynergy/FLOCL) OpenCL support tools to kernel files from a pre-existing OpenCL application from the [OpenDwarfs](https://github.com/VTSynergy/OpenDwarfs).

Both tools function like compilers, so the tutorial will introduce how to integrate them into an existing Makefile/Automake-based build system.

**MetaCL** is designed to consume OpenCL kernel files, and generate the host-to-device interface necessary to launch all the `kernel` functions contained within. This includes all `cl_program` and `cl_kernel`-related boilerplate, supporting simplified initialization and a reduced-complexity developer experience when producing OpenCL host code. Further, through its interaction with the MetaMorph OpenCL backend, it provides `cl_device`-related auto-selection and initialization, and automatic _reinitialization_ of MetaCL-wrapped kernels when the device is changed through the MetaMorph interface.

**FLOCL** is an extended version of the [clang-tidy](https://clang.llvm.org/extra/clang-tidy/) _linter_ tool. (Linters are intended to provide extra code verification, cleanup, and other maintenance _before_ compilation.) The FLOCL extensions to clang-tidy are designed to detect semantic faults and performance holes that are specific to OpenCL kernel development and OpenCL-on-FPGA kernel development, which were not previously available. (In time these modules will be folded into the upstream clang-tidy, and cease to be a standalone project.) By providing _preliminary_ static analysis passes, FLOCL is intended to reduce the frequency and severity of errors detected _after_ lengthy FPGA compile/place/route steps, reducing the total number of CPR iterations and thus improving developer productivity.

# Environment setup

**Prerequisites:**
* A virtual machine hypervisor (we used VirtualBox 5.2.22 for designing the tutorial)
* An internet connection

A pre-configured VM export was provided for the original tutorial. Instructions for how to configure an identical environment can be found [here](ManualInstallation.md).
**Expected time, including downloads and builds: 1-2 hours, depending on your internet connection and number of cores and RAM provided to the VM.**

**Post-conditions:**
* An Ubuntu 18.04.2 VirtualMachine with FLOCL, MetaCL, and all their dependencies installed
* (If using the provided VM export, it will also contain the example application and patch files referenced below)

# Example application setup (OpenDwarfs/LUD)

#### If you are using the pre-configured VM, the example setup has been performed already. Proceed to [Minimal MetaCL-ization](#Complete-MetaCL-ization).
We will use LUD from the OpenDwarfs. (For future reproducibility, we will use commit `fdec39374f7e08d`, which was the current master branch HEAD at the time of the tutorial).

Open a terminal (Click on the square of dots in the bottom left corner, or hit your Windows/Super key, and type "term" and select "Terminal" that comes up.)

Clone the OpenDwarfs
* `git clone https://github.com/VTSynergy/OpenDwarfs`
Switch the the Dwarfs directory
* `cd OpenDwarfs`
And make sure you're on the intended revision by checking out the hash below
* `git checkout fdec39374f7e08d`

OpenDwarfs comes with several test data sets for the applications. We will make multiple copies of the Dwarfs source tree throughout the workshop, and we don't want to waste space with excess copies of the tests. Move them out to the home directory
* `mv test ~/OpenDwarfs-tests`

Finally, this tutorial provides several patch files that achieve all of the changes necessary throughout each of the stages of the tutorial. Return to the home directory and clone this repository to download them.
* `cd ~`
* `git clone https://github.com/VTSynergy/Tutorials.git`
The patch files will then be contained in `~/Tutorials/SHREC SMW19 FLOCL+MetaCL/`. For consistency with the preconfigured VM, copy them to your host directory
* `cp ~/Tutorials/SHREC\ SMW19\ FLOCL+MetaCL/LUD.* ~/`

Finally, we will build the default version of the LUD application from the Dwarfs to confirm our OpenCL environment is working.
* `cd OpenDwarfs`
* `./autogen.sh`
* `cd build`
* `../configure --with-apps=lud`
* `make`

Then, we will run LUD on a small validation test
* `./lud -i ~/OpendDwarfs-tests/dense-linear-algebra/lud/64.dat -v`
If successful, it should print the contents of the matrix, and a line that says `SUCCESS: 0 difference of more than 0.0001f (absolute)`, followed by OpenDwarfs's timing information.

# Minimal MetaCL-ization

Now that the environment and test appliation are set up, we can proceed with the minimal conversion of the application to use a MetaCL-generated OpenCL host-to-device interface. The detailed instructions can be found [here](MinimalMetaCLization.md).

# Complete MetaCL-ization

The previous step perfomed the bare minimum steps to running OpenCL kernels via MetaCL-generated host wrappers. [These instructions](CompleteMetaCLization.md) detail additional options for minimizing the boilerplate requirements and accessing extra features afforded by MetaCL and MetaMorph.

# FLOCL-ization

As a separate step, the application's OpenCL kernel files can be passed through the FLOCL (clang-tidy with added OpenCL- and OpenCL-on-FPGA-specific linters) tool to analyze them for semantic or performance faults. This tutorial and associated patch file presume this step is performed after the "Complete MetaCL-ization" step. [Detailed instructions and commentary.](FLOCLization.md)