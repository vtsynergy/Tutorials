# Complete MetaCL-ization and MetaMorph-osis

The previous version left in a lot of the existing boilerplate features, which are now redundant. It also didn't take advantage of some of the behind-the-scenes features that MetaMorph offers. In this step we complete the MetaCL-ization of the code by eliminating any now-unnecessary boilerplate. Further we "MetaMorph-osize" the code to use MetaMorph's buffer and data transfer API to take advantage of its implicit timing capability. (Though the OpenDwarfs have their own manual timing infrastructure, not all projects do. Auto-timing of MetaCL-ized kernels will be implemented as future work.)

#### As with the previous step, the changes for this step are all captured within the patch file `LUD.MetaCL-complete.patch`.
* `cd ~`
* `cp OpenDwarfs-MetaCL-minimal OpenDwarfs-MetaCL-complete`
* `patch -p1 < ../LUD.MetaCL-complete.patch`

### Replace existing OpenCL device/context management with MetaMorph
In the OpenDwarfs, all the initial device setup code is abstracted away behind a call to `ocd_initCL`, a manually written function. Typically this takes a minimum of OpenCL API calls to identify an OpenCL vendor platform and a device within it, and then initialized an execution/memory context and command queue for it. All of these steps are captured by MetaMorph, which underlies MetaCL-generated code and provides auto-initialization and device-swapping mechanisms. To switch, we just remove the existing calls and replace them with a call to `meta_set_acc`, which takes a device number and a "MetaMorph mode" (which for this work should be hardcoded to `metaModePreferOpenCL`
```
-	ocd_initCL();
+	meta_set_acc(-1, metaModePreferOpenCL);
```
Typically the zeroth parameter is the OpenCL `[0:N-1]` device number you wish to target. These are iterated over from all OpenCL platforms installed on the system. `-1` is a signifier that the user should let MetaMorph choose. In this case the OpenCL backend will print all discovered devices out and choose the zeroth. (There is also an environment variable `TARGET_DEVICE="<string device name>"` that will choose the first device with a matching name.)

You may then want to reuse MetaMorph's OpenCL state with your existing OpenCL code. This can be achieved via the `meta_get_state_OpenCL` call, which takes pointers to a `cl_platform_id`, `cl_device_id`, `cl_context`, and `cl_command_queue`. These will be assigned to the values currently in use whenever it is called. (The pointers will not be modified if the device is changed via a subsequent call to `meta_set_acc` without another `get_state` query.)
```
+	cl_platform_id plat;
+	meta_get_state_OpenCL(&plat, &device_id, &context, &commands);
```

### (Optional) Explicit registration of the MetaCL-generated wrapper with MetaMorph
Registration of the MetaCL-generated wrapper module occurs automatically at the first call to a generated wrapper function. This will then perform all the auto-initialization steps necessary to build the OpenCL programs and kernels for the current MetaMorph OpenCL state. Depending on the number of kernels and speed of your OpenCL backend compiler, this can have a peformance cost that you may not want to pay in your critical loop. To pre-pay it, you may instead explicitly register it. (This should be performed after you have set any custom build arguments for the OpenCL kernels, as they will be auto-initialized at this time.) Each generated module will have a registry function, simply pass a pointer to it to `meta_register_module`. (The registry function is not intended for user modification and is the interface through which MetaMorph learns the capabilities of the MetaCL-generated module.)
* `meta_register_module(&meta_gen_opencl_LUDModule_registry);`

### Replace existing buffer allocations, transfers, and releases with MetaMorph's API
While you may continue to use any existing `cl_mem` buffers created for the MetaMorph-generated `cl_context`, MetaMorph provides a simplified API for creating buffers. It provides some functionalities behind the scenes if you intend to interoperate with non-OpenCL platforms via MetaMorph, but for our purposes we are just using them for their implicit auto-timing. (The OpenDwarfs also have event-based timing, but typically smaller hand-written OpenCL applications do not leverage them.) Like MetaCL-wrapped kernels, MetaMorph buffer transfers are assumed to be synchronous by default, removing the need for a `clFinish`, but can be set to asynchronous with a boolean flag.
```
-    d_m = clCreateBuffer(context, CL_MEM_READ_WRITE, matrix_dim*matrix_dim*sizeof(float), NULL, &errcode);
+	errcode = meta_alloc(&d_m, matrix_dim*matrix_dim*sizeof(cl_float));
-	errcode = clEnqueueWriteBuffer(commands, d_m, CL_TRUE, 0, matrix_dim*matrix_dim*sizeof(float), (void *) m, 0, NULL, &ocdTempEvent);
-
-	clFinish(commands);
-
-	START_TIMER(ocdTempEvent, OCD_TIMER_H2D, "Matrix Copy", ocdTempTimer)
-		END_TIMER(ocdTempTimer)
+	errcode = meta_copy_h2d(d_m, (void *) m, matrix_dim*matrix_dim*sizeof(cl_float), 0);
...
-	clReleaseMemObject(d_m);
+	meta_free(d_m);
```

### Remove unused boilerplate
At this point, we can strip out all the OpenCL API variables, boilerplate calls, and error checking that we no longer need. The patch file shows the lines removed for LUD, which we won't cover here. Typically, most `cl_kernel`s, `cl_program`s, program source buffers, and error checking-related variables and calls can be safely eliminated at this time.

### Validation
As with the previous step, we need to validate against the built-in CPU implementation. Return to the build directory, build it, and run it with the same dataset as the unmodified version.
```
cd ../../build
make
./lud -i ~/OpendDwarfs-tests/dense-linear-algebra/lud/64.dat -v
```