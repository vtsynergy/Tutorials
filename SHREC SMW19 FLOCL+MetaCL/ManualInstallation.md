The instructions below will help you set up the intended software environment for this tutorial.

#### Expected time 1.5 hours
* Configuring the VM: 5 minutes
* Prerequisite Installs: 30 minutes
    * Ubuntu install: 15 minutes
	* Apt updates and dependencies: 15 minutes
* Downloading and Building MetaMorph and MetaCL: 10 Minutes
    * Building MetaCL: 5 minutes
	* Building MetaMorph: 3 minutes
	* Environment Configuration 2 minutes
* Downloading and Building FLOCL: 40 Minutes
    * Building FLOCL: 38 minutes
	* Environment Configuration: 2 minutes

#### Prerequisites
* A virtual machine hypervisor (we used VirtualBox 5.2.22 for designing the tutorial)
* An internet connection

#### Postconditions
* Ubuntu 18.04 VM created and updated with tutorial dependencies installed via apt-get
* MetaCL+MetaMorph installed in the VM
* FLOCL installed in the VM

# Setting up the Virtual Machine (5 minutes)
Download ubuntu 18.04.2 disk image
Available at: http://releases.ubuntu.com/18.04/ubuntu-18.04.2-desktop-amd64.iso (Download took ~4 minutes on VT's eduroam)

The creation wizard will help with the following
* Select 4GB RAM
* Select create a new disk image -> VDI (dynamically allocated)
* On the next page pick the size. The base Ubuntu install takes about 9GB, we allocated 16GB for safety.


Choose the VM
* Select Settings--> System-->Processor (choose the number of CPUs)
Insert media(Settings -> storage -> controller: IDE -> "Empty" -> Optical drive (on the right, click the CD image) -> select the iso you downloaded previously
Start the VM to proceed with the next stage

# Installing Ubuntu and pre-requisites to the VM
#### Expected time 30 minutes

If you've installed Ubunutu before these steps should feel familiar.
1. At the welcome screen select "Install Ubuntu"
2. Select your desired keyboard layout
3. Select Minimal installation and "download updates while installing ubuntu"
4. Select "Erase disk and install Ubuntu"
5. "Install Now"
6. When it asks if you want to "Write the changes to disks?" select "continue"
7. Pick your appropriate home time zone
8. **Some paths may assume the username is "shrec" as was configured in the pre-packaged VM export, pick that and an appropriate computer name**
9. Restart to finish the install (it will tell you to eject the install media before restarting, but VirtualBox should have done this for you)

After restart we want to install all the dependency packages our tools and test application will rely on.
* bring up a terminal (press your Super/Windows key or click on the squares in the lower left corner, and type "term" and select the gterminal application that comes up)
Install all updates
* `sudo apt-get update` 
* `sudo apt-get upgrade`
Install our dependencies
* `sudo apt-get install vim cmake git ocl-icd-libopencl1 opencl-headers clinfo ocl-icd-opencl-dev make build-essential zlib1g zlib1g-dev autoconf libtool libpocl-dev libclang-6.0-dev`
By this point the graphical package manager should have popped up and asked you to install security updates. Select them all and it will ask you to restart after they are installed.

# Downloading, Building, and Configuration of MetaCL and MetaMorph
#### Expected time 5 minutes
We will first install the MetaCL code generator from source. It relies on the Clang 6.0.0 packages we installed earlier.

First clone the MetaMorph/MetaCL repository from GitHub
* `git clone https://github.com/VTSynergy/MetaMorph`

Then change into the source directory
* `cd MetaMorph`

To compile MetaCL, we need to build the "generators" make target. (MetaCL's Makefile should detect the apt-get installation of Clang we performed earlier)
* `make generators`
This will produce the `metaCL` binary in `~/MetaMorph/metamorph-generators/opencl/`. We'll come back to this later

We will also need the MetaMorph library when the time comes to run our test application. It uses environment variables for its configuration. More details on that can be found on GitHub. For our purposes, we only have a POCL (Open source CPU OpenCL) installation, and don't want to take advantage of CUDA or OpenMP backends. We will also make use of some of MetaMorph's automatic timing infrastructure in later steps.
* `USE_OPENMP=false USE_TIMERS=true USE_OPENCL=true make all`
This step will produce `libmetamorph.so` and `libmm_opencl_backend.so` in `~/MetaMorph/lib`

For convenience later we want to configure a few environment variables in our user profile to make using MetaCL and MetaMorph easier
* Edit `~/.bashrc` (I use `vim` from the terminal, but you may also use `gedit` graphically or your prefered command line tool)
* at the bottom add the following three lines to configure your environment to locate the MetaCL binary, MetaMorph libraries, and MetaMorph header files
    * `export PATH=$PATH:$HOME/MetaMorph/metamorph-generators/opencl/`
    * `export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/MetaMorph/lib`
    * `export C_INCLUDE_PATH=$C_INCLUDE_PATH:/usr/include/clang/6.0.0/include`
    * `export C_INCLUDE_PATH=$C_INCLUDE_PATH:$HOME/MetaMorph/include`

**Note for those who wish to use a distribution/version other than 18.04.2: Follow the install instructions from the [MetaMorph/MetaCL repository](https://github.com/vtsynergy/MetaMorph/tree/master/metamorph-generators/opencl/docs/tutorials)**

# Downloading, Building, and Configuration of FLOCL
Expected time 40 minutes

FLOCL is a special branch of the `clang-tidy` linter tool. Eventually our new passes will be upstreamed but for now it exists as a separate repository. For this tutorial we have created a custom branch that includes numerous backports and special standalone build commands to make it compatible with the Clang 6.0.0 installed in Ubuntu 18.04. This has the benefit of not requiring a complete source install of Clang as is typical for clang-tidy.

**For all other environments you will want to follow the official [Extra Clang Tools 9 documentation](https://clang.llvm.org/extra/clang-tidy/Contributing.html) to do an in-source-tree build and substitute our repository's _master_ branch for the `llvm/tools/clang/tools/extra` directory.**

Begin by returning to your home directory and cloning the `smw19-backport` branch of our FLOCL repository
* `cd ~`
* `git clone --branch smw19-backport https://github.com/VTSynergy/FLOCL`

Navigate into FLOCL's `standalone_build` directory and set up and build space.
* `cd standalone_build`
* `mkdir build`
* `cd build`

Invoke our custom CMake commands to configure the Makefiles
* `cmake ../`

And then build and link the `clang-tidy` target (FLOCL). This step will compile all the dependencies of clang-tidy, including the existing linters which are not specific to OpenCL or FPGA development but an added benefit of using an existing tool. It should take about 30-35 minutes with the 4GB of RAM and 2 cores we configured in the VM setup.
* `make clang-tidy -j2`

This will result in the `clang-tidy` binary being output in `~/FLOCL/standalone_build/build/flocl/clang-tidy/tool/`

Like MetaCL, we will add the new path to a few environment variables to make using FLOCL easier.
* Edit `~/.bashrc` again and at the bottom add
    * `export PATH=$PATH:$HOME/FLOCL/standalone_build/build/flocl/clang-tidy/tool/`
	* `alias flocl=clang-tidy`

With that, the VM should be set up and ready to use.