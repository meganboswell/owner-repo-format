# AOSP Quick Start Guide

This is a condensed version of the AOSP setup process for experienced developers. For detailed explanations, see [AOSP_SETUP.md](AOSP_SETUP.md).

## Prerequisites

- Ubuntu 18.04+ (64-bit)
- 250GB+ free disk space
- 16GB+ RAM

## Quick Setup Steps

### 1. Install Dependencies

```bash
sudo apt-get update && sudo apt-get install -y \
    git-core gnupg flex bison build-essential zip curl \
    zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 \
    libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev \
    lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip \
    fontconfig python3 python3-pip
```

### 2. Install Repo

```bash
mkdir -p ~/.bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+rx ~/.bin/repo
export PATH="${HOME}/.bin:${PATH}"
```

### 3. Configure Git

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### 4. Download Source

```bash
mkdir ~/aosp && cd ~/aosp
repo init -u https://android.googlesource.com/platform/manifest -b main
repo sync -c -j8
```

### 5. Build

```bash
source build/envsetup.sh
lunch aosp_x86_64-eng
m -j$(nproc)
```

### 6. Run Emulator

```bash
emulator
```

## Common Commands

- **Clean build**: `make clean`
- **Incremental build**: `m -j$(nproc)`
- **Update source**: `repo sync`
- **List targets**: `lunch` (without arguments)

For detailed documentation and troubleshooting, refer to [AOSP_SETUP.md](AOSP_SETUP.md).
