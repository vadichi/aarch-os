<show-structure for="chapter,procedure" depth="3"/>

# Hello, GRUB

Before we start, we need to ensure QEMU is properly configured; 
download UEFI and GRUB, and configure them to run in a virtual machine.


## QEMU
First, ensure that you have QEMU installed and available in `$PATH`.
Specifically, we will be using `qemu-system-aarch64`.

If you don't have it, you can get it with:
```Bash
brew install qemu
```
{prompt="$"}

To run an aarch64 virtual machine, some basic parameters need to be set:
- Machine type (`-M`): `virt` – a generic PC-like system.
This is a versioned machine, where breaking changes can occur with new QEMU versions,
so we explicitly specify version 8.2 (latest as of writing) for reproducibility.

- CPU model (`-cpu`): `cortex-a53` – ARM Cortex-A53, a common aarch64 processor.
It is not particularly important which one we use, but we need to specify one explicitly,
as QEMU defaults to a 32-bit CPU.

- RAM amount (`-m`): `1024` – 1GiB. This should be more than enough for our purposes.

- Disable graphical output (`-nographic`): we will start out with a simple shell-based interface,
so we don't need a graphical window.

Your final command should look like this:
```Bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a53 \
    -m 1024 \
    -nographic
```
{prompt="$"}

This command will start a machine with the specified parameters, which will do nothing.
QEMU will capture the keyboard input. To exit, press <shortcut>Ctrl+A</shortcut>
(the escape sequence), followed by <shortcut>X</shortcut>.
Ensure that this runs without errors before proceeding. 

At the end of the tutorial, we will combine all necessary flags into a Makefile target
for easier execution.

## UEFI
Most real machines supporting UEFI will have their own UEFI firmware.
When using QEMU, however, we will need to provide our own. 

