# GitHub Actions Workflows for AOSP

## Overview

Automate your AOSP development lifecycle with comprehensive GitHub Actions workflows for continuous integration, automated testing, security scanning, and deployment.

## Table of Contents

1. [AOSP Source Sync Automation](#aosp-source-sync-automation)
2. [Continuous Integration Pipeline](#continuous-integration-pipeline)
3. [Automated Testing](#automated-testing)
4. [Security Scanning](#security-scanning)
5. [Build Artifact Management](#build-artifact-management)

## AOSP Source Sync Automation

### Daily Source Sync Workflow

```yaml
# .github/workflows/aosp-sync.yml
name: AOSP Source Sync

on:
  schedule:
    # Run daily at 2 AM UTC
    - cron: '0 2 * * *'
  workflow_dispatch:
    inputs:
      branch:
        description: 'AOSP branch to sync'
        required: true
        default: 'main'
        type: choice
        options:
          - main
          - android-14.0.0_r1
          - android-13.0.0_r1

env:
  AOSP_DIR: ${{ github.workspace }}/aosp
  CCACHE_DIR: ${{ github.workspace }}/.ccache

jobs:
  sync-source:
    runs-on: ubuntu-latest
    timeout-minutes: 480
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        
      - name: Set up environment
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            git-core gnupg flex bison build-essential zip curl \
            zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 \
            lib32ncurses5-dev x11proto-core-dev libx11-dev \
            lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip \
            fontconfig python3 python3-pip
            
      - name: Install repo tool
        run: |
          mkdir -p ~/.bin
          curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
          chmod a+rx ~/.bin/repo
          echo "$HOME/.bin" >> $GITHUB_PATH
          
      - name: Configure Git
        run: |
          git config --global user.name "AOSP Bot"
          git config --global user.email "aosp-bot@example.com"
          
      - name: Restore source cache
        uses: actions/cache/restore@v3
        with:
          path: ${{ env.AOSP_DIR }}
          key: aosp-source-${{ github.event.inputs.branch || 'main' }}-${{ github.run_id }}
          restore-keys: |
            aosp-source-${{ github.event.inputs.branch || 'main' }}-
            
      - name: Initialize repo
        working-directory: ${{ env.AOSP_DIR }}
        run: |
          if [ ! -d .repo ]; then
            mkdir -p ${{ env.AOSP_DIR }}
            cd ${{ env.AOSP_DIR }}
            repo init -u https://android.googlesource.com/platform/manifest \
              -b ${{ github.event.inputs.branch || 'main' }}
          fi
          
      - name: Sync source code
        working-directory: ${{ env.AOSP_DIR }}
        run: |
          repo sync -c -j$(nproc) --force-sync --no-tags --no-clone-bundle
          
      - name: Generate manifest snapshot
        working-directory: ${{ env.AOSP_DIR }}
        run: |
          repo manifest -r -o manifest-snapshot.xml
          
      - name: Upload manifest snapshot
        uses: actions/upload-artifact@v3
        with:
          name: manifest-snapshot
          path: ${{ env.AOSP_DIR }}/manifest-snapshot.xml
          
      - name: Save source cache
        uses: actions/cache/save@v3
        if: always()
        with:
          path: ${{ env.AOSP_DIR }}
          key: aosp-source-${{ github.event.inputs.branch || 'main' }}-${{ github.run_id }}
          
      - name: Notify on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'AOSP Source Sync Failed',
              body: `The AOSP source sync failed for branch ${context.payload.inputs?.branch || 'main'}.
              
              [View workflow run](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})`,
              labels: ['aosp', 'sync-failure']
            })
```

## Continuous Integration Pipeline

### Build and Test Workflow

```yaml
# .github/workflows/build-and-test.yml
name: AOSP Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      target:
        description: 'Build target'
        required: true
        default: 'aosp_x86_64-eng'
        type: string

env:
  AOSP_DIR: ${{ github.workspace }}/aosp
  OUT_DIR: ${{ github.workspace }}/aosp/out
  CCACHE_DIR: ${{ github.workspace }}/.ccache

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 600
    strategy:
      matrix:
        target: 
          - aosp_x86_64-eng
          - aosp_arm64-eng
    
    steps:
      - name: Maximize build space
        run: |
          sudo rm -rf /usr/share/dotnet
          sudo rm -rf /opt/ghc
          sudo rm -rf "/usr/local/share/boost"
          sudo rm -rf "$AGENT_TOOLSDIRECTORY"
          df -h
          
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Set up build environment
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            git-core gnupg flex bison build-essential zip curl \
            zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 \
            lib32ncurses5-dev x11proto-core-dev libx11-dev \
            lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip \
            fontconfig python3 python3-pip ccache
            
      - name: Restore AOSP source
        uses: actions/cache/restore@v3
        with:
          path: ${{ env.AOSP_DIR }}
          key: aosp-source-main-
          
      - name: Restore ccache
        uses: actions/cache/restore@v3
        with:
          path: ${{ env.CCACHE_DIR }}
          key: ccache-${{ matrix.target }}-${{ github.sha }}
          restore-keys: |
            ccache-${{ matrix.target }}-
            
      - name: Configure ccache
        run: |
          ccache -M 50G
          ccache -z
          export USE_CCACHE=1
          export CCACHE_DIR=${{ env.CCACHE_DIR }}
          
      - name: Build AOSP
        working-directory: ${{ env.AOSP_DIR }}
        run: |
          source build/envsetup.sh
          lunch ${{ matrix.target }}
          m -j$(nproc)
          
      - name: ccache statistics
        run: ccache -s
        
      - name: Save ccache
        uses: actions/cache/save@v3
        if: always()
        with:
          path: ${{ env.CCACHE_DIR }}
          key: ccache-${{ matrix.target }}-${{ github.sha }}
          
      - name: Run unit tests
        working-directory: ${{ env.AOSP_DIR }}
        run: |
          source build/envsetup.sh
          lunch ${{ matrix.target }}
          m -j$(nproc) test-art-host
          
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: aosp-build-${{ matrix.target }}
          path: |
            ${{ env.OUT_DIR }}/target/product/*/system.img
            ${{ env.OUT_DIR }}/target/product/*/userdata.img
            ${{ env.OUT_DIR }}/target/product/*/ramdisk.img
          retention-days: 7
          
      - name: Generate build report
        run: |
          cat > build-report.json << EOF
          {
            "target": "${{ matrix.target }}",
            "status": "success",
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "duration": "${{ job.duration }}",
            "commit": "${{ github.sha }}",
            "branch": "${{ github.ref_name }}"
          }
          EOF
          
      - name: Upload build report
        uses: actions/upload-artifact@v3
        with:
          name: build-report-${{ matrix.target }}
          path: build-report.json

  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-suite:
          - cts
          - vts
          - gts
    
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: aosp-build-aosp_x86_64-eng
          
      - name: Set up emulator
        run: |
          # Install emulator dependencies
          sudo apt-get update
          sudo apt-get install -y qemu-kvm libvirt-daemon-system
          
      - name: Run ${{ matrix.test-suite }} tests
        run: |
          # Launch emulator
          emulator -avd test_device -no-window -no-audio &
          adb wait-for-device
          
          # Run test suite
          case "${{ matrix.test-suite }}" in
            cts)
              ./tools/cts-tradefed run cts --skip-preconditions
              ;;
            vts)
              ./tools/vts-tradefed run vts
              ;;
            gts)
              ./tools/gts-tradefed run gts
              ;;
          esac
          
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results-${{ matrix.test-suite }}
          path: test-results/
```

## Automated Testing

### Continuous Testing Workflow

```yaml
# .github/workflows/continuous-testing.yml
name: Continuous Testing

on:
  push:
    paths:
      - 'frameworks/**'
      - 'packages/**'
      - 'system/**'
  pull_request:

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run framework tests
        run: |
          cd ${{ env.AOSP_DIR }}/frameworks/base
          m test-art-host-run-test-dependencies
          atest FrameworksCoreTests
          
      - name: Run system tests
        run: |
          cd ${{ env.AOSP_DIR }}/system/core
          atest libcutils_test libutils_test
          
      - name: Generate coverage report
        run: |
          python3 scripts/generate_coverage.py \
            --output coverage-report.html \
            --format html
            
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: coverage.xml
          flags: unittests
          name: aosp-coverage

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run integration tests
        run: |
          source build/envsetup.sh
          lunch aosp_x86_64-eng
          m -j$(nproc) tradefed-all
          
          # Run integration test suite
          tradefed.sh run commandAndExit integration-tests
          
      - name: Parse test results
        run: |
          python3 scripts/parse_test_results.py \
            --input test_result.xml \
            --output test-summary.json
```

## Security Scanning

### Security Scan Workflow

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  codeql:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
      
    strategy:
      matrix:
        language: [ 'cpp', 'java', 'python' ]
        
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: ${{ matrix.language }}
          queries: security-extended,security-and-quality
          
      - name: Autobuild
        uses: github/codeql-action/autobuild@v2
        
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
        with:
          category: "/language:${{ matrix.language }}"

  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
          
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'aosp'
          path: '.'
          format: 'HTML'
          
      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: dependency-scan-results
          path: reports/

  container-scan:
    name: Container Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: |
          docker build -t aosp-builder:${{ github.sha }} -f .devcontainer/Dockerfile .
          
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'aosp-builder:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
          
      - name: Scan with Anchore
        uses: anchore/scan-action@v3
        with:
          image: 'aosp-builder:${{ github.sha }}'
          fail-build: true
          severity-cutoff: high

  secrets-scan:
    name: Secrets Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: TruffleHog scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          
      - name: GitLeaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Build Artifact Management

### Artifact Publishing Workflow

```yaml
# .github/workflows/publish-artifacts.yml
name: Publish Build Artifacts

on:
  release:
    types: [published]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        
      - name: Package artifacts
        run: |
          tar -czf aosp-images-${{ github.ref_name }}.tar.gz \
            aosp-build-*/system.img \
            aosp-build-*/userdata.img
            
      - name: Generate checksums
        run: |
          sha256sum aosp-images-${{ github.ref_name }}.tar.gz > checksums.txt
          
      - name: Upload to release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            aosp-images-${{ github.ref_name }}.tar.gz
            checksums.txt
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Push to cloud storage
        run: |
          # Google Cloud Storage
          gsutil cp aosp-images-${{ github.ref_name }}.tar.gz \
            gs://aosp-builds/releases/
            
          # AWS S3
          aws s3 cp aosp-images-${{ github.ref_name }}.tar.gz \
            s3://aosp-builds/releases/
```

## Advanced Workflows

### Matrix Build Strategy

```yaml
# .github/workflows/matrix-build.yml
name: Matrix Build

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-20.04, ubuntu-22.04]
        target: [aosp_x86_64-eng, aosp_arm64-eng]
        variant: [user, userdebug, eng]
        include:
          - os: ubuntu-22.04
            target: aosp_x86_64-eng
            variant: eng
            experimental: true
        exclude:
          - os: ubuntu-20.04
            variant: user
            
    steps:
      - uses: actions/checkout@v4
      
      - name: Build ${{ matrix.target }}-${{ matrix.variant }}
        run: |
          source build/envsetup.sh
          lunch ${{ matrix.target }}
          m -j$(nproc) BUILD_VARIANT=${{ matrix.variant }}
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      target:
        required: true
        type: string
      variant:
        required: false
        type: string
        default: 'eng'
    secrets:
      BUILD_TOKEN:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Authenticate
        run: |
          echo "${{ secrets.BUILD_TOKEN }}" | gh auth login --with-token
          
      - name: Build
        run: |
          source build/envsetup.sh
          lunch ${{ inputs.target }}
          m -j$(nproc)
```

## Monitoring and Notifications

### Build Status Notifications

```yaml
# .github/workflows/notifications.yml
name: Build Notifications

on:
  workflow_run:
    workflows: ["AOSP Build and Test"]
    types: [completed]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Slack notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Build ${{ github.event.workflow_run.conclusion }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "AOSP Build Status: *${{ github.event.workflow_run.conclusion }}*"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
          
      - name: Email notification
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 465
          username: ${{ secrets.EMAIL_USERNAME }}
          password: ${{ secrets.EMAIL_PASSWORD }}
          subject: AOSP Build ${{ github.event.workflow_run.conclusion }}
          body: |
            Build ${{ github.event.workflow_run.conclusion }} for commit ${{ github.sha }}
            View run: ${{ github.event.workflow_run.html_url }}
          to: developers@example.com
```

## Best Practices

1. **Use caching**: Cache AOSP source, ccache, and dependencies
2. **Parallel builds**: Use matrix strategy for multiple targets
3. **Resource optimization**: Maximize disk space, use self-hosted runners
4. **Security**: Scan code, dependencies, and containers
5. **Monitoring**: Track build metrics and send notifications
6. **Artifact management**: Version and store build outputs
7. **Reusable workflows**: Create modular, composable workflows

## Conclusion

GitHub Actions provides powerful automation capabilities for AOSP development, enabling continuous integration, automated testing, and secure deployment pipelines.
