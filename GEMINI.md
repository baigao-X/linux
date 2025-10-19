# GEMINI.md

## Project Overview

This is the source code for the Linux kernel. It is a monolithic, Unix-like operating system kernel. The source code is primarily written in C and assembly language.

## Building and Running

### Prerequisites

Before building the kernel, you need to install several tools. The exact versions are listed in `Documentation/process/changes.rst`. The main requirements are:

*   GCC
*   GNU Make
*   binutils
*   flex
*   bison
*   pahole
*   Perl
*   Python
*   OpenSSL

### Configuration

The kernel is highly configurable. To configure the kernel, you can use one of the following commands:

*   `make menuconfig`: A text-based menu-driven configuration tool.
*   `make xconfig`: A graphical configuration tool based on Qt.
*   `make gconfig`: A graphical configuration tool based on GTK.
*   `make defconfig`: Creates a default configuration for your architecture.
*   `make oldconfig`: Updates an existing `.config` file.

### Building

To build the kernel, run the following command:

```bash
make
```

This will build the kernel and the modules. The resulting kernel image will be in `arch/<your_architecture>/boot/`.

To build only the kernel, run:

```bash
make vmlinux
```

To build only the modules, run:

```bash
make modules
```

### Installation

To install the kernel and the modules, run the following command:

```bash
make install
make modules_install
```

This will install the kernel to `/boot`, the modules to `/lib/modules/<kernel_release>`, and update the bootloader.

## Development Conventions

The Linux kernel has a strict set of coding style guidelines, which are documented in `Documentation/process/coding-style.rst`. The kernel development process is also well-documented in the `Documentation/process` directory.

The kernel is developed using the Git version control system. The main repository is hosted on kernel.org.
