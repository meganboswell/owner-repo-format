# AOSP Troubleshooting Guide

This guide covers common issues encountered during AOSP setup and development.

## Table of Contents

1. [Repository Sync Issues](#repository-sync-issues)
2. [Build Failures](#build-failures)
3. [Environment Issues](#environment-issues)
4. [Performance Issues](#performance-issues)
5. [Emulator Issues](#emulator-issues)

## Repository Sync Issues

### Issue: Repo sync fails with network errors

**Symptoms:**
```
error: Exited sync due to fetch errors
```

**Solutions:**

1. **Check network connection:**
   ```bash
   ping google.com
   ```

2. **Reduce parallel jobs:**
   ```bash
   repo sync -j1
   ```

3. **Resume interrupted sync:**
   ```bash
   repo sync -c --no-tags
   ```

4. **Use local mirror (if available):**
   ```bash
   repo init -u https://android.googlesource.com/platform/manifest --mirror
   ```

### Issue: Authentication errors

**Symptoms:**
```
fatal: Authentication failed
```

**Solutions:**

1. **Generate and add SSH key:**
   ```bash
   ssh-keygen -t rsa -C "your@email.com"
   cat ~/.ssh/id_rsa.pub
   ```
   Add the key to your Google account.

2. **Use HTTPS instead of SSH:**
   Edit `.repo/manifests/default.xml` to use HTTPS URLs.

## Build Failures

### Issue: Out of memory during build

**Symptoms:**
```
FAILED: out of memory
```

**Solutions:**

1. **Reduce build parallelism:**
   ```bash
   m -j2  # Use only 2 parallel jobs
   ```

2. **Increase swap space:**
   ```bash
   sudo fallocate -l 8G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

3. **Close other applications:**
   ```bash
   # Monitor memory usage
   free -h
   ```

### Issue: Missing dependencies

**Symptoms:**
```
command not found: bison
flex: not found
```

**Solutions:**

1. **Install missing packages:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y bison flex
   ```

2. **Verify all dependencies:**
   ```bash
   which git python3 flex bison make
   ```

### Issue: Python version conflicts

**Symptoms:**
```
python2 is not supported
```

**Solutions:**

1. **Verify Python 3 is default:**
   ```bash
   python3 --version
   which python3
   ```

2. **Create alias (if needed):**
   ```bash
   alias python=python3
   ```

3. **Use virtualenv:**
   ```bash
   python3 -m venv aosp-env
   source aosp-env/bin/activate
   ```

### Issue: Ninja build failures

**Symptoms:**
```
ninja: error: loading 'build.ninja'
```

**Solutions:**

1. **Clean build files:**
   ```bash
   rm -rf out/
   source build/envsetup.sh
   lunch aosp_x86_64-eng
   m -j$(nproc)
   ```

2. **Check disk space:**
   ```bash
   df -h
   ```

## Environment Issues

### Issue: envsetup.sh not found

**Symptoms:**
```
bash: build/envsetup.sh: No such file or directory
```

**Solutions:**

1. **Verify you're in the AOSP root directory:**
   ```bash
   pwd
   ls build/envsetup.sh
   ```

2. **Complete repo sync first:**
   ```bash
   repo sync
   ```

### Issue: Lunch target not available

**Symptoms:**
```
** Don't have a product spec for: 'aosp_device'
```

**Solutions:**

1. **List available targets:**
   ```bash
   source build/envsetup.sh
   lunch
   ```

2. **Use a generic target:**
   ```bash
   lunch aosp_x86_64-eng
   ```

### Issue: PATH issues with repo tool

**Symptoms:**
```
repo: command not found
```

**Solutions:**

1. **Add to PATH:**
   ```bash
   export PATH="${HOME}/.bin:${PATH}"
   echo 'export PATH="${HOME}/.bin:${PATH}"' >> ~/.bashrc
   source ~/.bashrc
   ```

2. **Verify installation:**
   ```bash
   which repo
   repo version
   ```

## Performance Issues

### Issue: Build takes too long

**Symptoms:**
- Build takes more than 8-10 hours

**Solutions:**

1. **Use ccache:**
   ```bash
   export USE_CCACHE=1
   export CCACHE_DIR=~/.ccache
   ccache -M 50G
   ```

2. **Optimize parallel jobs:**
   ```bash
   # Use number of CPU cores minus 2
   m -j$(($(nproc) - 2))
   ```

3. **Use SSD for build:**
   - Ensure AOSP directory is on SSD, not HDD

4. **Disable unnecessary modules:**
   ```bash
   # Build specific module instead of full system
   m <module_name>
   ```

### Issue: Disk space running low

**Symptoms:**
```
No space left on device
```

**Solutions:**

1. **Check disk usage:**
   ```bash
   df -h
   du -sh ~/aosp/out
   ```

2. **Clean build artifacts:**
   ```bash
   make clean
   # or for full clean
   rm -rf out/
   ```

3. **Clean repo cache:**
   ```bash
   rm -rf .repo/project-objects/
   repo sync -l
   ```

## Emulator Issues

### Issue: Emulator fails to start

**Symptoms:**
```
emulator: ERROR: ...
```

**Solutions:**

1. **Check emulator binary:**
   ```bash
   which emulator
   file $(which emulator)
   ```

2. **Verify build completed:**
   ```bash
   ls out/target/product/*/system.img
   ```

3. **Run with verbose output:**
   ```bash
   emulator -verbose
   ```

### Issue: KVM not available

**Symptoms:**
```
KVM is not installed or not available
```

**Solutions:**

1. **Check KVM support:**
   ```bash
   egrep -c '(vmx|svm)' /proc/cpuinfo
   ```

2. **Install KVM:**
   ```bash
   sudo apt-get install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
   sudo adduser $USER kvm
   ```

3. **Verify KVM access:**
   ```bash
   ls -l /dev/kvm
   ```

### Issue: Graphics acceleration issues

**Symptoms:**
- Slow or garbled display
- Black screen

**Solutions:**

1. **Try software rendering:**
   ```bash
   emulator -gpu swiftshader
   ```

2. **Check OpenGL support:**
   ```bash
   glxinfo | grep OpenGL
   ```

3. **Update graphics drivers:**
   ```bash
   ubuntu-drivers devices
   sudo ubuntu-drivers autoinstall
   ```

## Additional Tips

### Enable verbose error messages

```bash
# During build
m -v -j$(nproc)

# During repo sync
repo sync -v
```

### Save build logs

```bash
m -j$(nproc) 2>&1 | tee build.log
```

### Check build configuration

```bash
source build/envsetup.sh
lunch  # Shows current target
printconfig
```

### Reset to clean state

If all else fails:

```bash
cd ~/aosp
rm -rf out/
repo sync -c --force-sync
source build/envsetup.sh
lunch aosp_x86_64-eng
m -j$(nproc)
```

## Getting Help

If you continue to experience issues:

1. Check the [official AOSP documentation](https://source.android.com/)
2. Search [Stack Overflow](https://stackoverflow.com/questions/tagged/aosp)
3. Join the [android-building](https://groups.google.com/g/android-building) group
4. Check your specific device's documentation

## Reporting Bugs

When reporting issues, include:
- AOSP version/branch
- Host OS and version
- Hardware specifications
- Complete error messages
- Steps to reproduce
- Build logs (if applicable)
