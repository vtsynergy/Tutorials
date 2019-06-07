# Minimal MetaCL-ization of OpenDwarfs/LUD

In this step we will walk through the _minimum_ set of changes necessary to convert all kernel calls in a small application to using a MetaCL-generated interface. Further, we will ensure the interface is **always** up-to-date with changes to the kernels by integrating it into a typical Makefile/automake-based build system.

#### The changes for this step are all captured within the patch file `LUD.MetaCL-minimal.patch`. We will perform the steps manually but if you want to compare, you can apply it starting from your home directory with the following set of commands
* `cp -r OpenDwarfs OpenDwarfs-MetaCL-minimal`
* `cd OpenDwarfs-MetaCL-minimal`
* `patch -p1 < ~/LUD.MetaCL-minimal.patch`

# Structure of a MetaCL command-line launch
MetaCL functions much like a compiler and is invoked in a similar manner with similar arguments. A full description of the options can be found in the main [MetaCL user tutorial](https://github.com/vtsynergy/MetaMorph/tree/master/metamorph-generators/opencl/docs/tutorials), but we will briefly deconstruct the invocation here.
* `path/to/binary/metaCL <file(s)> [options] -- -cl-std=<OpenCL version code> --include opencl-c.h -I /path/to/clang/OpenCL/headers/dir/ [args]`
    * `path/to/binary/metaCL` is just the path to the metaCL binary. If you are using the preconfigured VM or followed those intstructions, it is located at `~/MetaMorph/metamorph-generators/opencl/metaCL`, (The VM is also set up with that directory in the users PATH variable, so you may directly call `metaCL` without the leading path.)
	* `<file(s)>` is just a list of one or more `.cl` OpenCL kernel files that you wish the generate host-to-device interfaces for.
	* `[options]` is a set of _MetaCL options_ that control how MetaCL generates the host-to-device interface. The only one we will use for this tutorial is `--unified-output-file=<string>` which merges all the interfaces from multiple input files into one output `.c` file and associated `.h` header. (The default behavior is to generate one `.c/.h` pair per input, with an identical name to the input file.)
	* `--` Is a special _required_ token that indicates where the MetaCL files/options end, and where arguments to the underlying Clang compiler begin. (It must be present even if the rest of the invocation is empty.)
	* `-cl-std=<OpenCL version code>` While Clang supports parsing OpenCL, it is not a default behavior. We provide this argument to indicate which OpenCL standard we want it to parse for. (Supported options include `CL1.0`, `CL1.1`, `CL1.2`, and `CL2.0`, and this should be set to whatever your kernel compiler -- i.e. OpenCL vendor -- supports.)
	* `--include opencl-c.h` Similarly to the previous option, once Clang is running in OpenCL mode, it needs definitions of all the built-in kernel functions and data types that OpenCL supports. These are provided in a header file called `opencl-c.h`, and you must provide the path to the folder it is stored in via the `-I` argument. (For the preconfigured VM, this file is located in `/usr/lib/llvm-6.0/lib/clang/6.0.0/include`)
	* `[args]` Finally, in order to parse the OpenCL kernels, Clang needs to know about any other kernel preprocessing information that is specific to your project. This may include `-D` conditional compilation directives, or `-I` include paths. (For example the LUD application covered in this tutorial requires the `-D BLOCK_SIZE=16` define.) In a pre-existing application, these can typically be extracted from the existing `clBuildProgram` call.
	
# Make Targets to Auto-trigger MetaCL
As noted above, MetaCL functions like a compiler, and is intended to be a recurring part of the build system. In particular, we want to guarantee that every time a kernel file is changed, the MetaCL-generated interface is recreated to capture any elements that must be exposed to the host. Rather than relying on the developer to remember _to_ invoke, much less _how_ we will capture it as a make target just like we would a compiler invocation.

To begin, if you have not applied the patch file as noted above, from your home directory make a copy of the OpenDwarfs directory
* `cp -r OpenDwarfs-MetaCL-minimal`

Then change to the source directory for LUD, where we will make all our changes, and open the automake input file `Makefile.mk`. (If you prefer a graphical editor, you can follow along with the code changes with the "gedit" editor installed in the VM.)
* `cd OpenDwarfs-MetaCL-minimal/dense-linear-algebra/lud`
* `vim Makefile.mk`

First we will create a make target to specify the a command to generate a `.c` file for our interface. Since LUD has two kernels, we will use the `--unified-output-file` option and name the output `LUDModule`. Add the following code to `Makefile.mk`
```
lud_KERNELARGS = -DOPENCL -I . -D BLOCK_SIZE=16
dense-linear-algebra/lud/LUDModule.c: $(top_srcdir)/dense-linear-algebra/lud/lud_kernel.cl $(top_srcdir)/dense-linear-algebra/lud/lud_kernel_opt_gpu.cl
	cd ${top_srcdir}/dense-linear-algebra/lud && ~/MetaMorph/metamorph-generators/opencl/metaCL lud_kernel.cl lud_kernel_opt_gpu.cl --unified-output-file="LUDModule" -- -cl-std=CL1.2 --include opencl-c.h -I /usr/lib/llvm-6.0/lib/clang/6.0.0/include ${lud_KERNELARGS}
```
The second line specifies that `LUDModule.c` depends on both `lud_kernel.cl` and `lud_kernel_opt_gpu.cl`, and should be re-run any time either of those files is updated. The following line specifies the command to be run, which switches to the current source directory and then invokes MetaCL on the two kernel files. (In the first line, we use a make variable for the kernel preprocessor options to make updating them easier.)

The above changes will trigger MetaCL when the kernel files are updated _before_ any file that depends on `LUDModule.c` is compiled. However, the host application code that targets the MetaCL-generated interface will only reference the generated header file `LUDModule.h` and thus may be separately compiled _before_ `LUDModule.c`, in which case `LUDModule.h` may not exist yet, resulting in a compiler error. To account for this we add an additional target for `LUDModule.h` and add it to automake's `BUILT_SOURCES` list. The below lines should be added _above_ the previous step's.
```
BUILT_SOURCES = dense-linear-algebra/lud/LUDModule.h
dense-linear-algebra/lud/LUDModule.h: dense-linear-algebra/lud/LUDModule.c
```
`BUILT_SOURCES` is an automake variable that specifies files that should be generated before any other targets are run. This will ensure that LUDModule.h is already available 

# Updating existing Make Targets to use the generated interface
The final LUD binary will depend on a new object file for the generated host-to-device interface, as well as the MetaMorph libraries that provide the OpenCL support features that MetaCL leverages. To finish updating the Makefile we need to update three options.
```
lud_SOURCES = dense-linear-algebra/lud/lud.c dense-linear-algebra/lud/common.c dense-linear-algebra/lud/LUDModule.c
lud_LDFLAGS = -lmetamorph -lmm_opencl_backend -L$(HOME)/MetaMorph/lib -R$(HOME)/MetaMorph/lib
lud_CFLAGS = -I ~/MetaMorph/include -I ~/MetaMorph/metamorph-backends/opencl-backend/
```
* `lud_SOURCES` in an automake variable defining which `.c` files should be compiled to objects and linked into the final binary, add `dense-linear-algebra/lud/LUDModule.c` to it.
* `lud_LDFLAGS` is an automake variable to specifiy arguments for the final link step. `-lmetamorph` and `-lmm_opencl_backend` are the MetaMorph library dependencies, `-L$(HOME)/MetaMorph/lib` is the link-time path to the MetaMorph library directory, and the `-R` is the runtime path to the same.

We are now done with the Makefile upgrades. Save and close your editor. (On vim, hit escape and then type `wq` and Enter)

# Updating host application code
Now that the Makefile is updated, we need to return to the build directory and run `make` once to trigger the MetaCL invocation
* `cd ../../build && make`
On a successful run, the printed output will just be a list of the processed kernel files, and `LUDModule.c` and `LUDModule.h` will be generated in the LUD source directory

Navigate back to the source directory and open the main host file for editing
* `cd ../dense-linear-algebra/lud && vim lud.c`

The minimum set of changes only requires four real tasks:
1. Adding new header files
2. Sharing existing OpenCL state with the MetaCL-generated interface and MetaMorph
3. Informing the MetaCL-generated module of any custom build arguments required for the kernels
4. Replacing existing kernel launches

### Adding new header files
To reference the newly-generated interface, we must add its header file to each file that wants to use it. Additionally, we must include the MetaMorph OpenCL backend's header. Add the following two lines at the end of the header list:
```
#include "mm_opencl_backend.h"
#include "LUDModule.h"
```

### Sharing existing OpenCL state from the host application to MetaMorph and the MetaCL-generated interface
Since we are porting an existing application and trying to minimize the cost of entry, we will reuse the existing OpenCL state that was already manually created. To do this, you must explicitly share a cohesive set of OpenCL `cl_device_id`, `cl_platform_id`, `cl_context`, and `cl_command_queue` to MetaMorph via a call to `meta_set_state_OpenCL`. This will inform MetaCL of the state, and allow it to _auto-initialize_ the MetaCL-wrapped kernels for it. Add the below lines _after_ the existing state has been created but _before_ any `cl_programs` are built for it. (LUD does not have the `cl_platform_id` exposed, so we query it using the standard OpenCL API.)
```
   cl_platform_id plat;
	clGetDeviceInfo(device_id, CL_DEVICE_PLATFORM, sizeof(cl_platform_id), &plat, NULL);
	meta_set_state_OpenCL(plat, device_id, context, commands);
```

### Using custom build arguments
The underlying OpenCL compiler still needs information about any custom build arguments required for the kernel. These may be preprocessor options like those provided to the MetaCL invocation, or they may be performance-related options such as `--cl-fast-relaxed-math`. At the time of writing this tutorial, those can be set on a per-kernel-file basis via an `extern` exposed `char *`, which can be found in the `LUDModule.h` header file.
```
	__meta_gen_opencl_lud_kernel_custom_args = arg;
	__meta_gen_opencl_lud_kernel_opt_gpu_custom_args = arg;
```

### Replacing existing kernel launches
Finally, we must replace the existing kernel launches, their forward argument assignments, and any following `clFinish` synchronizations. We can also safely remove any error checks associated with `clSetKernelArg`, `clEnqueueNDRangeKernel`, `clEnqueueTask`, and `clFinish` used for invoking the kernel because by default MetaCL generates an error check for every single OpenCL API call is generates. (This can be disabled via an option, if desired.)
```
//errcode = clSetKernelArg(clKernel_diagonal, 0, sizeof(cl_mem), (void *) &d_m);
//errcode |= clSetKernelArg(clKernel_diagonal, 1, sizeof(int), (void *) &matrix_dim);
//errcode |= clSetKernelArg(clKernel_diagonal, 2, sizeof(int), (void *) &i);
//CHKERR(errcode, "Failed to set kernel arguments!");
//errcode = clEnqueueNDRangeKernel(commands, clKernel_diagonal, 1, NULL, globalWorkSize, localWorkSize, 0, NULL, &ocdTempEvent);
//clFinish(commands);
//errcode = clEnqueueNDRangeKernel(commands, clKernel_diagonal, 1, NULL, globalWorkSize, localWorkSize, 0, NULL, &ocdTempEvent);
errcode = meta_gen_opencl_lud_kernel_lud_diagonal(commands, grid, block, &d_m, matrix_dim, i, 0, &ocdTempEvent);
```

Additionally, we must convert any existing `globalWorkSize` and `localWorkSize` to a `grid` and `block` with the `a_dim3` data type provided by MetaMorph. (For consistency reasons MetaMorph and MetaCL-generated codes use the CUDA grid/block model rather than OpenCL's worksize model. `block == localWorkSize` and `grid == globalWorkSize/localWorkSize`. As OpenCL already requires that the globalWorkSize in each dimension be an integer multiple of the corresponding localWorkSize, it is only a semantic difference.) All three dimensions should be specified, but unused dimensions can be safely set to `1`. We add these new size structs before any of the kernels, and then replace the size assignments. Since MetaCL assumes all kernels are 3D, we must also be sure to reset any unused dimensions to 1.
```
a_dim3 grid={1,1,1}, block={1,1,1};
//localWorkSize[0] = BLOCK_SIZE;
//localWorkSize[1] = BLOCK_SIZE;
//globalWorkSize[0] = ((matrix_dim-i)/BLOCK_SIZE-1)*localWorkSize[0];
//globalWorkSize[1] = ((matrix_dim-i)/BLOCK_SIZE-1)*localWorkSize[1];
block[0] = BLOCK_SIZE;
block[1] = BLOCK_SIZE;
grid[0] = ((matrix_dim-i)/BLOCK_SIZE-1);
grid[1] = ((matrix_dim-i)/BLOCK_SIZE-1);
//RUN A 2D Kernel
grid[1] = 1; block[1] = 1;
```

The default assumption of the MetaCL-generated wrappers is that all kernels are run _synchronously_ (i.e. with a tailing `clFinish`), if you wish to run one asynchronously, you can set the `async` parameter of the generated interface to `true` or `1`. Finally, a `cl_event *` is exposed so that you may get the Enqueue event from the kernel, which in this case we will use for the OpenDwarfs timing infrastructure. These are the second to last, and last parameters of the generated wrapper, respectively.


### Validation
After MetaCL-izing the kernel invocations, we need to build and run the code to make sure it continues to behave as expected. Return to the build directory, build it, and run it with the same dataset as the unmodified version.
```
cd ../../build
make
./lud -i ~/OpendDwarfs-tests/dense-linear-algebra/lud/64.dat -v
```