Building XNU for macOS Mojave 10.14.1

The macOS Mojave 10.14.1 kernel (XNU) source has been released here:  source ,  tarball .

Building XNU requires some patience, and some open source dependencies which are not pre-installed. This post walks through all the steps necessary to run your own custom-built XNU on supported Apple hardware. A VM is recommended, but the following steps will build a kernel suitable for both a VM and bare metal.
TL;DR

There is a Makefile which automates the downloading and building of all prerequisites. You can find the Makefile  here , and invoke it like:

$ make -f Makefile.xnudeps xnudeps

NEW: this Makefile will now automatically detect the correct versions of source code to download based on the version of macOS you specify. By default, the version is 10.14.1, however you can select a different version like:

$ make -f Makefile.xnudeps macos_version=10.13.1 xnudeps

You can also see other features of the Makefile using the help target. Note that full 10.13.x compilation support will be coming soon.
Manual XNU Building

All of the source for both XNU and required dependencies is available from  opensource.apple.com . Here are the manual steps necessary to build XNU:

    Download and Install Xcdoe
        Make sure you have Xcode 10 (or 10.1) installed. You can install it via the App Store, or by manual download here:  https://developer.apple.com/download/more/
        NOTE: for older versions of macOS, you may need older versions of Xcode
    Download the source
        export TARBALLS=https://opensource.apple.com/tarballs
        curl -O ${TARBALLS}/dtrace/dtrace-284.200.15.tar.gz
        curl -O ${TARBALLS}/AvailabilityVersions/AvailabilityVersions-33.200.4.tar.gz
        curl -O ${TARBALLS}/libplatform/libplatform-177.200.16.tar.gz
        curl -O ${TARBALLS}/libdispatch/libdispatch-1008.220.2.tar.gz
        curl -O ${TARBALLS}/xnu/xnu-4903.221.2.tar.gz
    Build CTF tools from dtrace
        tar zxf dtrace-284.200.15.tar.gz
        cd dtrace- 284.200.15
        mkdir obj sym dst
        xcodebuild install -sdk macosx  -target ctfconvert -target ctfdump \ -target ctfmerge ARCHS=x86_64 SRCROOT=$PWD OBJROOT=$PWD/obj \

        SYMROOT=$PWD/sym DSTROOT=$PWD/dst \

        HEADER_SEARCH_PATHS="$PWD/compat/opensolaris/** $PWD/lib/**"

        sudo ditto \

        $PWD/dst/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain \

        /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain
        cd ..
    Install AvailabilityVersions
        tar zxf AvailabilityVersions-33.200.4.tar.gz
        cd AvailabilityVersions- 33.200.4
        mkdir dst
        make install SRCROOT=$PWD DSTROOT=$PWD/dst

        sudo ditto \

        $PWD/dst/usr/local/libexec \

        $(xcrun -sdk macosx -show-sdk-path)/usr/local/libexec
        cd ..
    Install libplatform headers
        tar zxf  libplatform-177.200.16.tar.gz
        cd libplatform- 177.200.16

        sudo mkdir -p \

        $(xcrun -sdk macosx -show-sdk-path)/usr/local/include/os/internal

        sudo ditto $PWD/private/os/internal \

        $(xcrun -sdk macosx -show-sdk-path)/usr/local/include/os/internal
        cd ..
    Install XNU headers
        tar zxf  xnu-4903.221.2.tar.gz
        cd  xnu- 4903.221.2
        make SDKROOT=macosx ARCH_CONFIGS=X86_64 installhdrs
        sudo ditto $PWD/BUILD/dst $(xcrun -sdk macosx -show-sdk-path)
        cd ..
    Build firehose from libdispatch
        tar zxf  libdispatch-1008.220.2.tar.gz
        cd  libdispatch- 1008.220.2
        mkdir obj sym dst

        awk '/include "<DEVELOPER/ {next;} /SDKROOT =/ {print "SDKROOT = macosx"; next;} {print $0}' \

        xcodeconfig/libdispatch.xcconfig > .__tmp__ && \

        mv -f .__tmp__ 
        xcodeconfig/libdispatch.xcconfig
        awk '/#include / { next; } { print $0 }' \

        xcodeconfig/libfirehose_kernel.xcconfig > .__tmp__ && \

        mv -f .__tmp__ 
        xcodeconfig/libfirehose_kernel.xcconfig

        xcodebuild install -sdk macosx -target libfirehose_kernel \

        SRCROOT=$PWD OBJROOT=$PWD/obj 
        SYMROOT=$PWD/sym DSTROOT=$PWD/dst

        sudo ditto $PWD/dst/usr/local \

        $(xcrun -sdk macosx -show-sdk-path)/usr/local
        cd ..
    Build XNU  (checkout the README.md for more options!)
        cd  xnu- 4903.221.2
        make SDKROOT=macosx ARCH_CONFIGS=X86_64 KERNEL_CONFIGS=RELEASE

Check out the README.md file at the top of the XNU source tree for more options to the build system. Some common and useful options include:  KERNEL_CONFIGS=DEVELOPMENT ,  BUILD_LTO=n and  LOGCOLORS=y .
Install and Run XNU

WARNING: Unfortunately, the open source Mojave 10.14.1 kernel does not include all symbols necessary to successfully create a prelinkedkernel image. This means that open source XNU will not boot, even in a VM.

After the final build step, you should have a new kernel built in  $PWD/BUILD/obj/kernel . In order to run this kernel, you will need to install it, and rebuild the prelinkedkernel image. Installing a kernel could potentially render your system un-bootable, so trying this out in a VM first is recommended. To install and run your kernel:

    cd  xnu- 4903.221.2
    sudo ditto $PWD/BUILD/obj/kernel /System/Library/Kernels/kernel

    sudo kextcache -v -invalidate /

    / locked; waiting for lock.

    Lock acquired; proceeding

    ...

    sudo reboot

    ...
    uname -a

If you build a different variant of XNU, you may need to ditto a different kernel name, e.g.,  kernel.development instead of just  kernel .

Note that you can select different prelinkedkernel variants from which to boot using the kcsuffix boot-arg. For example, if you built a development kernel (using KERNEL_CONFIGS=DEVELOPMENT in the make invocation), you would install and run it like so:

    sudo ditto $PWD/BUILD/obj/kernel.development \

    /System/Library/Kernels/kernel.development
    sudo kextcache -v -invalidate /
    sudo nvram boot-args="kcsuffix=development"
    sudo reboot

If you have existing boot-args, you can, of course, preserve them in the nvram boot-args variable.

https://hk.saowen.com/a/6c0e7317603ea6223bef7e57a1d817a09a0660101bfb88876f76ba6d4a5c9631
