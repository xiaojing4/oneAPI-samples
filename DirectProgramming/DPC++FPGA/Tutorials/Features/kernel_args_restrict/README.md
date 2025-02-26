
# Avoiding Aliasing of Kernel Arguments
This tutorial explains the  `kernel_args_restrict` attribute and its effect on the performance of FPGA kernels.

| Optimized for                     | Description
|:---                               |:---
| OS                                | Linux* Ubuntu* 18.04/20.04 <br> RHEL*/CentOS* 8 <br> SUSE* 15 <br> Windows* 10
| Hardware                          | Intel&reg; Programmable Acceleration Card (PAC) with Intel Arria&reg; 10 GX FPGA <br> Intel&reg; FPGA Programmable Acceleration Card (PAC) D5005 (with Intel Stratix&reg; 10 SX) <br> Intel&reg; FPGA 3rd party / custom platforms with oneAPI support <br> **Note**: Intel&reg; FPGA PAC hardware is only compatible with Ubuntu 18.04*
| Software                          | Intel&reg; oneAPI DPC++ Compiler <br> Intel&reg; FPGA Add-On for oneAPI Base Toolkit
| What you will learn               |  The problem of *pointer aliasing* and its impact on compiler optimizations. <br> The behavior of the `kernel_args_restrict` attribute and when to use it on your kernel. <br> The effect this attribute can have on your kernel's performance on FPGA.
| Time to complete                  | 20 minutes


## Purpose
Due to pointer aliasing, the compiler must be conservative about optimizations that reorder, parallelize or overlap operations that could alias. This tutorial demonstrates the use of the SYCL*-compliant `[[intel::kernel_args_restrict]]` kernel attribute, which should be applied any time you can guarantee that kernel arguments do not alias. This attribute enables more aggressive compiler optimizations and often improves kernel performance on FPGA.


### What Is Pointer Aliasing?
Pointer aliasing occurs when the same memory location can be accessed using different *names* (i.e., variables). For example, consider the code below. Here, the value of the variable `pi` can be changed in three ways: `pi=3.14159`, `*a=3.14159` or `*b=3.14159`. In general, the compiler has to be conservative about which accesses may alias to each other and avoid making optimizations that reorder and/or parallelize operations.

```c++
float pi = 3.14;
float *a = &pi;
float *b = a;
```
### Pointer Aliasing of Arguments
Consider the function illustrated below. Though the intention of the code is clear to the reader, the compiler cannot guarantee that `in` does not alias with `out`. Imagine a degenerate case where the function was called: like this `myCopy(ptr, ptr+1, 10)`. This would cause `in[i]` and `out[i+1]` to alias to the same address, for all `i` from 0 to 9.
```c++
void myCopy(int *in, int *out, size_t int size) {
  for(size_t int i = 0; i < size; i++) {
    out[i] = in[i];
  }
}
```
This possibility of aliasing forces the compiler to be conservative. Without more information from the developer, it cannot make any optimizations that overlap, vectorize or reorder the assignment operations. Doing so would result in functionally incorrect behavior if the compiled function is called with aliasing pointers.

If this code is compiled to FPGA, the performance penalty of this conservatism is severe. The loop in `myCopy` cannot be pipelined, because the next iteration of the loop cannot begin until the current iteration has completed.

### A Promise to the Compiler
The developer often knows that pointer arguments will never alias in practice, as with the `myCopy` function. In your program, you can use the `[[intel::kernel_args_restrict]]` attribute to inform the compiler that none of a kernel's arguments will alias to any another, thereby enabling more aggressive optimizations. If the non-aliasing assumption is violated at runtime, the result will be undefined behavior.

C and OpenCL programmers may recognize this concept as the `restrict` keyword.

### Tutorial Code Description
In this tutorial, we will show how to use the `kernel_args_restrict` attribute for your kernel and its effect on performance. We show two kernels that perform the same function; one with and one without `[[intel::kernel_args_restrict]]` being applied to it. The function of the kernel is simple: copy the contents of one buffer to another. We will analyze the effect of the `[[intel::kernel_args_restrict]]` attribute on the kernel's performance by analyzing loop II in the reports and the latency of the kernel on actual hardware.

