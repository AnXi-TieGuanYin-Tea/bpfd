# Install and runn BCC tools and BPFd from packages.

If you only care about running/adding/modifying new BCC tools and don't care about changing the core infrastructure of
BCC/clang/BPFd code, download the [`bpfd-full.deb`](http://bit.do/bpfd-full-dl) file. This package contains BCC
tools, LLVM libraries and BPFd ready to go. All source code for these binaries are available in Joel's github repositories.

Note that the .deb package is only for Ubuntu 64-bit x86 machines.

Once downloaded, run:
```
dpkg -i bcc-full.deb
```
This will install all the binaries into an isolated `/usr/local/bpfd-install/` directory. With the exception being a
`/usr/bin/bcc-init` required to start the BCC environment sandbox.

Start the sandbox by running following as root:
```
sudo bcc-init
```
This gets you a shell prompt that looks like this:
```
(bcc) root@hostname:~/
```
Next setup your target environment based on your needs. For this the `bcc-set` command convinently helps.
The following settings are mandatory unless you're planning to run BCC locally.

Setup the path to the kernel source directory. Make sure the kernel build has completed atleast once in the source
directory, and take note of the path to it.
```
bcc-set --kernelsrc /path/to/kernel-sources/
```

Setup the architecture of your target:
```
bcc-set --arch "arm64"
```
Setup the remote communication method, for Android devices you'd use 'adb'. In the future other networking protocols may
be trivially added.
```
bcc-set --remote "adb"
```
To check all settings made so far, you can run:
```
bcc-set --print
```
You can set the remote to "process" if you just want to run bpfd locally in a forked "process", which is probably only
good for some local BPFd testing.

Next in order to run the tool, you'll need to push BPFd binary to your target device. This really depends on the type of
remote you're using. For the `adb` remote, `bpfd` binary is expected at `/data/bpfd`. For sake of convenience, an arm64
binary comes with the .deb package. Run the following to copy it to over to your arm64 Android device if that's what
you're testing on:
```
adb push /usr/local/bpfd-install/share/bpfd.arm64.static /data/bpfd
```
You should be all set. All BCC tools are available in the $PATH and ready to run from your sandbox. Try simple tools
like `opensnoop` or `filetop` to make sure its working. For debugging, run the following commands before running the
tools. This will show you flags passed to the kernel build, path to the kernel, messages communicated to BPFd, etc.
```
bcc-set --debug
```
To stop debugging, run:
```
bcc-set --nodebug
```

# (Alternately, IF you want to) Build and running everything yourself
This is an example of a typical build and installation procedure, it should be fairly straight forward to get these steps
working for other remotes or architectures. For this example, we'll refer to the machine where you do all your
development and have your kernel sources available as the `development machine` and the machine you're tracing as the
`target`.

#### Build LLVM on your development host
```
git clone http://llvm.org/git/llvm.git
cd llvm/tools; git clone http://llvm.org/git/clang.git
cd ..; mkdir -p build/install; cd build
cmake -G "Unix Makefiles" -DLLVM_TARGETS_TO_BUILD="BPF;X86" \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PWD/install ..
make
make install
export PATH=$PWD/install/bin:$PATH
```
LLVM's libraries are needed to run BCC tools. Add the last line above to your `.bashrc` to keep it persistent.

#### Build BCC tools on your development host
These steps were executed on Ubuntu distro. If they don't work for your distro, [check BCC project's
INSTALL.md](https://github.com/iovisor/bcc/blob/master/INSTALL.md) for other instructions.
```
git clone git@github.com:joelagnel/bcc.git
cd bcc
git checkout -b bcc-bpfd origin/bcc-bpfd
mkdir -p build; cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr ..
make
make install
```

#### Build/Install BPFd for your target machine
Admittedly, the build process for BPFd is quite simple and naive. Patches for a more configurable build/install process
are welcome.

Clone and build the BPFd sources:
```
git clone git@github.com:joelagnel/bpfd.git; cd bpfd
make
```
To build for arm64:
```
git clone git@github.com:joelagnel/bpfd.git; cd bpfd
cp Makefile.arm64 Makefile
make
```
The built binaries (`bpfd` and `libbpf_bpfd.so`) are available in the build directory. Incase of errors, check that the
compiler paths in the Makefile are suitable for your distribution.

Installation really depends on the remote target. For arm64, copy the `build/bpfd` to your bin/ directory. For Android
arm64 devices, push bpfd to the data partition by running:
```
adb push build/bpfd /data/
```

#### Prepare your kernel sources
Make sure the kernel sources are available on your development machine somewhere, and that the kernel build has
completed atleast once in the kernel source directory.

#### Setup environment variables and run tools
The following environment variables need to be setup:
- `ARCH` should point to the architecture of the `target` such as `x86` or `arm64`.
- `BCC_KERNEL_SOURCE` should point to the kernel source directory.
- `BCC_REMOTE` should point to the remote mechanism such as `adb`.

If you'd like to set the environment variables more easily, you can use the `bcc-set` and `bcc-env` tools. Simply copy
both of them from the cloned BPFd sources in `src/setup-scripts/bcc/` to your dev machine's `bin` directory. Check the
above instructions on how to use `bcc-set`.

You can also source the example .rc files from my BCC tree. Two example environment variable settings are provided for
sourcing. Here's one for [adb interface with an arm64 Android
target](https://github.com/joelagnel/bcc/blob/bcc-bpfd/arm64-adb.rc) and another one for a [local x86 target with a
process remote](https://github.com/joelagnel/bcc/blob/bcc-bpfd/x86-local.rc).

You should be all set, try running simple tools like `opensnoop` or `syscount`. For debugging, you could set the
following environment variables and check the output.

To debug kernel build process (path to kernel sources, flags):
```
export BCC_KBUILD_DEBUG=1
```
To debug BCC remote communications with BPFd, run:
```
export BCC_REMOTE__DEBUG=1
```