We will be using [TianoCore EDK II](https://github.com/tianocore/edk2), 
an open-source implementation of UEFI by Intel. You can try to build it
from source following the instructions in the repository, but it's easier to use
pre-built binaries provided in some Linux package registries.

For example, Debian provides the `qemu-efi-aarch64` package, which contains EDK II
binaries built specifically for QEMU. Download the package from the 
[Debian repository](https://packages.debian.org/unstable/qemu-efi-aarch64).

<path>.deb</path> files are just tarballs, so you can extract the contents of the package with:
```Bash
tar -xvf qemu-efi-aarch64_2024.02-2_all.deb
```
{prompt="$"}

```
x debian-binary
x control.tar.xz
x data.tar.xz
```
{collapsible="true" default-state="collapsed"}

The binaries we are after are located in the <path>data.tar.xz</path> file. Extract it as well:
```Bash
tar -xvzf data.tar.xz
```
{prompt="$"}

```
x ./
x ./usr/
x ./usr/share/
x ./usr/share/AAVMF/
x ./usr/share/AAVMF/AAVMF_CODE.no-secboot.fd
x ./usr/share/AAVMF/AAVMF_CODE.secboot.fd
x ./usr/share/AAVMF/AAVMF_VARS.fd
x ./usr/share/AAVMF/AAVMF_VARS.ms.fd
x ./usr/share/AAVMF/AAVMF_VARS.snakeoil.fd
x ./usr/share/doc/
x ./usr/share/doc/qemu-efi-aarch64/
x ./usr/share/doc/qemu-efi-aarch64/README.Debian
x ./usr/share/doc/qemu-efi-aarch64/changelog.Debian.gz
x ./usr/share/doc/qemu-efi-aarch64/copyright
x ./usr/share/qemu/
x ./usr/share/qemu/firmware/
x ./usr/share/qemu/firmware/40-edk2-aarch64-secure-enrolled.json
x ./usr/share/qemu/firmware/50-edk2-aarch64-secure.json
x ./usr/share/qemu/firmware/60-edk2-aarch64.json
x ./usr/share/qemu-efi-aarch64/
x ./usr/share/qemu-efi-aarch64/PkKek-1-snakeoil.key
x ./usr/share/qemu-efi-aarch64/PkKek-1-snakeoil.pem
x ./usr/share/qemu-efi-aarch64/QEMU_EFI.fd
x ./usr/share/AAVMF/AAVMF_CODE.fd
x ./usr/share/AAVMF/AAVMF_CODE.ms.fd
x ./usr/share/AAVMF/AAVMF_CODE.snakeoil.fd
```
{collapsible="true" default-state="collapsed"}



We are only interested in two of the <path>AAVMF</path> files:
- <path>AAVMF_CODE.no-secboot.fd</path> - the UEFI binary, configured to run without secure boot.
- <path>AAVMF_VARS.fd</path> – the variables file, holding persistent UEFI configuration.

Copy these two into the <path>qemu</path> directory in the root of the repository.
Feel free to delete the rest of the files.

We will connect both of these files to the virtual machine as system flash memory (`pflash`)
(this is the role filled by the motherboard EEPROM in a real machine)
using the `-drive` flag.

Both files are raw binaries, so we will use the `raw` format for the drives.
We will also mark the firmware as read-only, so we don't accidentally overwrite it from the virtual machine.

```Bash
qemu-system-aarch64 -M virt -cpu cortex-a53 -m 1024 -nographic \
    -drive if=pflash,format=raw,readonly=on,file=qemu/AAVMF_CODE.no-secboot.fd \
    -drive if=pflash,format=raw,file=qemu/AAVMF_VARS.fd
```
{prompt="$"}

Running this command, you should see the EDK II version flash momentarily.
Then, UEFI will attempt and fail to boot from a drive, triggering the network boot process:

```
>>Start PXE over IPv4.
```
{collapsible="true" default-state="collapsed"}

You will have to wait a few minutes, while all the network protocols time out.
After that, you will finally be greeted by the UEFI shell:
```
UEFI Interactive Shell v2.2
EDK II
UEFI v2.70 (Debian distribution of EDK II, 0x00010000)
map: No mapping found.
Press ESC in 1 seconds to skip startup.nsh or any other key to continue.
Shell>
```
{collapsible="true" default-state="collapsed"}

Let's configure UEFI to skip network boot to avoid the excruciating wait in the future.

First, dump the current boot order:
```
bcfg boot dump
```
{prompt="Shell>"}

```
Option: 00. Variable: Boot0000
  Desc    - UiApp
  DevPath - Fv(64074AFE-340A-4BE6-94BA-91B5B4D0F71E)/FvFile(462CAA21-7614-4503-836E-8AB6F4662331)
  Optional- N
Option: 01. Variable: Boot0001
  Desc    - UEFI PXEv4 (MAC:525400123456)
  DevPath - PciRoot(0x0)/Pci(0x1,0x0)/MAC(525400123456,0x1)/IPv4(0.0.0.0)
  Optional- Y
Option: 02. Variable: Boot0002
  Desc    - UEFI PXEv6 (MAC:525400123456)
  DevPath - PciRoot(0x0)/Pci(0x1,0x0)/MAC(525400123456,0x1)/IPv6(0000:0000:0000:0000:0000:0000:0000:0000)
  Optional- Y
Option: 03. Variable: Boot0003
  Desc    - UEFI HTTPv4 (MAC:525400123456)
  DevPath - PciRoot(0x0)/Pci(0x1,0x0)/MAC(525400123456,0x1)/IPv4(0.0.0.0)/Uri()
  Optional- Y
Option: 04. Variable: Boot0004
  Desc    - UEFI HTTPv6 (MAC:525400123456)
  DevPath - PciRoot(0x0)/Pci(0x1,0x0)/MAC(525400123456,0x1)/IPv6(0000:0000:0000:0000:0000:0000:0000:0000)/Uri()
  Optional- Y
Option: 05. Variable: Boot0005
  Desc    - EFI Internal Shell
  DevPath - Fv(64074AFE-340A-4BE6-94BA-91B5B4D0F71E)/FvFile(7C04A583-9E3E-4F1C-AD65-E05268D0B4D1)
  Optional- N
```
{collapsible="true" default-state="collapsed"}

Move the EFI Internal Shell (option 05) above all the network boot options 
and dump the boot order again to verify the result. `UiApp` should remain as the first option, 
followed by `EFI Internal Shell`.

```
bcfg boot mv 05 01
```
{prompt="Shell>"}

```
bcfg boot dump
```
{prompt="Shell>"}

All changes should be saved automatically, so you can now exit the shell.

Finally, verify that the changes are persisted by rebooting the machine.
It should show a black screen for a few seconds, and boot into the UEFI shell,
bypassing network boot.


## GRUB

> Much of the following is based on the instructions on the [page on GRUB](https://wiki.osdev.org/GRUB_2)
> in the OSDev wiki. You can refer to it for more detailed information.

### Building GRUB tools

Setting up GRUB is a little more involved, as you will have to build the tooling ourselves
before we can actually create a bootable image.

Find a temporary directory to work in, and clone the GRUB repository:
```Bash
git clone git://git.savannah.gnu.org/grub.git
```
{prompt="$"}

Navigate into the repository and bootstrap the build system:

```Bash
cd grub && ./bootstrap && ./autogen.sh
```
{prompt="$"}

Create a build directory:

```Bash
mkdir build && cd build
```
{prompt="$"}

Configure the build system, specifying the target architecture and the toolchain:

```Bash
../configure --with-platform=efi --target=aarch64-none-elf  \
    --disable-werror \
    TARGET_CC=aarch64-none-elf-gcc \
    TARGET_NM=aarch64-none-elf-nm \
    TARGET_STRIP=aarch64-none-elf-strip \
    TARGET_OBJCOPY=aarch64-none-elf-objcopy \
    TARGET_RANLIB=aarch64-none-elf-ranlib
```
{prompt="$"}

> The build script makes use of `awk`, a text processing tool, which is installed on macOS by default.
> However, it uses some GNU-specific functionality, not supported by the system `awk`. 
> 
> Instead, we will use `gawk`, GNU's version of `awk`.
{style="warning"}


This can be installed with Homebrew:

```Bash
brew install gawk
```
{prompt="$"}

You will also need it to `$PATH`, with a higher priority than the system `awk`.

```Bash
export PATH="/opt/homebrew/opt/gawk/libexec/gnubin:$PATH"
```
{prompt="$"}

It might also be useful to this line it to your <path>~/.zprofile</path> or equivalent to have 
`gawk` available in every shell.

Finally, build and install the GRUB tools.
Don't worry, this only installs some developer tools and will **not** affect the bootloader on your machine.

```Bash
make && make install
```
{prompt="$"}


### Creating a bootable image
You should now have the necessary tools to create a bootable image.

First, we will


