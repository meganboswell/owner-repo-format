# Cloud-Native AOSP Development

## Overview

Transform AOSP development into a cloud-native experience with containerization, Kubernetes orchestration, and distributed build systems for maximum scalability and efficiency.

## Table of Contents

1. [Docker for AOSP](#docker-for-aosp)
2. [Kubernetes Build Clusters](#kubernetes-build-clusters)
3. [Cloud Build Services](#cloud-build-services)
4. [Container Registry Management](#container-registry-management)
5. [Distributed Build Caching](#distributed-build-caching)

## Docker for AOSP

### Base Docker Image

```dockerfile
# Dockerfile.aosp-base
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Install AOSP dependencies
RUN apt-get update && apt-get install -y \
    git-core gnupg flex bison build-essential zip curl \
    zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 \
    libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev \
    lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip \
    fontconfig python3 python3-pip rsync wget \
    ccache distcc icecc openjdk-11-jdk && \
    rm -rf /var/lib/apt/lists/*

# Install repo tool
RUN mkdir -p /usr/local/bin && \
    curl https://storage.googleapis.com/git-repo-downloads/repo > /usr/local/bin/repo && \
    chmod a+x /usr/local/bin/repo

# Configure ccache
ENV USE_CCACHE=1
ENV CCACHE_DIR=/ccache
ENV CCACHE_SIZE=50G

# Set up build environment
RUN mkdir -p /aosp /ccache
WORKDIR /aosp

# Create build user
RUN useradd -m -s /bin/bash builder && \
    echo "builder ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

USER builder

CMD ["/bin/bash"]
```

### Build Docker Image

```bash
# Build base image
docker build -t aosp-base:latest -f Dockerfile.aosp-base .

# Push to registry
docker tag aosp-base:latest gcr.io/your-project/aosp-base:latest
docker push gcr.io/your-project/aosp-base:latest
```

### Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  aosp-builder:
    image: aosp-base:latest
    container_name: aosp-dev
    volumes:
      - ./aosp:/aosp
      - ccache:/ccache
      - ~/.gitconfig:/home/builder/.gitconfig:ro
    environment:
      - USE_CCACHE=1
      - CCACHE_DIR=/ccache
      - ANDROID_BUILD_TOP=/aosp
    privileged: true
    tty: true
    stdin_open: true
    working_dir: /aosp
    command: /bin/bash
    
  ccache-server:
    image: ccache/ccache:latest
    container_name: ccache-server
    volumes:
      - ccache:/cache
    ports:
      - "8080:8080"
    environment:
      - CCACHE_DIR=/cache
      
  distcc-master:
    image: distcc/distcc:latest
    container_name: distcc-master
    ports:
      - "3632:3632"
    environment:
      - DISTCC_HOSTS=localhost
      
volumes:
  ccache:
    driver: local
```

### Running Containerized Builds

```bash
# Start development environment
docker-compose up -d aosp-builder

# Enter container
docker-compose exec aosp-builder bash

# Inside container - initialize repo
repo init -u https://android.googlesource.com/platform/manifest -b main
repo sync -c -j16

# Build
source build/envsetup.sh
lunch aosp_x86_64-eng
m -j$(nproc)
```

## Kubernetes Build Clusters

### Kubernetes Deployment Configuration

```yaml
# k8s/aosp-builder-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aosp-builder
  namespace: aosp-builds
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aosp-builder
  template:
    metadata:
      labels:
        app: aosp-builder
    spec:
      containers:
      - name: builder
        image: gcr.io/your-project/aosp-base:latest
        resources:
          requests:
            memory: "32Gi"
            cpu: "16"
          limits:
            memory: "64Gi"
            cpu: "32"
        volumeMounts:
        - name: aosp-source
          mountPath: /aosp
        - name: ccache
          mountPath: /ccache
        env:
        - name: USE_CCACHE
          value: "1"
        - name: CCACHE_DIR
          value: "/ccache"
      volumes:
      - name: aosp-source
        persistentVolumeClaim:
          claimName: aosp-source-pvc
      - name: ccache
        persistentVolumeClaim:
          claimName: ccache-pvc
```

### Persistent Volume Claims

```yaml
# k8s/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: aosp-source-pvc
  namespace: aosp-builds
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: fast-ssd

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ccache-pvc
  namespace: aosp-builds
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
  storageClassName: fast-ssd
```

### Build Job Template

```yaml
# k8s/aosp-build-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: aosp-build-{{ .BuildID }}
  namespace: aosp-builds
spec:
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: builder
        image: gcr.io/your-project/aosp-base:latest
        command:
        - /bin/bash
        - -c
        - |
          cd /aosp
          source build/envsetup.sh
          lunch {{ .Target }}
          m -j$(nproc)
        resources:
          requests:
            memory: "48Gi"
            cpu: "24"
        volumeMounts:
        - name: aosp-source
          mountPath: /aosp
        - name: build-output
          mountPath: /aosp/out
      volumes:
      - name: aosp-source
        persistentVolumeClaim:
          claimName: aosp-source-pvc
      - name: build-output
        persistentVolumeClaim:
          claimName: build-output-pvc
```

### Deploying to Kubernetes

```bash
# Create namespace
kubectl create namespace aosp-builds

# Deploy persistent volumes
kubectl apply -f k8s/pvc.yaml

# Deploy builder pods
kubectl apply -f k8s/aosp-builder-deployment.yaml

# Submit build job
kubectl apply -f k8s/aosp-build-job.yaml

# Monitor build
kubectl logs -f -n aosp-builds job/aosp-build-001
```

## Cloud Build Services

### Google Cloud Build

```yaml
# cloudbuild.yaml
steps:
  # Sync AOSP source
  - name: 'gcr.io/$PROJECT_ID/aosp-base'
    args:
      - '-c'
      - |
        repo init -u https://android.googlesource.com/platform/manifest -b main
        repo sync -c -j32
    volumes:
      - name: 'aosp'
        path: '/workspace/aosp'
    
  # Build AOSP
  - name: 'gcr.io/$PROJECT_ID/aosp-base'
    args:
      - '-c'
      - |
        cd /workspace/aosp
        source build/envsetup.sh
        lunch aosp_x86_64-eng
        m -j32
    volumes:
      - name: 'aosp'
        path: '/workspace/aosp'
    timeout: 14400s
    
  # Upload artifacts
  - name: 'gcr.io/cloud-builders/gsutil'
    args: ['cp', '-r', '/workspace/aosp/out/target/', 'gs://$PROJECT_ID-aosp-builds/']

options:
  machineType: 'E2_HIGHCPU_32'
  diskSizeGb: 500
  logging: CLOUD_LOGGING_ONLY
  
timeout: 21600s
```

### AWS CodeBuild

```yaml
# buildspec.yml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.9
    commands:
      - apt-get update
      - apt-get install -y repo git-core build-essential
      
  pre_build:
    commands:
      - mkdir -p ~/aosp
      - cd ~/aosp
      - repo init -u https://android.googlesource.com/platform/manifest -b main
      - repo sync -c -j16
      
  build:
    commands:
      - cd ~/aosp
      - source build/envsetup.sh
      - lunch aosp_x86_64-eng
      - m -j$(nproc)
      
  post_build:
    commands:
      - aws s3 sync ~/aosp/out/target/ s3://$ARTIFACTS_BUCKET/builds/

artifacts:
  files:
    - '**/*'
  base-directory: '~/aosp/out'
  
cache:
  paths:
    - '/root/.ccache/**/*'
```

### Azure Pipelines

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  CCACHE_DIR: $(Pipeline.Workspace)/.ccache

steps:
- task: Docker@2
  inputs:
    command: build
    dockerfile: Dockerfile.aosp-base
    tags: |
      aosp-base:latest

- task: Docker@2
  inputs:
    command: run
    image: aosp-base:latest
    volumes: |
      $(Build.SourcesDirectory):/aosp
      $(Pipeline.Workspace)/.ccache:/ccache
    script: |
      cd /aosp
      repo init -u https://android.googlesource.com/platform/manifest -b main
      repo sync -c -j16
      source build/envsetup.sh
      lunch aosp_x86_64-eng
      m -j$(nproc)

- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: '$(Build.SourcesDirectory)/out'
    artifactName: 'aosp-build'
```

## Container Registry Management

### Multi-Registry Strategy

```bash
#!/bin/bash
# scripts/push_to_registries.sh

IMAGE_NAME="aosp-base"
VERSION="1.0.0"

# Google Container Registry
docker tag ${IMAGE_NAME}:latest gcr.io/${GCP_PROJECT}/${IMAGE_NAME}:${VERSION}
docker push gcr.io/${GCP_PROJECT}/${IMAGE_NAME}:${VERSION}

# Amazon ECR
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.us-west-2.amazonaws.com
docker tag ${IMAGE_NAME}:latest ${AWS_ACCOUNT}.dkr.ecr.us-west-2.amazonaws.com/${IMAGE_NAME}:${VERSION}
docker push ${AWS_ACCOUNT}.dkr.ecr.us-west-2.amazonaws.com/${IMAGE_NAME}:${VERSION}

# Azure Container Registry
az acr login --name ${ACR_NAME}
docker tag ${IMAGE_NAME}:latest ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${VERSION}
docker push ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${VERSION}

# Docker Hub
docker login -u ${DOCKERHUB_USERNAME} -p ${DOCKERHUB_TOKEN}
docker tag ${IMAGE_NAME}:latest ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${VERSION}
docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${VERSION}
```

## Distributed Build Caching

### icecc Configuration

```ini
# /etc/icecc/icecc.conf
ICECC_SCHEDULER_HOST=scheduler.aosp.local
ICECC_MAX_JOBS=32
ICECC_NICE_LEVEL=5
ICECC_REMOTE_CPP=1
```

### distcc Setup

```bash
#!/bin/bash
# scripts/setup_distcc.sh

# Install distcc
apt-get install -y distcc

# Configure distcc hosts
cat > ~/.distcc/hosts << EOF
localhost/4 worker1.aosp.local/16 worker2.aosp.local/16 worker3.aosp.local/16
EOF

# Set environment variables
export CCACHE_PREFIX="distcc"
export DISTCC_HOSTS=$(cat ~/.distcc/hosts)

# Build with distcc
make -j64 CC="distcc gcc" CXX="distcc g++"
```

### Bazel Remote Cache

```python
# .bazelrc
build --remote_cache=grpc://cache.aosp.local:9092
build --remote_timeout=600
build --remote_upload_local_results=true
build --experimental_remote_download_outputs=minimal

# Remote execution
build --remote_executor=grpc://executor.aosp.local:8980
build --remote_default_exec_properties=OSFamily=linux
build --remote_default_exec_properties=container-image=docker://aosp-base:latest
```

## Cloud Storage Integration

### Artifact Storage

```python
# scripts/upload_artifacts.py
from google.cloud import storage
import boto3
from azure.storage.blob import BlobServiceClient

def upload_to_gcs(bucket_name, source_dir, destination_dir):
    """Upload build artifacts to Google Cloud Storage"""
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    
    for file_path in Path(source_dir).rglob('*'):
        if file_path.is_file():
            blob_name = f"{destination_dir}/{file_path.relative_to(source_dir)}"
            blob = bucket.blob(blob_name)
            blob.upload_from_filename(str(file_path))

def upload_to_s3(bucket_name, source_dir, destination_dir):
    """Upload build artifacts to AWS S3"""
    s3 = boto3.client('s3')
    
    for file_path in Path(source_dir).rglob('*'):
        if file_path.is_file():
            key = f"{destination_dir}/{file_path.relative_to(source_dir)}"
            s3.upload_file(str(file_path), bucket_name, key)

def upload_to_azure(container_name, source_dir, destination_dir):
    """Upload build artifacts to Azure Blob Storage"""
    connection_string = os.environ['AZURE_STORAGE_CONNECTION_STRING']
    blob_service = BlobServiceClient.from_connection_string(connection_string)
    container_client = blob_service.get_container_client(container_name)
    
    for file_path in Path(source_dir).rglob('*'):
        if file_path.is_file():
            blob_name = f"{destination_dir}/{file_path.relative_to(source_dir)}"
            with open(file_path, 'rb') as data:
                container_client.upload_blob(name=blob_name, data=data)
```

## Monitoring and Observability

### Prometheus Metrics

```yaml
# prometheus-config.yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'aosp-builds'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - aosp-builds
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: aosp-builder
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "AOSP Build Metrics",
    "panels": [
      {
        "title": "Build Duration",
        "targets": [
          {
            "expr": "aosp_build_duration_seconds"
          }
        ]
      },
      {
        "title": "CPU Usage",
        "targets": [
          {
            "expr": "container_cpu_usage_seconds_total{namespace='aosp-builds'}"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{namespace='aosp-builds'}"
          }
        ]
      }
    ]
  }
}
```

## Cost Optimization

### Spot Instance Strategy

```python
# scripts/spot_instance_builder.py
import boto3

def create_spot_fleet():
    """Create EC2 Spot Fleet for AOSP builds"""
    client = boto3.client('ec2')
    
    response = client.request_spot_fleet(
        SpotFleetRequestConfig={
            'IamFleetRole': 'arn:aws:iam::account:role/fleet-role',
            'TargetCapacity': 10,
            'SpotPrice': '0.50',
            'LaunchSpecifications': [
                {
                    'ImageId': 'ami-aosp-builder',
                    'InstanceType': 'c5.9xlarge',
                    'KeyName': 'my-key',
                    'UserData': '''#!/bin/bash
                        cd /home/ubuntu/aosp
                        source build/envsetup.sh
                        lunch aosp_x86_64-eng
                        m -j$(nproc)
                    '''
                }
            ]
        }
    )
    
    return response['SpotFleetRequestId']
```

## Best Practices

1. **Immutable Infrastructure**: Build new images for each version
2. **Layer Caching**: Optimize Docker layer caching for faster builds
3. **Resource Limits**: Set appropriate CPU/memory limits
4. **Health Checks**: Implement liveness and readiness probes
5. **Security**: Scan images for vulnerabilities regularly
6. **Monitoring**: Track build metrics and performance
7. **Cost Management**: Use spot instances and auto-scaling

## Conclusion

Cloud-native AOSP development enables teams to scale builds efficiently, reduce infrastructure management overhead, and accelerate development cycles through containerization and orchestration.
