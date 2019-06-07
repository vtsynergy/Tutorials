# Automatic FLOCL Linting
After we have validated that the completly MetaCL-ized application is producing correct results, we can then start to analyze the kernels for improvements. As with MetaCL, FLOCL is intended to be included as a recurring part of the build process. In this step we will add Makefile targets to ensure FLOCL is invoked every time the project is built, to ensure kernels are analyzed frequently during development. We will then walk through two typical recommendations that are exhibited by the OpenDwarfs/LUD test application we have been looking at.

#### As with the previous steps, the changes for this step are all captured within the patch file `LUD.FLOCLized.patch`.
If you want to make all the changes yourself, just make a copy of the previous version's directory but do not apply the patch command.
* `cp -r OpenDwarfs-MetaCL-complete OpenDwarfs-FLOCLized`
* `cd OpenDwarfs-FLOCLized`
* `patch -p1 < LUD.FLOCLized.patch`

# Structure of a FLOCL command-line launch
Like MetaCL, FLOCL functions much like a compiler and is invoked in a similar manner with similar arguments. As an extension of the `clang-tidy` linter, [Clang's existing documentation](https://clang.llvm.org/extra/clang-tidy/) can be referred to for most issues. We detail FLOCL-specific parts of the invocation below
* `path/to/binary/clang-tidy <file(s)> [clang-tidy options] -- -cl-std=<OpenCL version code> --include opencl-c.h -I /path/to/clang/OpenCL/headers/dir/ [args]`
    * `path/to/binary/metaCL` is just the path to the clang-tidy/flocl binary. If you are using the preconfigured VM or followed those intstructions, it is located at `~/FLOCL/standalone_build/build/flocl/clang-tidy/tool/clang-tidy`, (The VM is also set up with that directory in the users PATH variable and a `flocl` alias, so you may directly call `flocl` without the leading path.)
	* `<file(s)>` is just a list of one or more `.cl` OpenCL kernel files that you wish the generate host-to-device interfaces for.
	* `[options]` is a set of _FLOCL/clang-tidy options_ that control what linters FLOCL runs and options to them. The only one we will use for this tutorial is `--checks=<comma-delimited-list>` which allows us to turn off the existing clang-tidy linters (for C++ and other non-OpenCL-kernel passes) and just run the FPGA and OpenCL ones.
	* `--` Is a special _required_ token that indicates where the FLOCL files/options end, and where arguments to the underlying Clang compiler begin. (It must be present even if the rest of the invocation is empty.)
	* `-cl-std=<OpenCL version code>` While Clang supports parsing OpenCL, it is not a default behavior. We provide this argument to indicate which OpenCL standard we want it to parse for. (Supported options include `CL1.0`, `CL1.1`, `CL1.2`, and `CL2.0`, and this should be set to whatever your kernel compiler -- i.e. OpenCL vendor -- supports.)
	* `--include opencl-c.h` Similarly to the previous option, once Clang is running in OpenCL mode, it needs definitions of all the built-in kernel functions and data types that OpenCL supports. These are provided in a header file called `opencl-c.h`, and you must provide the path to the folder it is stored in via the `-I` argument. (For the preconfigured VM, this file is located in `/usr/lib/llvm-6.0/lib/clang/6.0.0/include`)
	* `[args]` Finally, in order to parse the OpenCL kernels, Clang needs to know about any other kernel preprocessing information that is specific to your project. This may include `-D` conditional compilation directives, or `-I` include paths. (For example the LUD application covered in this tutorial requires the `-D BLOCK_SIZE=16` define.) In a pre-existing application, these can typically be extracted from the existing `clBuildProgram` call.


### Adding FLOCL Make targets
We will add make targets for FLOCL to ensure it is invoked regularly on the kernel files. However, unlike compilation steps or MetaCL code generation, no distinct output file is produced. Further, we want to run it every time the Makefile is used, regardless of whether the kernels have changed, so we will have to use a "force target".
Change to the source directory for LUD, where we will make all our changes, and open the automake input file `Makefile.mk`. (If you prefer a graphical editor, you can follow along with the code changes with the "gedit" editor installed in the VM.)
* `cd OpenDwarfs-MetaCL-minimal/dense-linear-algebra/lud`
* `vim Makefile.mk`

First we will set an automake variable to specify that the lud binary depends on the two new make targets we are going to create.
* `lud_DEPENDENCIES = flocl-lud-kernel flocl-lud-kernel-opt-gpu`

Next, we will add the two new make targets (one for each kernel file) with a dependency on a not-yet-created `lud-FORCE` target.
```
+flocl-lud-kernel: lud-FORCE
+	cd ${top_srcdir}/dense-linear-algebra/lud && ~/FLOCL/standalone_build/build/flocl/clang-tidy/tool/clang-tidy -checks=-*,OpenCL*,FPGA* lud_kernel.cl -- -cl-std=CL1.2 --include opencl-c.h -I /usr/lib/llvm-6.0/lib/clang/6.0.0/include ${lud_KERNELARGS}
+
+flocl-lud-kernel-opt-gpu: lud-FORCE
+	cd ${top_srcdir}/dense-linear-algebra/lud && ~/FLOCL/standalone_build/build/flocl/clang-tidy/tool/clang-tidy -checks=-*,OpenCL*,FPGA* lud_kernel_opt_gpu.cl -- -cl-std=CL1.2 --include opencl-c.h -I /usr/lib/llvm-6.0/lib/clang/6.0.0/include ${lud_KERNELARGS}
```
This has a similar structure to the MetaCL invocation we added in a previous step, and we reuse the preprocessor options we set up for it. Of note is the `checks` option to FLOCL, which demonstrates the comma-delimited syntax for specifying checks to be run. First we remove all checks from the list `-*` (minus wildcard), and then we add the OpenCL- (`OpenCL*`) and FPGA-specific (`FPGA*`) checks that make up FLOCL.

Finally we will add an empty target called `lud-FORCE` which is known as a Make "force target". Since it has no dependencies and no recipe, the make system treats as if it is has been freshly rebuilt every time the Makefile is invoked. In turn this forces the linting targets to be re-run since their dependency target has been "updated".
```
+lud-FORCE:
+
```

# Typical FLOCL Output
FLOCL emits warnings similar in structure to those emitted by a compiler (since it's built on a compiler!) Two examples of linter output that will show up in the selected version of OpenDwarfs/LUD are shown below.

### OpenCL-possibly-unreachable-barrier
The OpenCL standard specifies that when a barrier is used within a branching construct, all if one `workitem` in a `workgroup` encounters the barrier **all** workitems must encounter the barrier. When barriers are contained within loop or branch structures, if the condition is thread ID-dependent, there is a chance that this restriction may be violated. (In particular in loops with thread-dependent traversal counts, or `if`s without corresponding `else`s). FLOCL identifies barrier calls within such regions and flags them if the conditional has a chance of being ID-dependent.

This lint check has false-positive reduction based on heuristics. Even though the patch file adds else branches with a corresponding number of barriers, the check still flags some of them. You may silence **all** lint checks on a line by adding the `//NOLINT` comment to the end of the line.

### FPGA-unroll-loops
The Intel/Altera FPGA programming guide recommends that when loops are nested, interior loops be unrolled if the bound is known and small, as it can generate more efficient hardware. As one of its functions this linter detects inner loops that do not contain an unroll pragma to draw attention to them.
```
/home/shrec/OpenDwarfs-Example/dense-linear-algebra/lud/lud_kernel.cl:135:2: warning: The performance of the kernel could be improved by unrolling this loop with a #pragma unroll directive [FPGA-unroll-loops]
         for (i=0; i < BLOCK_SIZE; i++)
		 ^
```
Since this kernel makes use of a statically defined BLOCK_SIZE, the maximum number of iterations of the interior loops in this kernel is always known, we can thus safely add an unroll pragma to them.

### FPGA-id-dependent-backward-branch
Backward branches (i.e. loops) can cause inefficiencies and pipeline stalls in FPGAs if the loop condition is thread-dependent. It would be unsafe to start the the subsequent thread's loop in the pipeline until after the previous thread has finished all its runtime-dependent iterations. Therefore the Intel/Altera programming guide suggest that thread ID-dependent values be removed from loop conditional expressions if possible. In the LUD kernel, all the loops are actually thread invariant, so we do not have to make those changes. However, the linter produces diagnostic information about how it tracks ID-dependent values to identify such conditionals.
```
/home/shrec/OpenDwarfs-Example/dense-linear-algebra/lud/lud_kernel.cl:136:3: warning: assignment of ID-dependent variable 'sum' declared at /home/shrec/OpenDwarfs-Example/dense-linear-algebra/lud/lud_kernel.cl:128:2 [FPGA-ID-dependent-backward-branch]
                sum += m[(global_row_id+get_local_id(1))*matrix_dim+offset+i] * m[(offset+i)*matrix_dim+global_col_id+get_local_id(0)];
				^
```
As with the OpenCL-possibly-unreachable-barrier check, these outputs can be permanently silenced with the `NOLINT` comment tag.


# Validation
Since we have modified the kernels, we need to yet again validate that the application is still producing correct results. Return to the build directory, build it, and run it with the same dataset as the unmodified version.
```
cd ../../build
make
./lud -i ~/OpendDwarfs-tests/dense-linear-algebra/lud/64.dat -v
```