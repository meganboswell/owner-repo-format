# Automated Dependency Management for AOSP

## Overview

Implement intelligent dependency management systems for AOSP development with automated updates, vulnerability scanning, and smart binary fetching to streamline the development workflow.

## Table of Contents

1. [Automated Binary Management](#automated-binary-management)
2. [Dependency Version Control](#dependency-version-control)
3. [Security Vulnerability Scanning](#security-vulnerability-scanning)
4. [Smart Update Strategies](#smart-update-strategies)
5. [Dependency Caching](#dependency-caching)

## Automated Binary Management

### Binary Fetcher Service

```python
# scripts/binary_fetcher.py
import requests
import hashlib
import json
from pathlib import Path
from typing import Dict, List
import concurrent.futures

class BinaryFetcher:
    """Automated binary dependency fetcher for AOSP"""
    
    def __init__(self, manifest_path: str, cache_dir: str = ".binary_cache"):
        self.manifest_path = Path(manifest_path)
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(exist_ok=True)
        
    def load_manifest(self) -> Dict:
        """Load binary dependency manifest"""
        with open(self.manifest_path) as f:
            return json.load(f)
    
    def fetch_binary(self, name: str, url: str, checksum: str) -> bool:
        """Fetch and verify single binary"""
        cache_path = self.cache_dir / name
        
        # Check cache first
        if cache_path.exists():
            if self.verify_checksum(cache_path, checksum):
                print(f"✓ {name} (cached)")
                return True
            else:
                cache_path.unlink()
        
        # Download binary
        print(f"⬇ Downloading {name}...")
        response = requests.get(url, stream=True)
        response.raise_for_status()
        
        with open(cache_path, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
        
        # Verify checksum
        if not self.verify_checksum(cache_path, checksum):
            cache_path.unlink()
            raise ValueError(f"Checksum verification failed for {name}")
        
        print(f"✓ {name}")
        return True
    
    def verify_checksum(self, file_path: Path, expected: str) -> bool:
        """Verify file SHA-256 checksum"""
        sha256 = hashlib.sha256()
        with open(file_path, 'rb') as f:
            for chunk in iter(lambda: f.read(4096), b''):
                sha256.update(chunk)
        return sha256.hexdigest() == expected
    
    def fetch_all(self, parallel: int = 4) -> None:
        """Fetch all binaries in parallel"""
        manifest = self.load_manifest()
        binaries = manifest.get('binaries', [])
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=parallel) as executor:
            futures = {
                executor.submit(
                    self.fetch_binary,
                    binary['name'],
                    binary['url'],
                    binary['checksum']
                ): binary['name']
                for binary in binaries
            }
            
            for future in concurrent.futures.as_completed(futures):
                name = futures[future]
                try:
                    future.result()
                except Exception as e:
                    print(f"✗ {name}: {e}")
    
    def install_binaries(self, target_dir: str) -> None:
        """Install cached binaries to target directory"""
        target_path = Path(target_dir)
        target_path.mkdir(parents=True, exist_ok=True)
        
        manifest = self.load_manifest()
        for binary in manifest.get('binaries', []):
            src = self.cache_dir / binary['name']
            dst = target_path / binary['install_path']
            
            if src.exists():
                dst.parent.mkdir(parents=True, exist_ok=True)
                shutil.copy2(src, dst)
                if binary.get('executable', False):
                    dst.chmod(0o755)
                print(f"Installed {binary['name']} -> {dst}")

# Usage
if __name__ == "__main__":
    fetcher = BinaryFetcher("manifests/binaries.json")
    fetcher.fetch_all()
    fetcher.install_binaries("vendor/bin")
```

### Binary Manifest

```json
{
  "version": "1.0.0",
  "binaries": [
    {
      "name": "arm-none-eabi-gcc",
      "version": "12.2.0",
      "url": "https://developer.arm.com/-/media/Files/downloads/gnu/12.2.rel1/binrel/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-eabi.tar.xz",
      "checksum": "84be93d0f9e96a15addd490b6e237f588c641c8afdf90e7610a628007fc96867",
      "install_path": "toolchains/arm/bin/arm-none-eabi-gcc",
      "executable": true
    },
    {
      "name": "aarch64-linux-android-clang",
      "version": "r25c",
      "url": "https://dl.google.com/android/repository/android-ndk-r25c-linux.zip",
      "checksum": "769ee342ea75f80619d985c2da990c48b3d8eaf45f48783a2d48870d04b46108",
      "install_path": "ndk/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android-clang",
      "executable": true
    }
  ]
}
```

## Dependency Version Control

### Dependency Lock File

```python
# scripts/generate_lock_file.py
import subprocess
import json
from datetime import datetime
from typing import Dict, List

class DependencyLocker:
    """Generate dependency lock file for reproducible builds"""
    
    def __init__(self, aosp_dir: str):
        self.aosp_dir = aosp_dir
        
    def get_git_revisions(self) -> Dict[str, str]:
        """Get exact Git revisions for all projects"""
        result = subprocess.run(
            ['repo', 'forall', '-c', 'echo $REPO_PROJECT $REPO_LREV'],
            cwd=self.aosp_dir,
            capture_output=True,
            text=True
        )
        
        revisions = {}
        for line in result.stdout.strip().split('\n'):
            if line:
                project, revision = line.split()
                revisions[project] = revision
        
        return revisions
    
    def get_binary_versions(self) -> Dict[str, str]:
        """Get versions of binary dependencies"""
        binaries = {}
        
        # Check prebuilt binaries
        prebuilt_dir = Path(self.aosp_dir) / "prebuilts"
        for binary_dir in prebuilt_dir.glob("*/"):
            version_file = binary_dir / "VERSION"
            if version_file.exists():
                with open(version_file) as f:
                    binaries[binary_dir.name] = f.read().strip()
        
        return binaries
    
    def generate_lock_file(self, output: str = "aosp.lock") -> None:
        """Generate comprehensive lock file"""
        lock_data = {
            "generated": datetime.utcnow().isoformat(),
            "git_revisions": self.get_git_revisions(),
            "binary_versions": self.get_binary_versions(),
            "build_tools": {
                "repo": self.get_tool_version("repo"),
                "python": self.get_tool_version("python3"),
                "java": self.get_tool_version("java"),
            }
        }
        
        with open(output, 'w') as f:
            json.dump(lock_data, f, indent=2)
        
        print(f"Generated lock file: {output}")
    
    def get_tool_version(self, tool: str) -> str:
        """Get version of build tool"""
        try:
            result = subprocess.run(
                [tool, '--version'],
                capture_output=True,
                text=True,
                timeout=5
            )
            return result.stdout.split('\n')[0]
        except:
            return "unknown"
    
    def restore_from_lock(self, lock_file: str) -> None:
        """Restore exact versions from lock file"""
        with open(lock_file) as f:
            lock_data = json.load(f)
        
        print("Restoring from lock file...")
        
        # Restore Git revisions
        for project, revision in lock_data['git_revisions'].items():
            print(f"Restoring {project} to {revision[:8]}...")
            subprocess.run(
                ['git', 'checkout', revision],
                cwd=Path(self.aosp_dir) / project,
                capture_output=True
            )
        
        print("Restore complete")

# Usage
if __name__ == "__main__":
    import sys
    
    locker = DependencyLocker("~/aosp")
    
    if sys.argv[1] == "generate":
        locker.generate_lock_file()
    elif sys.argv[1] == "restore":
        locker.restore_from_lock("aosp.lock")
```

## Security Vulnerability Scanning

### Automated CVE Scanner

```python
# scripts/cve_scanner.py
import requests
import json
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class Vulnerability:
    cve_id: str
    severity: str
    description: str
    affected_versions: List[str]
    fixed_version: str

class CVEScanner:
    """Scan AOSP dependencies for known vulnerabilities"""
    
    def __init__(self):
        self.nvd_api_url = "https://services.nvd.nist.gov/rest/json/cves/2.0"
        self.github_api_url = "https://api.github.com/advisories"
        
    def scan_package(self, package: str, version: str) -> List[Vulnerability]:
        """Scan package for vulnerabilities"""
        vulnerabilities = []
        
        # Query NVD database
        params = {
            'keywordSearch': package,
            'resultsPerPage': 100
        }
        
        response = requests.get(self.nvd_api_url, params=params)
        data = response.json()
        
        for vuln in data.get('vulnerabilities', []):
            cve = vuln.get('cve', {})
            cve_id = cve.get('id')
            
            # Check if version is affected
            if self.is_version_affected(version, cve):
                vulnerabilities.append(Vulnerability(
                    cve_id=cve_id,
                    severity=self.get_severity(cve),
                    description=self.get_description(cve),
                    affected_versions=self.get_affected_versions(cve),
                    fixed_version=self.get_fixed_version(cve)
                ))
        
        return vulnerabilities
    
    def scan_aosp_dependencies(self, manifest: Dict) -> Dict[str, List[Vulnerability]]:
        """Scan all AOSP dependencies"""
        results = {}
        
        for dep in manifest.get('dependencies', []):
            print(f"Scanning {dep['name']} {dep['version']}...")
            vulns = self.scan_package(dep['name'], dep['version'])
            
            if vulns:
                results[dep['name']] = vulns
                print(f"  Found {len(vulns)} vulnerabilities")
            else:
                print(f"  No vulnerabilities found")
        
        return results
    
    def generate_report(self, results: Dict, output: str = "vulnerability-report.html"):
        """Generate HTML vulnerability report"""
        html = """
        <!DOCTYPE html>
        <html>
        <head>
            <title>AOSP Vulnerability Report</title>
            <style>
                body { font-family: Arial, sans-serif; margin: 20px; }
                .critical { color: #d73a49; }
                .high { color: #ff6b35; }
                .medium { color: #ffa500; }
                .low { color: #28a745; }
                table { border-collapse: collapse; width: 100%; }
                th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                th { background-color: #f2f2f2; }
            </style>
        </head>
        <body>
            <h1>AOSP Vulnerability Report</h1>
        """
        
        for package, vulns in results.items():
            html += f"<h2>{package}</h2>"
            html += "<table><tr><th>CVE ID</th><th>Severity</th><th>Description</th><th>Fixed Version</th></tr>"
            
            for vuln in vulns:
                severity_class = vuln.severity.lower()
                html += f"""
                <tr>
                    <td>{vuln.cve_id}</td>
                    <td class="{severity_class}">{vuln.severity}</td>
                    <td>{vuln.description}</td>
                    <td>{vuln.fixed_version}</td>
                </tr>
                """
            
            html += "</table>"
        
        html += "</body></html>"
        
        with open(output, 'w') as f:
            f.write(html)
        
        print(f"Report generated: {output}")
    
    def get_severity(self, cve: Dict) -> str:
        """Extract severity from CVE data"""
        metrics = cve.get('metrics', {})
        if 'cvssMetricV31' in metrics:
            return metrics['cvssMetricV31'][0]['cvssData']['baseSeverity']
        return "UNKNOWN"
    
    def get_description(self, cve: Dict) -> str:
        """Extract description from CVE data"""
        descriptions = cve.get('descriptions', [])
        if descriptions:
            return descriptions[0].get('value', 'No description available')
        return "No description available"

# GitHub Actions integration
def run_in_ci():
    """Run CVE scan in CI pipeline"""
    import sys
    
    scanner = CVEScanner()
    
    with open('manifests/dependencies.json') as f:
        manifest = json.load(f)
    
    results = scanner.scan_aosp_dependencies(manifest)
    
    # Generate reports
    scanner.generate_report(results)
    
    # Fail build if critical vulnerabilities found
    critical_count = sum(
        1 for vulns in results.values()
        for v in vulns if v.severity == "CRITICAL"
    )
    
    if critical_count > 0:
        print(f"❌ Found {critical_count} critical vulnerabilities!")
        sys.exit(1)
    else:
        print("✓ No critical vulnerabilities found")

if __name__ == "__main__":
    run_in_ci()
```

## Smart Update Strategies

### Intelligent Update Manager

```python
# scripts/update_manager.py
import subprocess
import json
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class Update:
    project: str
    current_version: str
    latest_version: str
    breaking_changes: bool
    changelog: str

class UpdateManager:
    """Intelligent dependency update manager"""
    
    def __init__(self, aosp_dir: str):
        self.aosp_dir = aosp_dir
        
    def check_updates(self) -> List[Update]:
        """Check for available updates"""
        updates = []
        
        # Check each AOSP project
        result = subprocess.run(
            ['repo', 'forall', '-c', 
             'git fetch origin && git log --oneline HEAD..origin/$(git rev-parse --abbrev-ref HEAD) | wc -l'],
            cwd=self.aosp_dir,
            capture_output=True,
            text=True
        )
        
        # Parse results and identify updates
        for line in result.stdout.strip().split('\n'):
            if line and line != '0':
                # Updates available
                project = self.get_project_name()
                updates.append(self.analyze_update(project))
        
        return updates
    
    def analyze_update(self, project: str) -> Update:
        """Analyze update for breaking changes"""
        # Get changelog
        changelog = subprocess.run(
            ['git', 'log', '--oneline', 'HEAD..origin/main'],
            cwd=f"{self.aosp_dir}/{project}",
            capture_output=True,
            text=True
        ).stdout
        
        # Check for breaking changes
        breaking = any(
            keyword in changelog.lower()
            for keyword in ['breaking', 'deprecated', 'removed', 'incompatible']
        )
        
        return Update(
            project=project,
            current_version=self.get_current_version(project),
            latest_version=self.get_latest_version(project),
            breaking_changes=breaking,
            changelog=changelog
        )
    
    def apply_updates(self, updates: List[Update], 
                     auto_merge: bool = False) -> None:
        """Apply updates with safety checks"""
        for update in updates:
            print(f"\n{'='*60}")
            print(f"Updating {update.project}")
            print(f"  {update.current_version} -> {update.latest_version}")
            
            if update.breaking_changes:
                print("  ⚠️  Breaking changes detected!")
                if not auto_merge:
                    response = input("  Continue? (y/N): ")
                    if response.lower() != 'y':
                        print("  Skipped")
                        continue
            
            # Apply update
            self.update_project(update.project)
            
            # Run tests
            if self.run_tests(update.project):
                print("  ✓ Tests passed")
            else:
                print("  ✗ Tests failed - rolling back")
                self.rollback_project(update.project)
    
    def update_project(self, project: str) -> None:
        """Update single project"""
        subprocess.run(
            ['git', 'pull', '--rebase'],
            cwd=f"{self.aosp_dir}/{project}"
        )
    
    def rollback_project(self, project: str) -> None:
        """Rollback project update"""
        subprocess.run(
            ['git', 'reset', '--hard', 'HEAD@{1}'],
            cwd=f"{self.aosp_dir}/{project}"
        )
```

## Dependency Caching

### Distributed Cache Manager

```python
# scripts/cache_manager.py
import redis
import hashlib
from pathlib import Path
from typing import Optional

class DistributedCacheManager:
    """Distributed caching for AOSP dependencies"""
    
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.cache_prefix = "aosp:dep:"
        
    def cache_key(self, name: str, version: str) -> str:
        """Generate cache key"""
        return f"{self.cache_prefix}{name}:{version}"
    
    def get_cached(self, name: str, version: str) -> Optional[bytes]:
        """Get dependency from cache"""
        key = self.cache_key(name, version)
        return self.redis.get(key)
    
    def cache_dependency(self, name: str, version: str, 
                        data: bytes, ttl: int = 86400) -> None:
        """Cache dependency with TTL"""
        key = self.cache_key(name, version)
        self.redis.setex(key, ttl, data)
        
        # Store metadata
        meta_key = f"{key}:meta"
        metadata = {
            "size": len(data),
            "checksum": hashlib.sha256(data).hexdigest()
        }
        self.redis.hset(meta_key, mapping=metadata)
    
    def verify_cached(self, name: str, version: str, 
                     expected_checksum: str) -> bool:
        """Verify cached dependency integrity"""
        key = self.cache_key(name, version)
        meta_key = f"{key}:meta"
        
        cached_checksum = self.redis.hget(meta_key, "checksum")
        return cached_checksum and cached_checksum.decode() == expected_checksum
```

## Best Practices

1. **Automated Fetching**: Fetch dependencies automatically during setup
2. **Version Locking**: Use lock files for reproducible builds
3. **Security Scanning**: Scan dependencies regularly for vulnerabilities
4. **Smart Updates**: Analyze updates before applying
5. **Distributed Caching**: Use Redis for team-wide dependency caching
6. **Checksum Verification**: Always verify binary integrity
7. **Rollback Strategy**: Have automated rollback for failed updates

## Conclusion

Automated dependency management reduces manual work, improves security, and ensures reproducible builds across the entire development team.
