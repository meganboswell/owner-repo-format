# Android AOSP Setup Guide

This guide provides comprehensive instructions for setting up an Android Open Source Project (AOSP) development environment.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Requirements](#system-requirements)
3. [Build Environment Setup](#build-environment-setup)
4. [Downloading the Source](#downloading-the-source)
5. [Building AOSP](#building-aosp)
6. [Troubleshooting](#troubleshooting)

## Prerequisites

Before beginning AOSP development, ensure you have:

- A Linux development machine (Ubuntu 18.04 LTS or later recommended)
- Sufficient disk space (at least 250GB for a single build, 400GB+ recommended)
- A reliable internet connection for downloading source code
- Basic knowledge of Linux command line operations
- Familiarity with Git version control

## System Requirements

### Hardware Requirements

- **CPU**: 64-bit x86 processor (multi-core recommended)
- **RAM**: Minimum 16GB (32GB or more recommended)
- **Storage**: 
  - 250GB for a single build
  - 400GB+ for multiple builds and branches
  - SSD strongly recommended for better performance

### Software Requirements

- **Operating System**: Ubuntu 18.04 LTS or later (64-bit)
- **Python**: Python 3.6 or later
- **Git**: Version 2.17 or later
- **JDK**: OpenJDK 11 (for Android 10 and later)

## Build Environment Setup

### 1. Install Required Packages

For Ubuntu 18.04/20.04/22.04:

```bash
sudo apt-get update
sudo apt-get install -y \
    git-core \
    gnupg \
    flex \
    bison \
    build-essential \
    zip \
    curl \
    zlib1g-dev \
    gcc-multilib \
    g++-multilib \
    libc6-dev-i386 \
    libncurses5 \
    lib32ncurses5-dev \
    x11proto-core-dev \
    libx11-dev \
    lib32z1-dev \
    libgl1-mesa-dev \
    libxml2-utils \
    xsltproc \
    unzip \
    fontconfig \
    python3 \
    python3-pip
```

### 2. Install Repo Tool

The Repo tool is used to manage the multiple Git repositories that make up the Android source.

```bash
mkdir -p ~/.bin
PATH="${HOME}/.bin:${PATH}"
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+rx ~/.bin/repo
```

Add `~/.bin` to your PATH permanently:

```bash
echo 'export PATH="${HOME}/.bin:${PATH}"' >> ~/.bashrc
source ~/.bashrc
```

### 3. Configure Git

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

## Downloading the Source

### 1. Create Working Directory

```bash
mkdir -p ~/aosp
cd ~/aosp
```

### 2. Initialize Repository

Initialize the repository for the desired branch. For example, to download the latest Android version:

```bash
repo init -u https://android.googlesource.com/platform/manifest -b main
```

Or for a specific Android version (e.g., Android 13):

```bash
repo init -u https://android.googlesource.com/platform/manifest -b android-13.0.0_r1
```

### 3. Sync the Repository

Download the source code. This may take several hours depending on your connection:

```bash
repo sync -c -j8
```

Options:
- `-c`: Download current branch only
- `-j8`: Use 8 parallel jobs (adjust based on your CPU cores)

## Building AOSP

### 1. Initialize Build Environment

```bash
source build/envsetup.sh
```

### 2. Choose Target

Select a build target. For a generic x86_64 emulator build:

```bash
lunch aosp_x86_64-eng
```

Common targets:
- `aosp_x86_64-eng`: x86_64 emulator, engineering build
- `aosp_arm64-eng`: ARM64 emulator, engineering build
- `aosp_blueline-userdebug`: Pixel 3, user-debug build

### 3. Build the Code

Start the build process:

```bash
m -j$(nproc)
```

This will use all available CPU cores. The first build can take several hours.

### 4. Run the Emulator

After a successful build:

```bash
emulator
```

## Troubleshooting

### Out of Memory Errors

If you encounter out of memory errors during build:

1. Reduce parallel jobs: `m -j4` instead of `m -j$(nproc)`
2. Increase swap space
3. Close other applications to free up RAM

### Disk Space Issues

Monitor disk usage regularly:

```bash
df -h
```

Clean build artifacts if needed:

```bash
make clean
```

### Repository Sync Issues

If `repo sync` fails:

1. Check internet connection
2. Try syncing with fewer parallel jobs: `repo sync -j1`
3. Resume interrupted sync: `repo sync -c`

### Build Failures

For build errors:

1. Check build logs for specific error messages
2. Clean and rebuild: `make clean && m -j$(nproc)`
3. Ensure all prerequisites are installed
4. Check disk space availability

### Python Version Issues

AOSP requires Python 3. Verify:

```bash
python3 --version
```

If needed, set Python 3 as default or use virtual environments.

## Additional Resources

- [Official AOSP Documentation](https://source.android.com/)
- [Android Source Building Guide](https://source.android.com/setup/build/building)
- [AOSP Repository List](https://android.googlesource.com/)
- [Android Dev Community](https://groups.google.com/g/android-building)

## Next Steps

After successfully setting up your AOSP environment:

1. Explore the source code structure
2. Make modifications to the system
3. Test your changes
4. Contribute back to AOSP if applicable

For device-specific builds, refer to device manufacturer documentation for additional requirements and instructions.
