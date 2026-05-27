# mikanos-build

This repository contains the procedures and tools for building [MikanOS](https://github.com/uchan-nos/mikanos), an educational OS developed by uchan.
Compatibility has been verified on Ubuntu 18.04.

The instructions provided here assume you are somewhat familiar with Linux command-line operations.
If you are unfamiliar with Linux commands, we recommend reading [Essential Linux Commands You Should Know](https://github.com/uchan-nos/os-from-zero/wiki/Basic-Linux-Commands) first.

The MikanOS build process consists of the following four main steps:

1. Setting up the build environment
2. Obtaining the MikanOS source code
3. Building the bootloader
4. Building MikanOS

## Setting up the build environment

Gather the tools and files necessary for building the bootloader and MikanOS itself.

### Downloading the repository

First, install Git and download the mikanos-build repository.

    $ sudo apt update
    $ sudo apt install git
    $ cd $HOME
    $ git clone https://github.com/uchan-nos/mikanos-build.git osbook

The latest version of mikanos-build supports Ubuntu 22.04. However, it has been found that using the included lld-14 causes link addresses to shift, resulting in the OS or applications malfunctioning.

- [Day 03a and later: Fails to reach kernel boot · Issue #134](https://github.com/uchan-nos/os-from-zero/issues/134)
- [Page faults occur when building tview with LLVM 10 · Issue #4](https://github.com/uchan-nos/mikanos/issues/4)

If you wish to use lld-7, please use Ubuntu 18.04 or 20.04. For Ubuntu 18.04 and 20.04, please run the following command to use the older mikanos-build.

    $ cd osbook
    $ git checkout 8d4882122ec548ef680b6b5a2ae841a0fd4d07a1

### Installing Development Tools

Next, we will set up development tools such as Clang and Nasm, as well as EDK II.
The required tools are listed in `ansible_provision.yml`.
It is easier to perform the setup using Ansible.

Note: `ansible_provision.yml` sets LLVM7 as the default. This is achieved by using Ubuntu’s `alternatives` mechanism to create a symbolic link in `/usr/bin`.

    $ sudo apt install ansible
    $ cd $HOME/osbook/devenv
    $ ansible-playbook -K -i ansible_inventory ansible_provision.yml

If the setup is successful, the `iasl` command should be installed, and a directory named `$HOME/edk2` should have been created.
If these are missing, the setup has failed.

    $ iasl -v
    $ ls $HOME/edk2

When you run the above commands on Ubuntu in WSL, a setting for the `DISPLAY` environment variable may be added to `$HOME/.profile`.
To enable this setting, you need to restart the terminal or run the following command.

    $ source $HOME/.profile

### About Standard Library Licenses

Additionally, the `ansible-playbook` command mentioned above downloads pre-built standard libraries to `$HOME/osbook/devenv/x86_64-elf`.

The files in this directory are builds of [Newlib](https://sourceware.org/newlib/), [libc++](https://libcxx.llvm.org/), and [FreeType](https://www.freetype.org/).
Their licenses are governed by the respective library-specific licenses.
Please note that these differ from the licenses for MikanOS and the entire mikanos-build repository.

The following files originate from Newlib. Please refer to `x86_64-elf/LICENSE.newlib` for the license.

    x86_64-elf/lib/
        libc.a
        libg.a
        libm.a
    x86_64-elf/include/
        All except c++/

The following files originate from libc++. Please refer to `x86_64-elf/LICENSE.libcxx` for the license.

    x86_64-elf/lib/
        libc++.a
        libc++abi.a
        libc++experimental.a
    x86_64-elf/include/c++/v1/
        All

The following files originate from FreeType. Please refer to `x86_64-elf/LICENSE.freetype` for the license.

    x86_64-elf/lib/
        libfreetype.a
    x86_64-elf/include/freetype2/
        All

## Obtaining the MikanOS Source Code

You can obtain it via Git.

    $ git clone https://github.com/uchan-nos/mikanos.git

The final `git clone` command creates a `mikanos` directory in the current directory and downloads the MikanOS source code into it.

## Building the Bootloader

Link the MikanOS bootloader directory to the EDK II directory.

| :warning: Please replace the following `/path/to/mikanos` with a string appropriate for your environment.                                                              |
| :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| If you are following the instructions in “Introduction to Building Your Own OS from Scratch,” the command should be `ln -s $HOME/workspace/mikanos/MikanLoaderPkg ./`. |

    $ cd $HOME/edk2
    $ ln -s /path/to/mikanos/MikanLoaderPkg ./

Specify the path to the `mikanos` directory you downloaded earlier with `git clone` for `/path/to/mikanos`.
If you can see the bootloader source code correctly, the link was successful.

    $ ls MikanLoaderPkg/Main.c

Next, run `edksetup.sh` to set the environment variables required for building EDK II.

    $ source edksetup.sh

When you run the `edksetup.sh` file, environment variables are set, and `Conf/target.txt` is automatically generated.
Open this file in an editor and modify the following entries.

| Setting         | Value                             |
| --------------- | --------------------------------- |
| ACTIVE_PLATFORM | MikanLoaderPkg/MikanLoaderPkg.dsc |
| TARGET          | DEBUG                             |
| TARGET_ARCH     | X64                               |
| TOOL_CHAIN_TAG  | CLANG38                           |

Once the configuration is complete, build the bootloader.

    $ build

- If you encounter the error “ModuleNotFoundError: No module named ‘distutils.util’”, running `sudo apt install python3-distutils` and then running `build` again may resolve the issue. Please try this.
- If the build fails with the error “Instance of library class [RegisterFilterLib] is not found,” see [MikanLoaderPkg cannot be built due to [RegisterFilterLib]-related errors](https://github.com/uchan-nos/os-from-zero/blob/main/faq.md#registerfilterlib-%E9%96%A2%E4%BF%82% E3%81%AE%E3%82%A8%E3%83%A9%E3%83%BC% E3%81%A7-mikanloaderpkg-%E3%81%8C%E3%83%93%E3%83%AB%E3%83%89%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%9B%E3%82%93).

If the Loader.efi file is generated, the build was successful.

    $ ls Build/MikanLoaderX64/DEBUG_CLANG38/X64/Loader.efi

## Building MikanOS

Load the environment variables required for the build.

    $ source $HOME/osbook/devenv/buildenv.sh

Perform the build.

    $ cd /path/to/mikanos
    $ ./build.sh

To run it in QEMU, specify the `run` option with `./build.sh`.

    $ ./build.sh run

To place apps in the `apps` directory and create a disk image that includes resources such as fonts, specify the `APPS_DIR` and `RESOURCE_DIR` variables.

    $ APPS_DIR=apps RESOURCE_DIR=resource ./build.sh run
