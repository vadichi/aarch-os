# aarch-os

> A simple aarch (ARMv8-A) bootloader and kernel.
> 
> The aarch64 QEMU `virt-8.2` machine is currently the primary target, but support for real hardware is planned. 



## Prerequisites

Note: The project uses the GNU compiler toolchain and binutils; some code may use GNU-specific extensions. Compatibility with alternative tooling is not guaranteed.

To build the project, you will need the [GNU toolchain for ARM64 development](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain) (`aarch64-none-elf`).
