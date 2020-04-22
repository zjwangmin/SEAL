# Microsoft SEAL

Microsoft SEAL is an easy-to-use open-source ([MIT licensed](LICENSE)) homomorphic encryption library developed by the Cryptography and Privacy Research group at Microsoft.
Microsoft SEAL is written in modern standard C++ and is easy to compile and run in many different environments.
For more information about the Microsoft SEAL project, see [sealcrypto.org](https://www.microsoft.com/en-us/research/project/microsoft-seal).

This document pertains to Microsoft SEAL version 3.5.
Users of previous versions of the library should look at the [list of changes](Changes.md).

## Contents

- [Introduction](#introduction)
  - [Core Concepts](#core-concepts)
  - [Homomorphic Encryption](#homomorphic-encryption)
  - [Microsoft SEAL](#microsoft-seal-1)
- [Installing Microsoft SEAL](#installing-microsoft-seal)
  - [Optional Dependencies](#optional-dependencies)
  - [Windows](#windows)
  - [Linux and macOS](#linux-and-macos)
  - [From NuGet package](#from-nuget-package)
- [Building Microsoft SEAL for .NET](#building-microsoft-seal-for-net)
  - [Windows](#windows-1)
  - [Linux and macOS](#linux-and-macos-1)
- [Getting Started](#getting-started)
- [Contributing](#contributing)
- [Citing Microsoft SEAL](#citing-microsoft-seal)

## Introduction

### Core Concepts

Most encryption schemes consist of three functionalities: key generation, encryption, and decryption.
Symmetric-key encryption schemes use the same secret key for both encryption and decryption; public-key encryption schemes use separately a public key for encryption and a secret key for decryption.
Therefore, public-key encryption schemes allow anyone who knows the public key to encrypt data, but only those who know the secret key can decrypt and read the data.
Symmetric-key encryption can be used for efficiently encrypting very large amounts of data, and enables secure outsourced cloud storage.
Public-key encryption is a fundamental concept that enables secure online communication today, but is typically much less efficient than symmetric-key encryption.

While traditional symmetric- and public-key encryption can be used for secure storage and communication, any outsourced computation will necessarily require such encryption layers to be removed before computation can take place.
Therefore, cloud services providing outsourced computation capabilities must have access to the secret keys, and implement access policies to prevent unauthorized employees from getting access to these keys.

### Homomorphic Encryption

Homomorphic encryption refers to encryption schemes that allow the cloud to compute directly on the encrypted data, without requiring the data to be decrypted first.
The results of such encrypted computations remain encrypted, and can be only decrypted with the secret key (by the data owner).
Multiple homomorphic encryption schemes with different capabilities and trade-offs have been invented over the past decade; most of these are public-key encryption schemes, although the public-key functionality may not always be needed.

Homomorphic encryption is not a generic technology: only some computations on encrypted data are possible.
It also comes with a substantial performance overhead, so computations that are already very costly to perform on unencrypted data are likely to be infeasible on encrypted data.
Moreover, data encrypted with homomorphic encryption is many times larger than unencrypted data, so it may not make sense to encrypt, e.g., entire large databases, with this technology.
Instead, meaningful use-cases are in scenarios where strict privacy requirements prohibit unencrypted cloud computation altogether, but the computations themselves are fairly lightweight.

Typically, homomorphic encryption schemes have a single secret key which is held by the data owner.
For scenarios where multiple different private data owners wish to engage in collaborative computation, homomorphic encryption is probably not a reasonable solution.

Homomorphic encryption cannot be used to enable data scientist to circumvent GDPR.
For example, there is no way for a cloud service to use homomorphic encryption to draw insights from encrypted customer data.
Instead, results of encrypted computations remain encrypted and can only be decrypted by the owner of the data, e.g., a cloud service customer.

### Microsoft SEAL

Microsoft SEAL is a homomorphic encryption library that allows additions and multiplications to be performed on encrypted integers or real numbers.
Other operations, such as encrypted comparison, sorting, or regular expressions, are in most cases not feasible to evaluate on encrypted data using this technology.
Therefore, only specific privacy-critical cloud computation parts of programs should be implemented with Microsoft SEAL.

It is not always easy or straightfoward to translate an unencrypted computation into a computation on encrypted data, for example, it is not possible to branch on encrypted data.
Microsoft SEAL itself has a steep learning curve and requires the user to understand many homomorphic encryption specific concepts, even though in the end the API is not too complicated.
Even if a user is able to program and run a specific computation using Microsoft SEAL, the difference between efficient and inefficient implementations can be several orders of magnitude, and it can be hard for new users to know how to improve the performance of their computation.

Microsoft SEAL comes with two different homomorphic encryption schemes with very different properties.
The BFV scheme allows modular arithmetic to be performed on encrypted integers.
The CKKS scheme allows additions and multiplications on encrypted real or complex numbers, but yields only approximate results.
In applications such as summing up encrypted real numbers, evaluating machine learning models on encrypted data, or computing distances of encrypted locations CKKS is going to be by far the best choice.
For applications where exact values are necessary, the BFV scheme is the only choice.

# Installing Microsoft SEAL

## Optional Dependencies

Microsoft SEAL has no required dependencies, but certain optional features can be enabled if it is compiled with support for specific third-party libraries.

### Microsoft GSL

Microsoft GSL (Guidelines Support Library) is a header-only library that implements `gsl::span`: a *view type* that provide safe (bounds-checked) array access to memory.
For example, if Microsoft GSL is available, Microsoft SEAL can allow `BatchEncoder` and `CKKSEncoder` to encode from and decode to a `gsl::span` instead of `std::vector`, which can in some cases have a significant performance benefit.

### ZLIB

ZLIB is a widely used compression library that implements the DEFLATE compression algorithm.
Microsoft SEAL can use ZLIB (if present) to automatically compress data that is serialized.
`Ciphertext` objects consist of a large number of integers modulo specific prime numbers (`coeff_modulus` primes).
When using the CKKS scheme, although these prime numbers can often be quite small (e.g., 30 bits), the numbers are nevertheless serialized as 64-bit integers.
In this case, more than half of data in a ciphertext are zeros that can be compressed away with a compression library, such as ZLIB.
The BFV scheme benefits typically less from this technique, because the prime numbers used for the `coeff_modulus` encryption parameter tend to be larger, and integers modulo these prime numbers fill more of each 64-bit word.
The compression is not only applied to `Ciphertext` objects, but to every serializable Microsoft SEAL object.

If ZLIB is available, it will be automatically used for serialization (see `Serialization::compr_mode_default` in `native/src/seal/serialization.h`.
However, it is always possible to explicitly pass `compr_mode_type::none` to serialization methods to disable compression.

**WARNING:** The compression rate for a `SecretKey` can (in theory at least) reveal information about the key.
In most common applications of Microsoft SEAL the size of a `SecretKey` would not be deliberately revealed to untrusted parties.
If this is a concern, one can always save the `SecretKey` in an uncompressed form by passing `compr_mode_type::none` to `SecretKey::save`.

## Windows

Microsoft SEAL comes with a Microsoft Visual Studio 2019 solution file `SEAL.sln` that can be used to conveniently build the library, examples, and unit tests.
Visual Studio 2019 is required to build Microsoft SEAL.

#### Platform

The Visual Studio solution `SEAL.sln` is configured to build Microsoft SEAL both for `Win32` and `x64` platforms. Please choose the right platform before building Microsoft SEAL.
The `SEAL_C` project and the .NET wrapper library `SEALNet` can only be built for `x64`.

#### Building Microsoft SEAL

Build the SEAL project `native\src\SEAL.vcxproj` from `SEAL.sln`.
This results in the static library `seal.lib` to be created in `lib\$(Platform)\$(Configuration)`.
When linking with applications, you need to add `native\src\` (full path) as an include directory for Microsoft SEAL header files.

#### [Optional] Debug and Release builds

You can easily switch from Visual Studio build configuration menu whether Microsoft SEAL should be built in `Debug` mode (no optimizations) or in `Release` mode.
Please note that `Debug` mode should not be used except for debugging Microsoft SEAL itself, as the performance will be orders of magnitude worse than in `Release` mode.

#### [Optional] Microsoft GSL

To build Microsoft SEAL with support for Microsoft GSL, clone first the Microsoft GSL library from [GitHub.com/Microsoft/GSL](https://GitHub.com/Microsoft/GSL) to some convenient directory, e.g., `C:\MyLibs\GSL` in this example.

Next, you will need to signal Microsoft SEAL to enable Microsoft GSL support by creating a new Windows environment variable `MSGSL_ROOT`, and setting its value to `C:\MyLibs\GSL\include`.
Restart Visual Studio at this point if you had it open, otherwise it will not have captured the newly created environment variable.
Rebuilding Microsoft SEAL should now automatically detect that Microsoft GSL is available, and enable both `gsl::span` and `gsl::multi_span` support.
To disable Microsoft GSL support, delete the `MSGSL_ROOT` environment variable, restart Visual Studio, and rebuild Microsoft SEAL.

If Microsoft SEAL is built with Microsoft GSL support, any programs or libraries consuming Microsoft SEAL will need access to the Microsoft GSL header files.
You need to add `$(MSGSL_ROOT)` to *Additional Include Directories* under the *C/C++* tab in your Visual Studio project properties.
Note that in the Microsoft SEAL projects this has already been set for you, so all projects in `SEAL.sln` should work without change.

#### [Optional] ZLIB

TODO: Is this part still accurate now?
ZLIB is usually not found on a typical Windows system.
You can clone it from [GitHub.com/madler/zlib](https://github.com/madler/zlib) to some convenient directory, e.g., `C:\MyLibs\zlib` in this example.
You need to build ZLIB first by opening *Developer Command Prompt for VS 2019*, go to `C:\MyLibs\zlib`, and run the following:

```shell
cmake .
cmake --build . --config Release
```

Next, you will need to signal Microsoft SEAL to enable ZLIB support by creating a new Windows environment variable `ZLIB_ROOT`, and setting its value to `C:\MyLibs\zlib`.
Restart Visual Studio at this point if you had it open, otherwise it will not have captured the newly created environment variable.
Rebuilding Microsoft SEAL should now automatically detect that ZLIB is available, and enable support for `compr_mode_type::deflate`.
To disable ZLIB support, delete the `ZLIB_ROOT` environment variable, restart Visual Studio, and rebuild Microsoft SEAL.

#### Building Examples

Build the SEALExamples project `native\examples\SEALExamples.vcxproj` from `SEAL.sln`.
This results in an executable `sealexamples.exe` to be created in `bin\$(Platform)\$(Configuration)`.

#### Building Unit Tests

The unit tests require the Google Test framework to be installed.
The appropriate NuGet package is already listed in `native\tests\packages.config`, so once you attempt to build the SEALTest project `native\tests\SEALTest.vcxproj` from `SEAL.sln` Visual Studio will automatically download and install it for you.

## Linux and macOS

Microsoft SEAL is very easy to configure and build in Linux and macOS using CMake (>= 3.12).
A modern version of GNU G++ (>= 6.0) or Clang++ (>= 5.0) is needed.
In macOS the Xcode toolchain (>= 9.3) will work.

In macOS you will need CMake with command line tools. For this, you can either

1. install the cmake package with [Homebrew](https://brew.sh), or
1. download CMake directly from [cmake.org/download](https://cmake.org/download) and
[enable command line tools](https://stackoverflow.com/questions/30668601/installing-cmake-command-line-tools-on-a-mac).

Below we give instructions for how to configure, build, and install Microsoft SEAL either system-wide (global install), or for a single user (local install).
A system-wide install requires elevated (root) privileges.

#### Building Microsoft SEAL

We assume that Microsoft SEAL has been cloned into a directory called `SEAL` and all commands presented below are assumed to be executed in the directory `SEAL`.

You can build Microsoft SEAL library for your machine by executing the following commands:

```shell
cmake .
make
```

#### [Optional] Debug and Release Modes

By default Microsoft SEAL is built in `Release` mode.
You can easily switch between `Debug` mode (no optimizations) or `Release` mode in CMake configuration options as follows:

```shell
cmake . -DCMAKE_BUILD_TYPE=Debug
make
```

Please note that `Debug` mode should not be used except for debugging Microsoft SEAL itself, as the performance will be orders of magnitude worse than in `Release` mode.

#### [Optional] Microsoft GSL

By default Microsoft GSL is downloaded as part of Microsoft SEAL library.
Microsoft GSL's header files are copied to `native/src/GSL` to be portable with Microsoft SEAL at the time of installation.
You can disable the dependency on Microsoft GSL in CMake configuration options as follows:

```shell
cmake . -DSEAL_USE_MSGSL=OFF
make
```

#### [Optional] ZLIB

By default ZLIB is downloaded and compiled as part of Microsoft SEAL library.
ZLIB's static archive is included in Microsoft SEAL's static or shared target object.
You can disable the dependency on ZLIB in CMake configuration options as follows:

```shell
cmake . -DSEAL_USE_ZLIB=OFF
make
```

#### [Optional] Shared Library

TODO: verify macOS shared lib names/links.

By default Microsoft SEAL builds only a static library that is `libseal-3.5.a` on Unix-like platforms.
You can enable building a shared library, `libseal.so*` in Linux or `libseal*.dylib` in macOS, in CMake configuration options as follows:

```shell
cmake . -DBUILD_SHARED_LIBS=ON
make
```

#### Building Examples

By default Microsoft SEAL does not build examples. You can enable building examples in CMake configuration options as follows:

```shell
cmake . -DSEAL_BUILD_EXAMPLES=ON
make
```

The `sealexamples` executable is located in `native/bin/`.

#### Building Unit Tests

By default Microsoft SEAL does not build tests.
You can enable building tests in CMake configuration options as follows:

```shell
cmake . -DSEAL_BUILD_TESTS=ON
make
```

This downloads and compiles the [GoogleTest](https://github.com/google/googletest) framework as a part of Microsoft SEAL.
The `sealtest` executable is located `native/bin/`.
All unit tests should pass successfully.

### Installing Microsoft SEAL

If you have root access to the system you can install Microsoft SEAL system-wide as follows:

```shell
cmake .
make
sudo make install
```

To instead install Microsoft SEAL locally, e.g., to `~/mylibs/`, do the following:

```shell
cmake . -DCMAKE_INSTALL_PREFIX=~/mylibs
make
make install
```

#### Linking with Microsoft SEAL through CMake

It is very easy to link your own applications and libraries with Microsoft SEAL if you use CMake.
Simply add the following to your `CMakeLists.txt`:

```shell
find_package(SEAL 3.5 REQUIRED)
target_link_libraries(<your target> SEAL::seal)
```

If Microsoft SEAL was installed globally, the above `find_package` command will likely find the library automatically.
To link with a locally installed Microsoft SEAL, e.g., installed in `~/mylibs` as described above, you may need to tell CMake where to look for Microsoft SEAL when you configure your application by running:

```shell
cd <directory containing your CMakeLists.txt>
cmake . -DCMAKE_PREFIX_PATH=~/mylibs
```

## From NuGet package

For .NET developers the easiest way of installing Microsoft SEAL is by using the multi-platform NuGet package available at [NuGet.org](https://www.nuget.org/packages/Microsoft.Research.SEALNet).
Simply add this package into your .NET project as a dependency and you are ready to go.

# Building Microsoft SEAL for .NET

TODO: Verify library locations in Windows.
TODO: Verify library .NET standard versions in Windows.

Microsoft SEAL provides a .NET Standard library that wraps the functionality in Microsoft SEAL for use in .NET development.

## Windows

The Microsoft Visual Studio 2019 solution file `SEAL.sln` contains the projects necessary to build the .NET assembly, a backing native shared library, .NET examples, and unit tests.

#### Native Library

Microsoft SEAL for .NET requires a native library that is invoked by the managed .NET library.
Build the SEAL_C project `native\src\SEAL_C_.vcxproj` from `SEAL.sln`.
Building SEAL_C results in the dynamic library `sealc.dll` to be created in `lib\$(Platform)\$(Configuration)`.
This library is meant to be used only by the .NET library, not by end users, and needs to be present in the same directory as your executable when running a .NET application.

#### .NET Library

Once you have built the shared native library (see above), build the SEALNet project `dotnet\src\SEALNet.csproj` from `SEAL.sln`.
Building SEALNet results in the assembly `SEALNet.dll` to be created in `lib\dotnet\$(Configuration)\netstandard2.1`.
This is the assembly you can reference in your application.

#### .NET Examples

Build the SEALNetExamples project `dotnet\examples\SEALNetExamples.csproj` from `SEAL.sln`.
This results in the assembly `SEALNetExamples.dll` to be created in `bin\dotnet\$(Configuration)\netcoreapp2.1`.
The project takes care of copying the native SEAL_C library to the output directory.

#### .NET Unit Tests

Build the SEALNet Test project `dotnet\tests\SEALNetTest.csproj` from `SEAL.sln`.
This results in the `SEALNetTest.dll` assembly to be created in `lib\dotnet\$(Configuration)\netcoreapp2.1`.
The project takes care of copying the native SEALNetNative library to the output directory.

#### Using Microsoft SEAL for .NET in Your Own Application

To use Microsoft SEAL for .NET in your own application you need to:

1. add a reference in your project to `SEALNet.dll`;
1. ensure `sealc.dll` is available for your application when run.
The easiest way to ensure this is to copy `sealc.dll` to the same directory where your application's executable is located.

#### Building Your Own NuGet Package

You can build your own NuGet package for Microsoft SEAL by following the instructions in [NUGET.md](dotnet/nuget/NUGET.md).

## Linux and macOS

Microsoft SEAL for .NET relies on a native shared library that can be easily configured and built using CMake (>= 3.12) and a modern version of GNU G++ (>= 6.0) or Clang++ (>= 5.0).
In macOS the Xcode toolchain (>= 9.3) will work.

For compiling .NET code you will need to install a .NET Core SDK (>= 2.1).
You can follow these [instructions for installing in Linux](https://dotnet.microsoft.com/download?initial-os=linux), or for [installing in macOS](https://dotnet.microsoft.com/download?initial-os=macos).

#### Native Library

If you only intend to run the examples and unit tests provided with Microsoft SEAL, you do not need to install the native shared library, you only need to compile it.
The SEALNetExamples and SEALNetTest projects take care of copying the native shared library to the appropriate assembly output directory.

Microsoft SEAL by default does not build SEAL_C.
You can enable it in CMake configuration options as follows:

```shell
cmake . -DSEAL_BUILD_SEAL_C=ON
make
```

This results in a shared native library `libsealc.so*` in Linux or `libsealc*.dylib` in macOS.

If you have root access to the system, you have the option to install the native shared library globally.
Then your application will always be able to find and load it.
Assuming Microsoft SEAL is build and installed globally, you can install the shared native library globally as follows:

```shell
sudo make install
```

#### .NET Library

To build the .NET Standard library, do the following:

```shell
dotnet build dotnet/src --configuration <Debug|Release>
```

This will result in a `SEALNet.dll` assembly to be created in `lib/dotnet/$(Configuration)/netstandard2.1`.
This assembly is the one you will want to reference in your own projects.
The optional `dotnet` parameter `--configuration <Debug|Release>` can be used to build either a `Debug` or `Release` version of the assembly.

#### .NET Examples

To build and run the .NET examples, do:

```shell
dotnet run -p dotnet/examples
```

As mentioned before, the .NET project will copy the shared native library to the assembly output directory.
You can use the `dotnet` parameter `--configuration <Debug|Release>` to run either `Debug` or `Release` versions of the examples.

#### .NET Unit Tests

To build and run the .NET unit tests, do:

```shell
dotnet test dotnet/tests
```

All unit tests should pass.
You can use the `dotnet` parameter `--configuration <Debug|Release>` to run `Debug` or `Relase` unit tests.
And you can use `--verbosity detailed` to print the list of unit tests that are being run.

#### Using Microsoft SEAL for .NET in Your Own Application

To use Microsoft SEAL for .NET in your own application you need to:

1. add a reference in your project to `SEALNet.dll`;
1. ensure the native shared library is available for your application when run.
The easiest way to ensure this is to copy the native shared library to the same directory where your application's executable is located.

# Getting Started

Using Microsoft SEAL will require the user to invest some time in learning fundamental concepts in homomorphic encryption.
The code comes with heavily commented examples that are designed to gradually teach such concepts as well as to demonstrate much of the API.
The code examples are available (and identical) in C++ and C#, and are divided into several source files in `native/examples/` (C++) and `dotnet/examples/` (C#), as follows:

|C++                  |C#                  |Description                                                                 |
|---------------------|--------------------|----------------------------------------------------------------------------|
|`examples.cpp`       |`Examples.cs`       |The example runner application                                              |
|`1_bfv_basics.cpp`   |`1_BFV_Basics.cs`   |Encrypted modular arithmetic using the BFV scheme                           |
|`2_encoders.cpp`     |`2_Encoders.cs`     |Encoding more complex data into Microsoft SEAL plaintext objects            |
|`3_levels.cpp`       |`3_Levels.cs`       |Introduces the concept of levels; prerequisite for using the CKKS scheme    |
|`4_ckks_basics.cpp`  |`4_CKKS_Basics.cs`  |Encrypted real number arithmetic using the CKKS scheme                      |
|`5_rotation.cpp`     |`5_Rotation.cs`     |Performing cyclic rotations on encrypted vectors in the BFV and CKKS schemes|
|`6_serialization.cpp`|`6_Serialization.cs`|Serializing data objects in Microsoft SEAL                                  |
|`7_performance.cpp`  |`7_Performance.cs`  |Performance tests for Microsoft SEAL                                        |

It is recommeded to read the comments and the code snippets along with command line printout from running an example.
For easier navigation, command line printout provides the line number in the associated source file where the associated code snippets start.

**WARNING: It is impossible to use Microsoft SEAL correctly without reading all examples or by simply re-using the code from examples.
Any developer attempting to do so will inevitably produce code that is *vulnerable*, *malfunctioning*, or *extremely slow*.**

# Contributing

For contributing to Microsoft SEAL, please see [CONTRIBUTING.md](CONTRIBUTING.md).

# Citing Microsoft SEAL

To cite Microsoft SEAL in academic papers, please use the following BibTeX entries.

### Version 3.5

    @misc{sealcrypto,
        title = {{M}icrosoft {SEAL} (release 3.5)},
        howpublished = {\url{https://github.com/Microsoft/SEAL}},
        month = may,
        year = 2020,
        note = {Microsoft Research, Redmond, WA.},
        key = {SEAL}
    }

### Version 3.4

    @misc{sealcrypto,
        title = {{M}icrosoft {SEAL} (release 3.4)},
        howpublished = {\url{https://github.com/Microsoft/SEAL}},
        month = oct,
        year = 2019,
        note = {Microsoft Research, Redmond, WA.},
        key = {SEAL}
    }

### Version 3.3

    @misc{sealcrypto,
        title = {{M}icrosoft {SEAL} (release 3.3)},
        howpublished = {\url{https://github.com/Microsoft/SEAL}},
        month = june,
        year = 2019,
        note = {Microsoft Research, Redmond, WA.},
        key = {SEAL}
    }

### Version 3.2

    @misc{sealcrypto,
        title = {{M}icrosoft {SEAL} (release 3.2)},
        howpublished = {\url{https://github.com/Microsoft/SEAL}},
        month = feb,
        year = 2019,
        note = {Microsoft Research, Redmond, WA.},
        key = {SEAL}
    }

### Version 3.1

    @misc{sealcrypto,
        title = {{M}icrosoft {SEAL} (release 3.1)},
        howpublished = {\url{https://github.com/Microsoft/SEAL}},
        month = dec,
        year = 2018,
        note = {Microsoft Research, Redmond, WA.},
        key = {SEAL}
    }

### Version 3.0

    @misc{sealcrypto,
        title = {{M}icrosoft {SEAL} (release 3.0)},
        howpublished = {\url{http://sealcrypto.org}},
        month = oct,
        year = 2018,
        note = {Microsoft Research, Redmond, WA.},
        key = {SEAL}
    }