### Additional Documentation
- [Explore SYCL* Through Intel&reg; FPGA Code Samples](https://software.intel.com/content/www/us/en/develop/articles/explore-dpcpp-through-intel-fpga-code-samples.html) helps you to navigate the samples and build your knowledge of FPGAs and SYCL.
- [FPGA Optimization Guide for Intel&reg; oneAPI Toolkits](https://software.intel.com/content/www/us/en/develop/documentation/oneapi-fpga-optimization-guide) helps you understand how to target FPGAs using SYCL and Intel&reg; oneAPI Toolkits.
- [Intel&reg; oneAPI Programming Guide](https://software.intel.com/en-us/oneapi-programming-guide) helps you understand target-independent, SYCL-compliant programming using Intel&reg; oneAPI Toolkits.

## Key Concepts
* The problem of *pointer aliasing* and its impact on compiler optimizations.
* The behavior of the `kernel_args_restrict` attribute and when to use it on your kernel.
* The effect this attribute can have on your kernel's performance on FPGA.

## Building the `kernel_args_restrict` Tutorial

> **Note**: If you have not already done so, set up your CLI
> environment by sourcing  the `setvars` script located in
> the root of your oneAPI installation.
>
> Linux*:
> - For system wide installations: `. /opt/intel/oneapi/setvars.sh`
> - For private installations: `. ~/intel/oneapi/setvars.sh`
>
> Windows*:
> - `C:\Program Files(x86)\Intel\oneAPI\setvars.bat`
>
>For more information on environment variables, see **Use the setvars Script** for [Linux or macOS](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-programming-guide/top/oneapi-development-environment-setup/use-the-setvars-script-with-linux-or-macos.html), or [Windows](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-programming-guide/top/oneapi-development-environment-setup/use-the-setvars-script-with-windows.html).

### Include Files
The included header `dpc_common.hpp` is located at `%ONEAPI_ROOT%\dev-utilities\latest\include` on your development system.

### Running Samples in Intel&reg; DevCloud
If running a sample in the Intel&reg; DevCloud, remember that you must specify the type of compute node and whether to run in batch or interactive mode. Compiles to FPGA are only supported on fpga_compile nodes. Executing programs on FPGA hardware is only supported on fpga_runtime nodes of the appropriate type, such as fpga_runtime:arria10 or fpga_runtime:stratix10.  Neither compiling nor executing programs on FPGA hardware are supported on the login nodes. For more information, see the Intel&reg; oneAPI Base Toolkit Get Started Guide ([https://devcloud.intel.com/oneapi/documentation/base-toolkit/](https://devcloud.intel.com/oneapi/documentation/base-toolkit/)).

When compiling for FPGA hardware, it is recommended to increase the job timeout to 12h.


### Using Visual Studio Code*  (Optional)

You can use Visual Studio Code (VS Code) extensions to set your environment, create launch configurations,
and browse and download samples.

The basic steps to build and run a sample using VS Code include:
 - Download a sample using the extension **Code Sample Browser for Intel&reg; oneAPI Toolkits**.
 - Configure the oneAPI environment with the extension **Environment Configurator for Intel&reg; oneAPI Toolkits**.
 - Open a Terminal in VS Code (**Terminal>New Terminal**).
 - Run the sample in the VS Code terminal using the instructions below.

To learn more about the extensions and how to configure the oneAPI environment, see the
[Using Visual Studio Code with Intel&reg; oneAPI Toolkits User Guide](https://software.intel.com/content/www/us/en/develop/documentation/using-vs-code-with-intel-oneapi/top.html).


### On a Linux* System

1. Generate the `Makefile` by running `cmake`.
     ```
   mkdir build
   cd build
   ```
   To compile for the Intel&reg; PAC with Intel Arria&reg; 10 GX FPGA, run `cmake` using the command:
    ```
    cmake ..
   ```
   Alternatively, to compile for the Intel&reg; FPGA PAC D5005 (with Intel Stratix&reg; 10 SX), run `cmake` using the command:
   ```
   cmake .. -DFPGA_BOARD=intel_s10sx_pac:pac_s10
   ```
   You can also compile for a custom FPGA platform. Ensure that the board support package is installed on your system. Then run `cmake` using the command:
   ```
   cmake .. -DFPGA_BOARD=<board-support-package>:<board-variant>
   ```

2. Compile the design through the generated `Makefile`. The following build targets are provided, matching the recommended development flow:

   * Compile for emulation (fast compile time, targets emulated FPGA device):
      ```
      make fpga_emu
      ```
   * Generate the optimization report:
     ```
     make report
     ```
   * Compile for FPGA hardware (longer compile time, targets FPGA device):
     ```
     make fpga
     ```
3. (Optional) As the above hardware compile may take several hours to complete, FPGA precompiled binaries (compatible with Linux* Ubuntu* 18.04) can be downloaded <a href="https://iotdk.intel.com/fpga-precompiled-binaries/latest/kernel_args_restrict.fpga.tar.gz" download>here</a>.

### On a Windows* System

1. Generate the `Makefile` by running `cmake`.
     ```
   mkdir build
   cd build
   ```
   To compile for the Intel&reg; PAC with Intel Arria&reg; 10 GX FPGA, run `cmake` using the command:
    ```
    cmake -G "NMake Makefiles" ..
   ```
   Alternatively, to compile for the Intel&reg; FPGA PAC D5005 (with Intel Stratix&reg; 10 SX), run `cmake` using the command:
   ```
   cmake -G "NMake Makefiles" .. -DFPGA_BOARD=intel_s10sx_pac:pac_s10
   ```
   You can also compile for a custom FPGA platform. Ensure that the board support package is installed on your system. Then run `cmake` using the command:
   ```
   cmake -G "NMake Makefiles" .. -DFPGA_BOARD=<board-support-package>:<board-variant>
   ```

2. Compile the design through the generated `Makefile`. The following build targets are provided, matching the recommended development flow:

   * Compile for emulation (fast compile time, targets emulated FPGA device):
     ```
     nmake fpga_emu
     ```
   * Generate the optimization report:
     ```
     nmake report
     ```
   * Compile for FPGA hardware (longer compile time, targets FPGA device):
     ```
     nmake fpga
     ```

> **Note**: The Intel&reg; PAC with Intel Arria&reg; 10 GX FPGA and Intel&reg; FPGA PAC D5005 (with Intel Stratix&reg; 10 SX) do not support Windows*. Compiling to FPGA hardware on Windows* requires a third-party or custom Board Support Package (BSP) with Windows* support.

> **Note**: If you encounter any issues with long paths when compiling under Windows*, you may have to create your ‘build’ directory in a shorter path, for example c:\samples\build.  You can then run cmake from that directory, and provide cmake with the full path to your sample directory.

### Troubleshooting
If an error occurs, you can get more details by running `make` with
the `VERBOSE=1` argument:
``make VERBOSE=1``
For more comprehensive troubleshooting, use the Diagnostics Utility for
Intel&reg; oneAPI Toolkits, which provides system checks to find missing
dependencies and permissions errors.
[Learn more](https://software.intel.com/content/www/us/en/develop/documentation/diagnostic-utility-user-guide/top.html).

 ### In Third-Party Integrated Development Environments (IDEs)

You can compile and run this tutorial in the Eclipse* IDE (in Linux*) and the Visual Studio* IDE (in Windows*). For instructions, refer to the following link: [FPGA Workflows on Third-Party IDEs for Intel&reg; oneAPI Toolkits](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-oneapi-dpcpp-fpga-workflow-on-ide.html).

## Examining the Reports
Locate `report.html` in the `kernel_args_restrict_report.prj/reports/` directory. Open the report in Chrome*, Firefox*, Edge*, or Internet Explorer*.

Navigate to the *Loop Analysis* report (*Throughput Analysis* > *Loop Analysis*). In the *Loop List pane*, you should see two kernels: one is the kernel without the attribute applied (*KernelArgsNoRestrict*) and the other with the attribute applied (*KernelArgsRestrict*). Each kernel has a single for-loop, which appears in the *Loop List* pane. Click on the loop under each kernel to see how the compiler optimized it.

Compare the loop initiation interval (II) between the two kernels. Notice that the loop in the *KernelArgsNoRestrict* kernel has a large estimated II, while the loop in the *KernelArgsRestrict* kernel has an estimated II of ~1. These IIs are estimates because the latency of global memory accesses varies with runtime conditions.

For the *KernelArgsNoRestrict* kernel, the compiler assumed that the kernel arguments can alias each other. Since`out[i]` and `in[i+1]` could be the same memory location, the compiler cannot overlap the iteration of the loop performing `out[i] = in[i]` with the next iteration of the loop performing `out[i+1] = in[i+1]` (and likewise for iterations `in[i+2]`, `in[i+3]`, ...). This results in an II equal to the latency of the global memory read of `in[i]` plus the latency of the global memory write to `out[i]`.

We can confirm this by looking at the details of the loop. Click on the *KernelArgsNoRestrict* kernel in the *Loop List* pane and then click on the loop in the *Loop Analysis* pane. Now consider the *Details* pane below. You should see something like:

- *Compiler failed to schedule this loop with smaller II due to memory dependency*
  - *From: Load Operation (kernel_args_restrict.cpp: 74 > accessor.hpp: 945)*
  - *To: Store Operation (kernel_args_restrict.cpp: 74)*
- *Most critical loop feedback path during scheduling:*
  - *144.00 clock cycles Load Operation (kernel_args_restrict.cpp: 74 > accessor.hpp: 945)*
  - *42.00 clock cycles Store Operation (kernel_args_restrict.cpp: 74)*

The first bullet (and its sub-bullets) tells you that a memory dependency exists between the load and store operations in the loop. This is the conservative pointer aliasing memory dependency described earlier. The second bullet shows you the estimated latencies for the load and store operations (note that these are board-dependent). The sum of these two latencies (plus 1) is the II of the loop.

Next, look at the loop details of the *KernelArgsRestrict* kernel. You will notice that the *Details* pane doesn't show a memory dependency. The usage of the `[[intel::kernel_args_restrict]]` attribute allowed the compiler to schedule a new iteration of the for-loop every cycle since it knows that accesses to `in` and `out` will never alias.


## Running the Sample

 1. Run the sample on the FPGA emulator (the kernel executes on the CPU):
     ```
     ./kernel_args_restrict.fpga_emu     (Linux)
     kernel_args_restrict.fpga_emu.exe   (Windows)
     ```
2. Run the sample on the FPGA device:
     ```
     ./kernel_args_restrict.fpga         (Linux)
     kernel_args_restrict.fpga.exe       (Windows)
     ```

### Example of Output
```
Kernel throughput without attribute: 8.06761 MB/s
Kernel throughput with attribute: 766.873 MB/s
PASSED
```

### Discussion of Results

The throughput observed when running the kernels with and without the `kernel_args_restrict` attribute should reflect the difference in loop II seen in the reports. The ratios will not exactly match because the loop IIs are estimates. An example ratio (compiled and run on the Intel&reg; Programmable Acceleration Card (PAC) with Intel Arria&reg; 10 GX FPGA) is shown.

|Attribute used?  | II | Kernel Throughput (MB/s)
|:--- |:--- |:---
|No  | ~187 | 8
|Yes  | ~1 | 767

> **Note**: This performance difference will be apparent only when running on FPGA hardware. The emulator, while useful for verifying functionality, will generally not reflect differences in performance.

## License
Code samples are licensed under the MIT license. See
[License.txt](https://github.com/oneapi-src/oneAPI-samples/blob/master/License.txt) for details.

Third party program Licenses can be found here: [third-party-programs.txt](https://github.com/oneapi-src/oneAPI-samples/blob/master/third-party-programs.txt).