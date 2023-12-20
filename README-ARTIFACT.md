# Introduction

This is the artifact for the SYCL-MLIR project, corresponding to the manuscript
titled *Experiences Building an MLIR-based SYCL Compiler* accepted for
presentation at CGO 2024. 

This short document describes how to build and use the artifact to build SYCL
programs with the MLIR-based compiler.

## Building the artifact

At this point, you have probably already unzipped the artifact, otherwise please
do so now. 

After unzipping, change directory to the folder this document resides in and
create a new build directory:
```sh
mkdir build
cd build
```

In the next step, use the buildbot configuration script to setup the build.

This requires all necessary prerequisities for LLVM to be installed on your
system, including `cmake` and `Ninja`. For faster linking, we also recommend the
installation and use of the `lld` linker.

For configuration, execute the following command:
```sh
python3 ../buildbot/configure.py -t Release -o . \
  --cmake-opt="-DLLVM_USE_LINKER=lld"
```

After configuration is successful, build the compiler:
```sh
ninja sycl-toolchain
```

After the build completes successfully, the compiler and SYCL runtime will be
located in the `bin` and `lib` subfolder of the build folder.

## Using the artifact to build SYCL programs

You can now use the artifact to build SYCL programs. Make sure that your PATH
environment variable includes the `bin` subfolder of the `build` folder and
`which clang++` points to that directory. 

As an example SYCL application, we can use the following SYCL code:
```c++
#include <sycl/sycl.hpp>

int main() {
  // Creating buffer of 4 elements to be used inside the kernel code
  sycl::buffer<size_t, 1> Buffer(4);

  // Creating SYCL queue
  sycl::queue Queue;

  // Size of index space for kernel
  sycl::range<1> NumOfWorkItems{Buffer.size()};

  // Submitting command group(work) to queue
  Queue.submit([&](sycl::handler &cgh) {
    // Getting write only access to the buffer on a device.
    sycl::accessor Accessor{Buffer, cgh, sycl::write_only};
    // Executing kernel
    cgh.parallel_for<class FillBuffer>(
        NumOfWorkItems, [=](sycl::id<1> WIid) {
          // Fill buffer with indexes.
          Accessor[WIid] = WIid.get(0);
        });
  });

  // Getting read only access to the buffer on the host.
  // Implicit barrier waiting for queue to complete the work.
  sycl::host_accessor HostAccessor{Buffer, sycl::read_only};

  // Check the results
  bool MismatchFound = false;
  for (size_t I = 0; I < Buffer.size(); ++I) {
    if (HostAccessor[I] != I) {
      std::cout << "The result is incorrect for element: " << I
                << " , expected: " << I << " , got: " << HostAccessor[I]
                << std::endl;
      MismatchFound = true;
    }
  }

  if (!MismatchFound) {
    std::cout << "The results are correct!" << std::endl;
  }

  return MismatchFound;
}
```

Copy this code into a file named `test-sycl.cpp`.

To compile the example SYCL code with the artifact, use the following command:

```sh
 clang++ -fsycl -fsycl-targets=spir64-unknown-unknown-syclmlir \
   -o sycl-test -w test-sycl.cpp 
```

To dump the MLIR for the SYCL application, use the following command:
```sh
clang++ -fsycl -fsycl-device-only -S -emit-mlir -w test-sycl.cpp
```

This will create a file called `test-sycl-sycl-spir64-unknown-unknown.mlir`
containing MLIR in textual format that you can inspect with any text editor. 

### Executing the compiled SYCL application

To execute the SYCL application compiled by the artifact compiler, first make
sure that the necessary low-level runtimes are installed. 

To this end, follow the instructions 
[here](https://github.com/intel/llvm/blob/sycl/sycl/doc/GetStartedGuide.md#install-low-level-runtime)
(the instructions can also be found in `sycl/doc/GetStartedGuide.md` in this
artifact).

Next, you need to make sure that the `lib` subfolder from the `build` folder is
on your `LD_LIBRARY_PATH`.

After that, you can use the `sycl-ls` command to ensure correct installation of
all device runtimes. This command should print the devices for which you
installed low-level runtimes. If no device is shown, please consult the guide
mentioned above for trouble shooting.

If a SYCL device for the LevelZero or OpenCL backend (other than FPGAs) is
successfully reported by `sycl-ls`, you can run the compiled application with
the simple command:
```sh
./sycl-test
```

