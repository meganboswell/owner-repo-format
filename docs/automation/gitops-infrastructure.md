# GitOps Infrastructure for AOSP

## Overview

Implement Infrastructure as Code (IaC) and GitOps principles for AOSP development environments, enabling declarative infrastructure management, automated deployments, and version-controlled configurations.

## Table of Contents

1. [Terraform for AOSP Infrastructure](#terraform-for-aosp-infrastructure)
2. [Kubernetes GitOps with ArgoCD](#kubernetes-gitops-with-argocd)
3. [Ansible Automation](#ansible-automation)
4. [Infrastructure Monitoring](#infrastructure-monitoring)
5. [Disaster Recovery](#disaster-recovery)

## Terraform for AOSP Infrastructure

### AWS Infrastructure

```hcl
# terraform/aws/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "aosp-terraform-state"
    key    = "infrastructure/terraform.tfstate"
    region = "us-west-2"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC for AOSP Build Infrastructure
resource "aws_vpc" "aosp_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "aosp-build-vpc"
    Environment = var.environment
    Project     = "aosp"
  }
}

# Build Server Auto Scaling Group
resource "aws_launch_template" "aosp_builder" {
  name_prefix   = "aosp-builder-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  
  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    aosp_branch = var.aosp_branch
    ccache_size = var.ccache_size
  }))
  
  block_device_mappings {
    device_name = "/dev/sda1"
    
    ebs {
      volume_size = 500
      volume_type = "gp3"
      iops        = 16000
      throughput  = 1000
      encrypted   = true
    }
  }
  
  iam_instance_profile {
    name = aws_iam_instance_profile.aosp_builder.name
  }
  
  tag_specifications {
    resource_type = "instance"
    
    tags = {
      Name = "aosp-builder"
      Role = "build-server"
    }
  }
}

resource "aws_autoscaling_group" "aosp_builders" {
  name                = "aosp-builders-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.builders.arn]
  health_check_type   = "ELB"
  
  min_size         = 1
  max_size         = 10
  desired_capacity = 3
  
  launch_template {
    id      = aws_launch_template.aosp_builder.id
    version = "$Latest"
  }
  
  tag {
    key                 = "Name"
    value               = "aosp-builder"
    propagate_at_launch = true
  }
}

# EFS for shared source code
resource "aws_efs_file_system" "aosp_source" {
  creation_token = "aosp-source"
  encrypted      = true
  
  performance_mode = "maxIO"
  throughput_mode  = "provisioned"
  provisioned_throughput_in_mibps = 1024
  
  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }
  
  tags = {
    Name = "aosp-source-efs"
  }
}

# ElastiCache for build caching
resource "aws_elasticache_cluster" "build_cache" {
  cluster_id           = "aosp-build-cache"
  engine               = "redis"
  node_type            = "cache.r6g.4xlarge"
  num_cache_nodes      = 3
  parameter_group_name = "default.redis7"
  port                 = 6379
  subnet_group_name    = aws_elasticache_subnet_group.aosp.name
  security_group_ids   = [aws_security_group.cache.id]
}

# S3 for build artifacts
resource "aws_s3_bucket" "artifacts" {
  bucket = "aosp-build-artifacts-${var.environment}"
  
  tags = {
    Name        = "AOSP Build Artifacts"
    Environment = var.environment
  }
}

resource "aws_s3_bucket_versioning" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  
  rule {
    id     = "archive-old-builds"
    status = "Enabled"
    
    transition {
      days          = 30
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 365
    }
  }
}
```

### Google Cloud Infrastructure

```hcl
# terraform/gcp/main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# GKE Cluster for distributed builds
resource "google_container_cluster" "aosp_build_cluster" {
  name     = "aosp-build-cluster"
  location = var.region
  
  # Use separate node pools
  remove_default_node_pool = true
  initial_node_count       = 1
  
  network    = google_compute_network.aosp_network.name
  subnetwork = google_compute_subnetwork.aosp_subnet.name
  
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  addons_config {
    http_load_balancing {
      disabled = false
    }
    
    horizontal_pod_autoscaling {
      disabled = false
    }
  }
}

resource "google_container_node_pool" "build_nodes" {
  name       = "build-node-pool"
  location   = var.region
  cluster    = google_container_cluster.aosp_build_cluster.name
  node_count = 3
  
  autoscaling {
    min_node_count = 1
    max_node_count = 20
  }
  
  node_config {
    machine_type = "n2-highmem-32"
    disk_size_gb = 500
    disk_type    = "pd-ssd"
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    labels = {
      role = "aosp-builder"
    }
    
    taint {
      key    = "workload"
      value  = "build"
      effect = "NO_SCHEDULE"
    }
  }
}

# Cloud Storage for artifacts
resource "google_storage_bucket" "artifacts" {
  name          = "aosp-artifacts-${var.project_id}"
  location      = "US"
  storage_class = "STANDARD"
  
  versioning {
    enabled = true
  }
  
  lifecycle_rule {
    condition {
      age = 90
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }
}

# Cloud Build for CI/CD
resource "google_cloudbuild_trigger" "aosp_build" {
  name = "aosp-build-trigger"
  
  github {
    owner = var.github_owner
    name  = var.github_repo
    push {
      branch = "^main$"
    }
  }
  
  build {
    step {
      name = "gcr.io/${var.project_id}/aosp-builder"
      args = ["build", "-j64"]
      
      volumes {
        name = "ccache"
        path = "/ccache"
      }
    }
  }
}
```

## Kubernetes GitOps with ArgoCD

### ArgoCD Installation

```yaml
# argocd/install.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-application-controller
  namespace: argocd
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: aosp-infrastructure
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/your-org/aosp-infrastructure
    targetRevision: HEAD
    path: k8s/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: aosp-builds
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Kustomize Configuration

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: aosp-builds

resources:
  - namespace.yaml
  - build-deployment.yaml
  - build-service.yaml
  - build-pvc.yaml
  - build-configmap.yaml
  - build-secret.yaml

configMapGenerator:
  - name: build-config
    literals:
      - AOSP_BRANCH=main
      - USE_CCACHE=1
      - CCACHE_SIZE=50G

secretGenerator:
  - name: build-secrets
    literals:
      - GITHUB_TOKEN=placeholder
```

### Overlay for Production

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: aosp-builds-prod

patches:
  - path: build-deployment-patch.yaml
    target:
      kind: Deployment
      name: aosp-builder

configMapGenerator:
  - name: build-config
    behavior: merge
    literals:
      - ENVIRONMENT=production
      - BUILD_PARALLELISM=64
```

## Ansible Automation

### Playbook for Build Server Setup

```yaml
# ansible/playbooks/setup-build-server.yml
---
- name: Configure AOSP Build Server
  hosts: build_servers
  become: yes
  
  vars:
    aosp_dir: /home/builder/aosp
    ccache_dir: /var/cache/ccache
    aosp_branch: main
  
  tasks:
    - name: Update system packages
      apt:
        update_cache: yes
        upgrade: dist
        
    - name: Install AOSP dependencies
      apt:
        name:
          - git-core
          - gnupg
          - flex
          - bison
          - build-essential
          - zip
          - curl
          - zlib1g-dev
          - gcc-multilib
          - g++-multilib
          - libc6-dev-i386
          - lib32ncurses5-dev
          - x11proto-core-dev
          - libx11-dev
          - lib32z1-dev
          - libgl1-mesa-dev
          - libxml2-utils
          - xsltproc
          - unzip
          - fontconfig
          - python3
          - python3-pip
          - ccache
        state: present
        
    - name: Create builder user
      user:
        name: builder
        groups: sudo
        shell: /bin/bash
        create_home: yes
        
    - name: Install repo tool
      get_url:
        url: https://storage.googleapis.com/git-repo-downloads/repo
        dest: /usr/local/bin/repo
        mode: '0755'
        
    - name: Configure Git
      become_user: builder
      git_config:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        scope: global
      loop:
        - { name: user.name, value: "Builder" }
        - { name: user.email, value: "builder@example.com" }
        
    - name: Create AOSP directory
      file:
        path: "{{ aosp_dir }}"
        state: directory
        owner: builder
        group: builder
        mode: '0755'
        
    - name: Initialize AOSP repository
      become_user: builder
      command: repo init -u https://android.googlesource.com/platform/manifest -b {{ aosp_branch }}
      args:
        chdir: "{{ aosp_dir }}"
        creates: "{{ aosp_dir }}/.repo"
        
    - name: Configure ccache
      become_user: builder
      blockinfile:
        path: /home/builder/.bashrc
        block: |
          export USE_CCACHE=1
          export CCACHE_DIR={{ ccache_dir }}
          export CCACHE_SIZE=50G
          
    - name: Create ccache directory
      file:
        path: "{{ ccache_dir }}"
        state: directory
        owner: builder
        group: builder
        mode: '0755'
        
    - name: Set up systemd service for automatic sync
      template:
        src: aosp-sync.service.j2
        dest: /etc/systemd/system/aosp-sync.service
        
    - name: Enable and start sync service
      systemd:
        name: aosp-sync
        enabled: yes
        state: started
```

### Inventory Management

```yaml
# ansible/inventory/production.yml
all:
  children:
    build_servers:
      hosts:
        builder-01:
          ansible_host: 10.0.1.10
          aosp_branch: main
        builder-02:
          ansible_host: 10.0.1.11
          aosp_branch: android-14.0.0_r1
        builder-03:
          ansible_host: 10.0.1.12
          aosp_branch: main
      vars:
        ansible_user: ubuntu
        ansible_become: yes
        
    cache_servers:
      hosts:
        cache-01:
          ansible_host: 10.0.2.10
        cache-02:
          ansible_host: 10.0.2.11
      vars:
        ansible_user: ubuntu
```

## Infrastructure Monitoring

### Prometheus Configuration

```yaml
# monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "rules/*.yml"

scrape_configs:
  - job_name: 'aosp-builders'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - aosp-builds
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: aosp-builder
        
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'builder-01:9100'
          - 'builder-02:9100'
          - 'builder-03:9100'
```

### Alert Rules

```yaml
# monitoring/prometheus/rules/aosp-alerts.yml
groups:
  - name: aosp_build_alerts
    interval: 30s
    rules:
      - alert: BuildServerDown
        expr: up{job="aosp-builders"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Build server {{ $labels.instance }} is down"
          description: "Build server has been down for more than 5 minutes"
          
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/aosp"} / node_filesystem_size_bytes{mountpoint="/aosp"}) * 100 < 20
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          
      - alert: BuildFailureRate
        expr: rate(aosp_build_failures_total[1h]) > 0.1
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "High build failure rate detected"
```

## Disaster Recovery

### Backup Strategy

```bash
#!/bin/bash
# scripts/backup-aosp.sh

set -euo pipefail

BACKUP_DIR="/backups/aosp"
AOSP_DIR="/home/builder/aosp"
S3_BUCKET="s3://aosp-backups"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

# Backup source code manifest
echo "Backing up manifest..."
cd "$AOSP_DIR"
repo manifest -r -o "manifest-${TIMESTAMP}.xml"

# Upload to S3
aws s3 cp "manifest-${TIMESTAMP}.xml" "${S3_BUCKET}/manifests/"

# Backup critical configurations
echo "Backing up configurations..."
tar -czf "configs-${TIMESTAMP}.tar.gz" \
    device/*/BoardConfig.mk \
    device/*/*.mk \
    vendor/*

aws s3 cp "configs-${TIMESTAMP}.tar.gz" "${S3_BUCKET}/configs/"

# Backup build artifacts
echo "Backing up build artifacts..."
tar -czf "artifacts-${TIMESTAMP}.tar.gz" out/target/product/*/

aws s3 cp "artifacts-${TIMESTAMP}.tar.gz" "${S3_BUCKET}/artifacts/"

echo "Backup completed successfully"
```

### Restore Procedure

```bash
#!/bin/bash
# scripts/restore-aosp.sh

set -euo pipefail

BACKUP_TIMESTAMP=$1
S3_BUCKET="s3://aosp-backups"
RESTORE_DIR="/home/builder/aosp-restore"

# Download manifest
echo "Downloading manifest..."
aws s3 cp "${S3_BUCKET}/manifests/manifest-${BACKUP_TIMESTAMP}.xml" .

# Initialize repo with backup manifest
mkdir -p "$RESTORE_DIR"
cd "$RESTORE_DIR"
repo init -u https://android.googlesource.com/platform/manifest
cp "../manifest-${BACKUP_TIMESTAMP}.xml" .repo/manifests/backup.xml
repo init -m backup.xml

# Sync source
repo sync -c -j16

# Restore configurations
aws s3 cp "${S3_BUCKET}/configs/configs-${BACKUP_TIMESTAMP}.tar.gz" .
tar -xzf "configs-${BACKUP_TIMESTAMP}.tar.gz"

echo "Restore completed successfully"
```

## Best Practices

1. **Infrastructure as Code**: Version control all infrastructure
2. **GitOps Workflow**: Use pull requests for infrastructure changes
3. **Automated Testing**: Test infrastructure changes before deployment
4. **Monitoring**: Comprehensive monitoring and alerting
5. **Backup Strategy**: Regular backups with tested restore procedures
6. **Security**: Encrypt secrets, use least privilege access
7. **Documentation**: Keep runbooks and disaster recovery plans updated

## Conclusion

GitOps and IaC principles enable reliable, scalable, and maintainable AOSP development infrastructure with automated deployments and version-controlled configurations.
