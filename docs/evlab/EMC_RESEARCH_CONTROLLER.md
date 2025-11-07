# Controller Build Pipeline Research

**Issue:** #2
**Epic:** #1 - External Management Cluster (EMC) Planning
**Date:** 2025-11-07
**Status:** Research Complete

## Executive Summary

This document provides a comprehensive analysis of the existing `hhfab` controller build pipeline. The goal is to understand how Hedgehog Fabricator builds and packages the controller appliance so we can mirror this pattern for the planned External Management Cluster (EMC).

**Key Finding:** The system generates **bootable ISO/USB images** containing Flatcar Linux with embedded installation payloads, NOT VMware OVA files. The build process creates self-contained installation media with automated provisioning via cloud-init (Butane/Ignition).

## Table of Contents

1. [Build Command Overview](#build-command-overview)
2. [Build Pipeline Architecture](#build-pipeline-architecture)
3. [Detailed Component Analysis](#detailed-component-analysis)
4. [Artifact Lifecycle](#artifact-lifecycle)
5. [Dependency Management](#dependency-management)
6. [Installation Process](#installation-process)
7. [Risks and Unknowns](#risks-and-unknowns)
8. [Recommendations for EMC](#recommendations-for-emc)

---

## Build Command Overview

### Entry Point

**File:** `cmd/hhfab/main.go:731-762`

### CLI Usage

```bash
hhfab build [OPTIONS]

Options:
  --mode, -m              Build mode: iso (default), usb, or manual
  --controls              Build control node(s) (default: true)
  --gateways              Build gateway node(s) (default: true)
  --join-token, -j        Join token for the cluster
  --o11y-targets          Inject extra observability targets
  --hydrate-mode          Set hydrate mode (default: if-not-present)
```

### Build Modes

| Mode | Description | Output Files |
|------|-------------|--------------|
| **iso** | Bootable ISO image (default) | `.iso` file |
| **usb** | Bootable USB image with GPT | `.img` file |
| **manual** | Separate archive + ignition | `.tgz` + `.ign` files |

### Example Commands

```bash
# Build controller ISO (most common)
hhfab build --mode iso

# Build USB image
hhfab build --mode usb

# Build only controls (skip gateways)
hhfab build --controls --no-gateways

# Build with custom join token
hhfab build --join-token "K10abc..."
```

---

## Build Pipeline Architecture

### High-Level Flow

```
User Input (fab.yaml + wiring diagrams)
         ↓
    hhfab build
         ↓
Load & Validate Config
         ↓
Create Artifact Downloader
         ↓
┌────────┴────────┐
│                 │
Build Controls   Build Gateways/Nodes
│                 │
└────────┬────────┘
         ↓
  Result Directory
  (workdir/result/)
```

### Key Components

1. **Configuration Loader** - Reads `fab.yaml` and wiring diagrams
2. **Artifact Downloader** - Fetches dependencies from OCI registry
3. **Build Engine** - Assembles installation bundles
4. **Image Builder** - Creates bootable ISO/USB images
5. **Installation Runtime** - Embedded binary that runs on target system

### File References

| Component | File Path | Key Functions |
|-----------|-----------|---------------|
| CLI Entry | `cmd/hhfab/main.go` | Line 731 (build command) |
| Build Orchestration | `pkg/hhfab/cmdbuild.go` | `Build()` (line 29) |
| Config Loading | `pkg/hhfab/cmdconfig.go` | `load()`, `loadHydrateValidate()` |
| Control Builder | `pkg/fab/recipe/control_build.go` | `ControlInstallBuilder.Build()` (line 70) |
| Build Engine | `pkg/fab/recipe/build.go` | `buildInstall()` (line 65), `buildUSBImage()` (line 194) |
| Artifact Download | `pkg/artificer/downloader.go` | `FromORAS()` (line 73), `GetOCI()` (line 220) |
| Installation Runtime | `cmd/hhfab-recipe/main.go` | `install` command |
| Control Install Logic | `pkg/fab/recipe/control_install.go` | `ControlInstall.Run()` (line 43) |

---

## Detailed Component Analysis

### 1. Build Orchestration Layer

**File:** `pkg/hhfab/cmdbuild.go:29-137`

The `Build()` function orchestrates the entire build process:

```go
func Build(ctx context.Context, workDir, cacheDir string, opts BuildOpts) error {
    // 1. Load configuration
    c, err := load(ctx, workDir, cacheDir, LoadOpts{
        HydrateMode: opts.HydrateMode,
    })

    // 2. Create artifact downloader
    d, err := artificer.NewDownloader(cacheDir, c.Fab.Spec.Config.Repo,
                                      c.Fab.Spec.Config.Prefix)

    // 3. Build control nodes
    if opts.BuildControls {
        for _, control := range c.Controls {
            builder := &recipe.ControlInstallBuilder{
                WorkDir:    resultDir,
                Fab:        c.Fab,
                Control:    control,
                Mode:       opts.BuildMode,
                Downloader: d,
            }
            builder.Build(ctx)
        }
    }

    // 4. Build gateway/fabric nodes
    if opts.BuildGateways {
        // Similar pattern for nodes
    }
}
```

**Key Observations:**

- Builds each control/gateway independently
- Results stored in `workdir/result/` directory
- Supports building subsets (controls only, gateways only)
- Downloader credentials from Docker config (`~/.docker/config.json`)

### 2. Artifact Naming Convention

All build artifacts follow a consistent naming pattern:

```
{type}--{name}--{suffix}
```

**Examples:**

```
control--ctrl-1--install/           # Installation directory
control--ctrl-1--install.tgz        # Archive (manual mode)
control--ctrl-1--install.ign        # Ignition config
control--ctrl-1--install-usb.img    # USB disk image
control--ctrl-1--install-usb.iso    # ISO image (most common)
control--ctrl-1--install.inhash     # Build cache hash
```

**File:** `pkg/fab/recipe/build.go:75-79`

### 3. Payload Assembly (Dependencies)

**File:** `pkg/fab/recipe/control_build.go:89-231`

The `addPayload()` method downloads and stages all required components:

#### Core Platform Components

| Component | Version | Artifact Reference | Files Included |
|-----------|---------|-------------------|----------------|
| **K3s** | v1.34.1-k3s1 | `fabricator/k3s-airgap` | `k3s` binary, `k3s-install.sh`, `k3s-airgap-images-amd64.tar.gz` |
| **K9s** | v0.50.15 | `fabricator/k9s` | `k9s` binary |
| **Zot Registry** | v2.1.9 | `fabricator/zot-airgap` | Zot container image + Helm chart |
| **Cert-Manager** | v1.18.2 | `fabricator/cert-manager-airgap` | Cert-manager image + Helm chart |
| **Flatcar Updater** | v4230.2.4 | `fabricator/flatcar-update` | `flatcar_production_update.gz` |
| **Bash Completion** | v2.16.0 | `fabricator/bash-completion` | Completion scripts |

**Code Reference:** Lines 91-169

#### Configuration Files

- **fab.yaml** - Fabricator configuration with control nodes
- **include.yaml** - Wiring diagram CRDs (fabric/gateway objects)

**Code Reference:** Lines 171-190

#### CLI Tools

- **hhfctl** - Fabric controller CLI
- **hhfabctl** - Fabricator controller CLI

**Code Reference:** Lines 192-209

#### Airgap Artifacts (Optional)

If registry mode is `airgap`, downloads ALL container images for offline installation:

```go
var AirgapArtifactsBase = []comp.ListOCIArtifacts{
    flatcar.Artifacts,      // Flatcar system containers
    certmanager.Artifacts,  // Cert-manager images
    zot.Artifacts,          // Zot registry images
    reloader.Artifacts,     // Config reloader
    fabric.Artifacts,       // Fabric controller + agent
    ntp.Artifacts,          // NTP service
    controlproxy.Artifacts, // Control node proxy
    f8r.Artifacts,          // Fabricator controller
    alloy.Artifacts,        // Observability agent
}
```

Plus gateway artifacts if gateways enabled:

```go
var AirgapArtifactsGateway = []comp.ListOCIArtifacts{
    gateway.Artifacts,      // Gateway controller + agent
}
```

**Code Reference:** Lines 211-228

**File:** `pkg/fab/comp/comp.go` - Component definitions

### 4. Ignition Configuration Generation

**File:** `pkg/fab/recipe/control_butane.tmpl.yaml`

The system uses Butane templates to generate Ignition configs for cloud-init:

#### Template Variables

- `Hostname` - Control node hostname
- `ManagementInterface` / `ManagementIP` - Management network config
- `ExternalInterface` / `ExternalIP` / `ExternalGateway` / `ExternalDNS` - External network
- `ControlVIP` - Control virtual IP for HA
- `DummyInterface` - Dummy interface for routing
- `PasswordHash` - Hashed password for users
- `SSHAuthorizedKeys` - SSH public keys
- `AutoInstall` - Path to auto-install directory

#### Auto-Install Systemd Unit

The critical piece that triggers automatic installation:

```yaml
- name: hhfab-install.service
  enabled: true
  contents: |
    [Unit]
    Description="Firstboot installation program for Hedgehog Fabricator"
    ConditionPathExists=!/opt/hedgehog/.install
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=oneshot
    ExecStart={{ .AutoInstall }}/hhfab-recipe install -v
    WorkingDirectory={{ .AutoInstall }}
    StandardOutput=journal+console
    StandardError=journal+console

    [Install]
    WantedBy=multi-user.target
```

**Key Points:**

- Runs once: `ConditionPathExists=!/opt/hedgehog/.install`
- Waits for network: `After=network-online.target`
- Executes embedded `hhfab-recipe` binary
- Output to console and journal

**Code Reference:** `pkg/fab/recipe/control_build.go:236-275`

### 5. Image Creation Process

**File:** `pkg/fab/recipe/build.go:194-397`

The `buildUSBImage()` function creates bootable disk images.

#### Base Image Download

Downloads Flatcar USB root from OCI registry:

**Artifact:** `fabricator/control-usb-root:v4230.2.4-hh1`

**Contents:**
- `boot/` - Bootloader files (GRUB/systemd-boot)
- `EFI/` - EFI boot binaries
- `images/efi.img` - EFI boot image
- `flatcar_production_image.bin.bz2` - Base OS image
- `flatcar_production_pxe_image.cpio.gz` - Initramfs
- `flatcar_production_pxe.vmlinuz` - Linux kernel

**Code Reference:** Lines 210-219

#### USB Image Structure (GPT Disk)

When `--mode usb`:

```
Disk: 9.5 GB raw disk image
├── Partition 1: ESP (EFI System Partition)
│   ├── Filesystem: FAT32
│   ├── Size: 500 MB
│   ├── Label: "ESP"
│   └── Contents:
│       ├── /EFI/               # EFI boot binaries
│       ├── /boot/              # Bootloader config
│       └── /images/            # Boot images
│           └── efi.img
└── Partition 2: HH-MEDIA (Installation Media)
    ├── Filesystem: FAT32
    ├── Size: 9 GB
    ├── Label: "HH-MEDIA"
    └── Contents:
        ├── flatcar_production_image.bin.bz2    # OS image
        ├── flatcar_production_pxe_image.cpio.gz # Initramfs
        ├── flatcar_production_pxe.vmlinuz      # Kernel
        ├── ignition.json                       # Cloud-init config
        └── control--{name}--install/           # Installation bundle
            ├── hhfab-recipe                    # Embedded installer binary
            ├── recipe.yaml                     # Install config
            ├── fab.yaml                        # Fabricator config
            ├── include.yaml                    # Wiring diagrams
            ├── k3s                             # K3s binary
            ├── k3s-install.sh                  # K3s installer
            ├── k3s-airgap-images-amd64.tar.gz  # K3s container images
            ├── zot-*.tar                       # Zot container images
            ├── certmanager-*.tar               # Cert-manager images
            └── ... (other components)
```

**Code Reference:** Lines 228-283

#### ISO Image Structure (ISO 9660)

When `--mode iso` (default):

```
ISO 9660 Filesystem with El Torito Boot
├── Label: "HH-MEDIA"
├── Boot: El Torito with EFI image (images/efi.img)
├── Extensions: Rock Ridge (POSIX attributes)
└── Contents: (Same as HH-MEDIA partition above)
    ├── /images/efi.img                         # EFI boot image
    ├── flatcar_production_image.bin.bz2
    ├── flatcar_production_pxe_image.cpio.gz
    ├── flatcar_production_pxe.vmlinuz
    ├── ignition.json
    └── control--{name}--install/
        └── ... (installation bundle)
```

**Code Reference:** Lines 284-306

**Implementation Details:**

- Uses `diskfs` Go library for disk/ISO creation
- No QEMU or Packer involved - programmatic image creation
- ISO created with `xorriso` command-line tool
- Boot configured for UEFI systems

---

## Artifact Lifecycle

### 1. Build Cache

**File:** `pkg/fab/recipe/build.go:81-94, 168-170`

#### Hash File

**Purpose:** Skip rebuilds if configuration unchanged

**File:** `{type}--{name}--install.inhash`

**Content:** Base64-encoded SHA256 hash of:
- Fabricator version
- `fab.yaml` content
- `include.yaml` content (wiring diagrams)
- Build mode

**Code:**
```go
func (b *buildInstaller) hash() (string, error) {
    h := sha256.New()
    h.Write([]byte(version.Version))
    h.Write(fabYaml)
    h.Write(includeYaml)
    h.Write([]byte(b.mode))
    return base64.StdEncoding.EncodeToString(h.Sum(nil)), nil
}
```

**Build Skip Logic:**

1. Read existing `.inhash` file
2. Compare with current config hash
3. If match AND all expected artifacts exist → skip build
4. Otherwise → proceed with build

**Benefit:** Significantly speeds up repeated builds during development

### 2. Download Cache

**File:** `pkg/artificer/downloader.go`

#### Cache Directory Structure

**Location:** `~/.hhfab-cache/v1/` (default) or `HHFAB_CACHE_DIR`

```
{cacheDir}/v1/
├── fabricator_k3s-airgap@v1.34.1-k3s1.oras/
│   ├── k3s
│   ├── k3s-install.sh
│   └── k3s-airgap-images-amd64.tar.gz
├── fabricator_zot-airgap@v2.1.9.oras/
│   ├── zot-image.tar
│   └── zot-chart.tgz
├── fabricator_cert-manager-airgap@v1.18.2.oras/
│   ├── certmanager-image.tar
│   └── certmanager-chart.tgz
├── fabricator_control-usb-root@v4230.2.4-hh1.oras/
│   ├── boot/
│   ├── EFI/
│   ├── images/
│   ├── flatcar_production_image.bin.bz2
│   ├── flatcar_production_pxe_image.cpio.gz
│   └── flatcar_production_pxe.vmlinuz
└── ... (other artifacts)
```

**Cache Key Format:**

```
{repo-prefix}_{artifact-name}@{version}.oras/
```

**Download Process:**

1. Check if artifact exists in cache
2. If missing or force refresh → download from OCI registry
3. Extract to cache directory
4. Copy to installation bundle

**Code Reference:** `pkg/artificer/downloader.go:73-100` (FromORAS method)

#### Progress Display

Downloads show progress bars using `mpb` library:

```
Downloading K3s airgap...
[====================] 1.2 GB / 1.2 GB  100%  ETA: 0s

Downloading Zot registry...
[=========>          ] 45 MB / 120 MB   38%  ETA: 12s
```

**Concurrent Downloads:** Up to 4 parallel downloads

### 3. Temporary Directories

During build, temporary directories are created:

**Installation Directory:**
```
workdir/result/{type}--{name}--install/
```

Created fresh for each build (deleted if exists).

**Disk Image Staging:**

For USB/ISO modes, creates temporary mount points:

```
/tmp/hhfab-{random}/
├── esp/        # ESP partition mount
└── media/      # HH-MEDIA partition mount
```

Cleaned up after image creation.

### 4. Final Artifact Locations

**File:** `pkg/hhfab/cmdbuild.go:67`

All artifacts written to:

```
{workdir}/result/
```

**Typical Result Directory:**

```
workdir/result/
├── control--ctrl-1--install/           # Installation bundle directory
├── control--ctrl-1--install.ign        # Ignition config
├── control--ctrl-1--install-usb.iso    # Bootable ISO (most common)
├── control--ctrl-1--install.inhash     # Build cache hash
├── gateway--gw-1--install/             # If gateways built
└── gateway--gw-1--install-usb.iso
```

**Artifact Sizes:**

- ISO/IMG files: ~9.5 GB (contains full Flatcar OS + all dependencies)
- Installation directory: ~2-3 GB (compressed artifacts)
- Ignition file: ~10-20 KB (JSON config)

### 5. Checksum Generation

**Current State:** No automatic checksums for final artifacts

**Recommendation:** Users should generate SHA256 checksums:

```bash
cd workdir/result/
sha256sum *.iso > checksums.txt
```

**Future Enhancement:** Could add automatic checksum generation in `buildInstall()` after artifact creation.

### 6. Distribution Mechanisms

**Current State:** No built-in distribution

**Manual Options:**

1. **Physical Media:** Burn ISO to USB/CD
   ```bash
   sudo dd if=control--ctrl-1--install-usb.iso of=/dev/sdX bs=4M status=progress
   ```

2. **VM Import:** Upload ISO to hypervisor (vCenter, Proxmox, etc.)

3. **Network Boot:** Serve ISO via HTTP (requires manual setup)

**Potential Future Enhancements:**

- Upload to S3/artifact registry
- Generate OVA packages for VMware
- PXE boot server integration
- Torrent/P2P distribution for large deployments

**Code Reference:** `pkg/artificer/oci.go` has `UploadOCIArchive()` function but not used for installers currently

---

## Dependency Management

### 1. Version Source of Truth

**File:** `pkg/fab/versions.go`

All component versions defined in single location:

```go
var Versions = fabapi.Versions{
    Platform: fabapi.PlatformVersions{
        K3s:               "v1.34.1-k3s1",
        Zot:               "v2.1.9",
        CertManager:       "v1.18.2",
        K9s:               "v0.50.15",
        Toolbox:           "v0.7.2",
        NTP:               "v0.0.4",
        Alloy:             "v1.11.2",
        BashCompletion:    "v2.16.0",
    },
    Fabricator: fabapi.FabricatorVersions{
        Controller:     version.FabricatorVersion,  // From build
        ControlUSBRoot: "v4230.2.4-hh1",           // Flatcar base
        Flatcar:        "v4230.2.4",               // Flatcar OS
    },
    Fabric: fabapi.FabricVersions{
        Controller: "v0.94.3",
        Agent:      "v0.94.3",
        NOS: map[fmeta.NOSType]meta.Version{
            fmeta.NOSTypeSONiCBCMVS:    "v4.5.0",
            fmeta.NOSTypeSONiCONIEVS:   "v4.5.0",
            fmeta.NOSTypeSONiCCLS:      "v0.12.0",
        },
    },
    Gateway: fabapi.GatewayVersions{
        Controller: "v0.25.0",
        Agent:      "v0.25.0",
    },
}
```

### 2. Version Hydration

**File:** `pkg/hhfab/cmdconfig.go:358`

During config load, the system:

1. **Merges** default versions with user overrides from `fab.yaml`
2. **Validates** version constraints and compatibility
3. **Populates** `Fabricator.Status.Versions` for runtime use

**User Override Example (fab.yaml):**

```yaml
apiVersion: fabricator.githedgehog.com/v1beta1
kind: Fabricator
spec:
  config:
    versions:
      platform:
        k3s: v1.35.0-k3s1  # Override default
```

### 3. External Registry

**Default Registry:** `ghcr.io` (GitHub Container Registry)

**Default Prefix:** `githedgehog`

**Can be customized:**

```bash
hhfab init --registry-repo docker.io --registry-prefix myorg
```

**Stored in:** `fab.yaml`

```yaml
spec:
  config:
    repo: ghcr.io
    prefix: githedgehog
```

### 4. Artifact Reference Pattern

All artifacts follow OCI reference format:

```
{repo}/{prefix}/{artifact-name}:{version}
```

**Examples:**

```
ghcr.io/githedgehog/fabricator/k3s-airgap:v1.34.1-k3s1
ghcr.io/githedgehog/fabricator/zot-airgap:v2.1.9
ghcr.io/githedgehog/fabricator/control-usb-root:v4230.2.4-hh1
ghcr.io/githedgehog/fabric/controller:v0.94.3
ghcr.io/githedgehog/gateway/controller:v0.25.0
```

### 5. Component Definitions

Each component has an artifact definition:

**File:** `pkg/fab/comp/{component}/{component}.go`

**Example: K3s Component**

```go
package k3s

const (
    ArtifactK3s         = "k3s"
    ArtifactK3sInstall  = "k3s-install.sh"
    ArtifactK3sAirgap   = "k3s-airgap-images-amd64.tar.gz"
)

var Artifacts = comp.NewArtifactDefinition(
    "k3s-airgap",
    []comp.Artifact{
        {Name: ArtifactK3s, Mode: 0o755},
        {Name: ArtifactK3sInstall, Mode: 0o755},
        {Name: ArtifactK3sAirgap, Mode: 0o644},
    },
)
```

**File References:**

- `pkg/fab/comp/k3s/k3s.go`
- `pkg/fab/comp/zot/zot.go`
- `pkg/fab/comp/certmanager/certmanager.go`
- `pkg/fab/comp/fabric/fabric.go`
- `pkg/fab/comp/gateway/gateway.go`

### 6. Dependency Tree

```
Control Node Installation
├── Platform Components
│   ├── K3s (Kubernetes)
│   │   └── K3s Airgap Images (container images)
│   ├── Cert-Manager (TLS certificates)
│   ├── Zot (OCI registry)
│   ├── K9s (Kubernetes CLI)
│   ├── NTP (time sync)
│   └── Alloy (observability)
├── Fabricator
│   ├── Fabricator Controller
│   ├── Fabricator CLI (hhfabctl)
│   └── Control Proxy
├── Fabric
│   ├── Fabric Controller
│   ├── Fabric Agent (for switches)
│   ├── Fabric CLI (hhfctl)
│   └── Switch NOS images
└── Gateway (optional)
    ├── Gateway Controller
    └── Gateway Agent
```

### 7. Airgap Mode

**Purpose:** Fully offline installations

**How it works:**

1. During build, download ALL container images for all components
2. Store as OCI archives in installation bundle
3. At install time, push all images to local Zot registry
4. Kubernetes pulls from local registry instead of internet

**Enabled in fab.yaml:**

```yaml
spec:
  config:
    registryMode: airgap  # vs. "proxy" or "upstream"
```

**Impact on Build:**

- **Size:** Significantly larger ISO (~12-15 GB vs ~9.5 GB)
- **Time:** Much longer build time (downloads all images)
- **Benefit:** Can install without internet connectivity

**Code Reference:** `pkg/fab/recipe/control_build.go:211-228`

---

## Installation Process

### 1. Boot Sequence

**Phase 1: Initial Boot**

1. System boots from ISO/USB media
2. UEFI firmware loads EFI boot image (`images/efi.img`)
3. Bootloader (GRUB/systemd-boot) loads kernel and initramfs
4. Kernel parameters include ignition config location:
   ```
   ignition.config.url=label:HH-MEDIA:/ignition.json
   ```

**Phase 2: Flatcar Ignition**

5. Flatcar initramfs reads ignition config from media
6. Applies network configuration (interfaces, IPs, routes)
7. Provisions users, SSH keys, systemd units
8. Installs Flatcar OS to disk (from `flatcar_production_image.bin.bz2`)
9. Reboots into installed system

**Phase 3: Auto-Install**

10. System boots from disk
11. Systemd starts `hhfab-install.service`
12. Service checks condition: `/opt/hedgehog/.install` doesn't exist
13. Executes: `hhfab-recipe install -v` from installation bundle

**File References:**

- Ignition template: `pkg/fab/recipe/control_butane.tmpl.yaml`
- Install service definition: Lines 236-275

### 2. Installation Runtime

**Binary:** `hhfab-recipe` (embedded in installation bundle)

**Source:** `cmd/hhfab-recipe/main.go`

**Commands:**

- `install` - First-time installation
- `upgrade` - Upgrade existing installation

### 3. Control Node Installation Steps

**File:** `pkg/fab/recipe/control_install.go:43-117`

The `ControlInstall.Run()` method executes these steps:

#### Step 1: Verify Network (Lines 46-50)

```go
// Check management IP is configured
verifyInterfaceIP(control.Management.Interface, control.Management.IP)

// Check control VIP is configured
verifyInterfaceIP(dummyInterface, controlVIP)
```

**Retry Logic:** 6 retries, 30s delay

#### Step 2: Install K3s (Lines 52-55)

```bash
# Copy K3s binary
cp k3s /opt/bin/k3s

# Copy airgap images
cp k3s-airgap-images-amd64.tar.gz /var/lib/rancher/k3s/agent/images/

# Copy Helm charts
cp zot-chart.tgz /var/lib/rancher/k3s/server/static/charts/
cp certmanager-chart.tgz /var/lib/rancher/k3s/server/static/charts/

# Generate K3s config
cat > /etc/rancher/k3s/config.yaml <<EOF
node-ip: {management-ip}
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san: {control-vip}
tls-san: {management-ip}
disable:
  - traefik
  - servicelb
  - local-storage
EOF

# Run K3s installer
INSTALL_K3S_SKIP_DOWNLOAD=true ./k3s-install.sh

# Wait for node ready
kubectl wait --for=condition=ready node/{hostname}
```

#### Step 3: Install Cert-Manager (Lines 64-66)

```bash
# Deploy via Helm
helm install cert-manager /var/lib/rancher/k3s/server/static/charts/certmanager-chart.tgz \
  --namespace cert-manager --create-namespace

# Wait for webhook ready
kubectl wait --for=condition=available deployment/cert-manager-webhook \
  -n cert-manager --timeout=300s
```

#### Step 4: Install Fabric CA (Lines 68-71)

```bash
# Generate self-signed CA certificate
openssl req -x509 -newkey rsa:4096 -keyout ca-key.pem -out ca-cert.pem \
  -days 3650 -nodes -subj "/CN=Hedgehog Fabric CA"

# Create Kubernetes secret
kubectl create secret tls fabric-ca -n default \
  --cert=ca-cert.pem --key=ca-key.pem

# Create ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: fabric-ca
spec:
  ca:
    secretName: fabric-ca
EOF

# Install CA cert to system trust
cp ca-cert.pem /etc/ssl/certs/hedgehog-fabric-ca.pem
update-ca-certificates
```

#### Step 5: Install Zot Registry (Lines 73-75)

```bash
# Generate registry users
for user in admin; do
  htpasswd -Bbn $user $password >> htpasswd
done

# Create secret
kubectl create secret generic zot-auth -n zot --from-file=htpasswd

# Deploy via Helm
helm install zot /var/lib/rancher/k3s/server/static/charts/zot-chart.tgz \
  --namespace zot --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=100Gi

# Wait for deployment ready
kubectl wait --for=condition=available deployment/zot -n zot --timeout=300s

# Test registry endpoint
curl -u admin:$password https://zot.zot.svc.cluster.local:5000/v2/_catalog
```

#### Step 6: Upload Airgap Artifacts (Lines 84-88)

**If airgap mode enabled:**

```bash
# For each OCI archive in installation bundle
for archive in *.oci.tar; do
  # Push to local Zot registry
  skopeo copy oci-archive:$archive \
    docker://zot.zot.svc.cluster.local:5000/$(image-name):$(tag) \
    --dest-creds admin:$password --dest-tls-verify=false
done
```

**File Reference:** `pkg/fab/recipe/control_install.go:84-88`

#### Step 7: Pre-cache Zot (Lines 90-92)

```bash
# Pull critical images into Zot for faster deployment
for image in fabric-controller fabric-agent gateway-controller; do
  crictl pull zot.zot.svc.cluster.local:5000/$image:$version
done
```

#### Step 8: Install Fabricator Controller (Lines 94-96)

```bash
# Apply Fabricator CRDs
kubectl apply -f fabricator-crds.yaml

# Deploy Fabricator controller
kubectl apply -f fabricator-controller.yaml

# Apply fab.yaml config
kubectl apply -f fab.yaml

# Wait for Fabricator ready
kubectl wait --for=condition=available deployment/fabricator-controller \
  -n fabricator-system --timeout=300s
```

#### Step 9: Setup Time Sync (Lines 103-105)

```bash
# Configure systemd-timesyncd to use control VIP
cat > /etc/systemd/timesyncd.conf <<EOF
[Time]
NTP={control-vip}
FallbackNTP=time.cloudflare.com
EOF

# Restart time sync service
systemctl restart systemd-timesyncd
```

#### Step 10: Install CLIs (Lines 107-109)

```bash
# Install hhfctl (Fabric CLI)
cp hhfctl /usr/local/bin/hhfctl
chmod +x /usr/local/bin/hhfctl

# Install hhfabctl (Fabricator CLI)
cp hhfabctl /usr/local/bin/hhfabctl
chmod +x /usr/local/bin/hhfabctl
```

#### Step 11: Install Wiring (Lines 111-113)

```bash
# Apply wiring diagrams from include.yaml
kubectl apply -f include.yaml

# Wait for switch profiles to be created
kubectl wait --for=condition=ready switchprofile --all --timeout=300s

# Verify all wiring objects created
kubectl get switches,connections,servers -A
```

#### Step 12: Mark Installation Complete (Lines 161-163)

```bash
# Create completion marker
mkdir -p /opt/hedgehog
echo "complete" > /opt/hedgehog/.install
```

**Effect:** Prevents `hhfab-install.service` from running again on reboot

### 4. Installation Logs

**Console Output:** All output goes to console and systemd journal

**View Logs:**

```bash
# View installation service logs
journalctl -u hhfab-install.service

# Follow live installation
journalctl -fu hhfab-install.service

# View all Hedgehog-related services
journalctl -t hhfab-recipe
```

### 5. Installation Failure Handling

**Current Behavior:**

- Service fails if any step errors
- Partial state may exist (e.g., K3s installed but Zot failed)
- Marker file NOT created on failure
- Service will retry on next boot

**Recovery:**

```bash
# Remove partial installation
rm -rf /var/lib/rancher/k3s
rm -rf /etc/rancher/k3s

# Reboot to retry
reboot
```

**Future Enhancement:** Add rollback logic for failed installations

---

## Risks and Unknowns

### Critical Observations

#### 1. ISO vs. OVA Terminology

**Finding:** The system generates **bootable ISO/USB images**, NOT VMware OVA files.

**Impact for EMC:**

- Issue #2 and Epic #1 mention "OVA artifacts"
- No OVA generation exists in current codebase
- ISOs can boot VMs but aren't packaged OVAs

**Recommendation:**

- Clarify requirements: Do we need actual OVAs or are ISOs sufficient?
- If OVAs needed, consider two approaches:
  1. Manual: Boot ISO in VM, install, export VMDK, create OVF descriptor
  2. Automated: Add Packer pipeline to automate VM creation from ISO

#### 2. Build Tool Architecture

**Finding:** Images created programmatically via `diskfs` library, NOT via QEMU/Packer.

**Implications:**

- No VM snapshots or pre-warmed images
- Every installation runs full provisioning process
- First boot takes 10-15 minutes for full K3s cluster setup

**For EMC:**

- Could use same pattern (ISO-based)
- Or add Packer to create pre-installed VM images
- Trade-off: Flexibility vs. speed

#### 3. Cache and Artifact Verification

**Risk:** No checksum verification of cached artifacts beyond OCI registry digests.

**Potential Issues:**

- Cache corruption could cause build failures
- Difficult to debug mismatched artifact versions
- No integrity checking of final ISO/IMG files

**Mitigation:**

- Add SHA256 checksum generation for final artifacts
- Verify downloaded artifacts against published manifests
- Implement cache validation/repair commands

#### 4. Partial Installation State

**Risk:** If `hhfab-recipe install` fails midway, system is in partial state.

**Current Behavior:**

- Marker file only created on success
- Service retries on next boot
- No automatic rollback

**Problems:**

- K3s might be installed but Zot failed
- Difficult to determine what succeeded/failed
- Manual cleanup required

**Recommendation:**

- Add installation state tracking (journal file)
- Implement rollback for failed steps
- Create pre-flight checks before starting
- Better error messages with recovery instructions

#### 5. Network Timing Dependencies

**Risk:** Installation assumes network is ready, but has retry logic (6 attempts, 30s each).

**Potential Issues:**

- DHCP might not be configured yet
- Management network might be misconfigured
- Control VIP might not route correctly

**Current Mitigation:**

- Retry loops with delays
- Ignition configures networking before installation

**Enhancement Needed:**

- More robust network validation
- Fallback to manual configuration prompt
- Network diagnostics in failure cases

#### 6. Large Image Sizes

**Observation:** ISO/IMG files are ~9.5 GB (standard) or ~12-15 GB (airgap).

**Implications:**

- Slow to distribute over network
- May not fit on some USB drives
- Large download times

**For EMC:**

- Could be even larger with additional services (Gitea, ArgoCD, Prometheus, Grafana)
- Consider compression or split archives
- Evaluate cloud-init pull model (download on first boot) vs. embedded

#### 7. Flatcar OS Dependency

**Risk:** System is tied to specific Flatcar version (`v4230.2.4-hh1`).

**Implications:**

- Flatcar updates require rebuilding base image
- Custom Flatcar build maintained by Hedgehog (`-hh1` suffix)
- Upgrades must coordinate Flatcar + component versions

**For EMC:**

- Same dependency
- Need process for updating Flatcar base
- Consider testing with upstream Flatcar (remove custom build)

### Unknowns to Investigate

#### 1. Multi-Control HA Behavior

**Question:** How does HA clustering work with multiple control nodes?

**Current Code:** Supports multiple control nodes in config, but unclear:

- Is K3s cluster formed automatically?
- How is initial cluster bootstrap coordinated?
- What happens if control-1 installs but control-2 fails?

**Investigation Needed:**

- Review `pkg/fab/comp/k3s/install.go` for join token handling
- Test multi-control deployment
- Document HA setup procedure

#### 2. Upgrade Process

**Question:** How do upgrades work? What can be upgraded vs. requires reinstall?

**Code Exists:** `hhfab-recipe upgrade` command, but:

- Upgrade constraints defined in `pkg/fab/recipe/install_upgrade.go`
- Not clear which components can be upgraded in place
- Flatcar OS upgrades via separate mechanism

**Investigation Needed:**

- Test upgrade path from v0.94.0 → v0.94.3
- Document supported upgrade scenarios
- Identify breaking changes that require reinstall

#### 3. Gateway vs. Control Build Differences

**Question:** How do gateway builds differ from control builds?

**Current Understanding:**

- Similar pattern: `NodeInstallBuilder` vs. `ControlInstallBuilder`
- Gateway runs subset of services
- Less clear what exact differences are

**Investigation Needed:**

- Compare `control_build.go` vs. `node_build.go`
- Document gateway-specific components
- Understand gateway networking requirements

#### 4. Registry Mode Trade-offs

**Question:** What are the practical differences between airgap/proxy/upstream modes?

**Modes:**

- **airgap:** All images embedded, no internet needed
- **proxy:** Zot proxies to upstream registries
- **upstream:** Kubernetes pulls directly from internet

**Need to document:**

- Size impacts
- Build time impacts
- Installation time impacts
- Operational differences

#### 5. PXE/Network Boot

**Question:** Can the system support network boot instead of ISO/USB?

**Current State:** No PXE boot support detected

**Potential:**

- Serve ISO contents via HTTP
- Use iPXE to boot kernel + initramfs
- Pass ignition config via kernel parameter

**Investigation Needed:**

- Is this a planned feature?
- Would require changes to current architecture?

#### 6. Distribution to Multiple Sites

**Question:** How to distribute builds to multiple data centers/sites?

**No Built-in Solution**

**Options to Consider:**

- Upload ISOs to S3/HTTP server
- OCI registry for ISO artifacts
- Torrent/P2P for large deployments
- USB shipping for airgap sites

**For EMC:**

- May need multi-site deployment
- Document recommended distribution methods

---

## Recommendations for EMC

Based on this research, here are recommendations for building the External Management Cluster (EMC):

### 1. Reuse Existing Build Pattern

**Recommendation:** Use the same ISO-based build architecture for EMC.

**Rationale:**

- Proven approach used by control nodes
- Consistent UX across components
- Leverages existing artifact downloader and caching
- No need to learn new build tools

**Implementation:**

- Create `EMCInstallBuilder` similar to `ControlInstallBuilder`
- Reuse `buildInstall()` and `buildUSBImage()` functions
- Add EMC-specific components to payload

### 2. EMC-Specific Components

**New Components to Add:**

| Component | Purpose | Version (Example) |
|-----------|---------|-------------------|
| **Gitea** | Git server for GitOps repos | v1.22.0 |
| **ArgoCD** | GitOps continuous deployment | v2.13.0 |
| **Prometheus** | Metrics collection | v2.54.0 |
| **Grafana** | Observability dashboards | v11.3.0 |
| **Alertmanager** | Alert routing | v0.27.0 |
| **Loki** | Log aggregation (optional) | v3.2.0 |

**Package as:**

- Follow existing pattern: create `pkg/fab/comp/gitea/`, `pkg/fab/comp/argocd/`, etc.
- Define artifact references in OCI registry
- Add to version definitions in `pkg/fab/versions.go`

### 3. Add EMC Build Mode

**Approach 1: Separate Command**

```bash
hhfab build-emc [OPTIONS]
```

**Approach 2: Extend Existing Command**

```bash
hhfab build --controls --gateways --emc
```

**Recommendation:** Approach 2 (extend existing) for consistency.

### 4. EMC Configuration in fab.yaml

**Add EMC section to Fabricator CRD:**

```yaml
apiVersion: fabricator.githedgehog.com/v1beta1
kind: Fabricator
spec:
  emc:
    enabled: true
    name: emc-1
    management:
      interface: eth0
      ip: 10.10.10.10/24
    external:
      interface: eth1
      ip: 192.168.1.100/24
      gateway: 192.168.1.1
      dns: 8.8.8.8
    services:
      gitea:
        enabled: true
        domain: git.example.com
      argocd:
        enabled: true
        domain: argocd.example.com
      prometheus:
        enabled: true
        retention: 30d
      grafana:
        enabled: true
        domain: grafana.example.com
```

**File:** Create `api/fabricator/v1beta1/emc_types.go`

### 5. EMC Installation Process

**Similar to Control Install:**

1. Verify network configuration
2. Install K3s cluster
3. Install cert-manager
4. Install Fabric CA
5. Install Zot registry (or reuse control's registry)
6. **NEW:** Install Gitea
7. **NEW:** Initialize GitOps repos
8. **NEW:** Install ArgoCD
9. **NEW:** Install Prometheus + Alertmanager
10. **NEW:** Install Grafana
11. **NEW:** Configure dashboards and alerts
12. Mark installation complete

**File:** Create `pkg/fab/recipe/emc_install.go`

### 6. Integration with Control Nodes

**Options:**

**Option A: Standalone EMC**

- EMC has its own K3s cluster
- Separate from control nodes
- Communicates via API

**Option B: Control Node + EMC**

- EMC services run on control K3s cluster
- Reuse existing infrastructure
- Simpler deployment

**Recommendation:** Start with Option A for isolation, consider Option B for small deployments.

### 7. Artifact Naming for EMC

**Follow existing pattern:**

```
emc--{name}--install/
emc--{name}--install.ign
emc--{name}--install-usb.iso
emc--{name}--install.inhash
```

**Example:**

```
emc--emc-1--install-usb.iso
```

### 8. GitOps Repository Initialization

**New Step in EMC Installation:**

```bash
# After Gitea is installed
# Create GitOps repos
curl -X POST http://gitea.gitea.svc.cluster.local:3000/api/v1/user/repos \
  -H "Authorization: token $GITEA_TOKEN" \
  -d '{"name": "hedgehog-config", "private": true}'

# Clone and initialize
git clone http://gitea.gitea.svc.cluster.local:3000/hedgehog/hedgehog-config
cd hedgehog-config
git init
mkdir -p apps/{fabric,gateway,observability}

# Create initial ArgoCD Application manifests
cat > apps/fabric/application.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fabric
spec:
  source:
    repoURL: http://gitea.gitea.svc.cluster.local:3000/hedgehog/hedgehog-config
    path: apps/fabric
  destination:
    server: https://kubernetes.default.svc
EOF

# Commit and push
git add .
git commit -m "Initial GitOps structure"
git push origin master
```

**File:** Add to `pkg/fab/recipe/emc_install.go`

### 9. Observability Pre-Configuration

**Pre-install Grafana dashboards:**

- Fabric metrics dashboard
- Gateway metrics dashboard
- Kubernetes cluster health
- Network topology visualization

**Pre-configure Prometheus:**

- Scrape control nodes
- Scrape gateway nodes
- Scrape fabric switches (if supported)

**Pre-configure alerts:**

- Control node down
- Gateway node down
- Switch agent not ready
- Certificate expiring soon

**Package dashboards and alert rules as ConfigMaps in installation bundle.**

### 10. Testing Strategy

**Build Testing:**

1. Verify EMC ISO builds successfully
2. Check artifact sizes (should be larger than control)
3. Validate all components downloaded to cache
4. Test airgap vs. proxy modes

**Installation Testing:**

1. Boot EMC ISO in VM
2. Verify network configuration applied
3. Verify all services start successfully
4. Verify Gitea accessible
5. Verify ArgoCD accessible
6. Verify Prometheus scraping targets
7. Verify Grafana dashboards load
8. Test GitOps workflow (commit to Gitea → ArgoCD sync)

**Integration Testing:**

1. Deploy control nodes
2. Deploy EMC
3. Verify EMC can monitor control nodes
4. Verify ArgoCD can deploy to control cluster
5. Test end-to-end GitOps flow

### 11. Documentation to Create

**Recommended Docs:**

1. **EMC Architecture Decision Record (ADR)**
   - Why ISO vs. OVA
   - Why standalone vs. integrated
   - Component selection rationale

2. **EMC Build Guide**
   - How to configure fab.yaml for EMC
   - How to run `hhfab build --emc`
   - Artifact verification steps

3. **EMC Installation Guide**
   - Network prerequisites
   - Boot from ISO
   - Post-install verification
   - Troubleshooting common issues

4. **EMC User Guide**
   - Accessing Gitea/ArgoCD/Grafana
   - Creating GitOps repos
   - Deploying applications via ArgoCD
   - Using Grafana dashboards

5. **EMC Upgrade Guide**
   - Supported upgrade paths
   - Backup procedures
   - Rollback procedures

### 12. Branching Strategy

**Recommendation:** Long-lived feature branch

```bash
# Create feature branch from master
git checkout master
git pull upstream master
git checkout -b feature/emc-build

# Make changes iteratively
git commit -m "feat(emc): add EMC types to Fabricator CRD"
git commit -m "feat(emc): add EMC component definitions"
git commit -m "feat(emc): add EMC build command"
git commit -m "feat(emc): add EMC installation logic"

# Keep in sync with upstream
git fetch upstream master
git rebase upstream/master

# When ready, create PR
git push origin feature/emc-build
gh pr create --title "feat: Add External Management Cluster build support"
```

**Keep fork in sync:**

```bash
# Add upstream remote (one time)
git remote add upstream https://github.com/githedgehog/fabricator.git

# Regularly sync
git fetch upstream master
git checkout master
git merge upstream/master
git push origin master
```

### 13. Incremental Implementation

**Phase 1: Foundation (Week 1-2)**

- [ ] Add EMC types to Fabricator CRD
- [ ] Create EMC component definitions (Gitea, ArgoCD, Prometheus, Grafana)
- [ ] Add EMC version tracking
- [ ] Document decisions in ADR

**Phase 2: Build Pipeline (Week 3-4)**

- [ ] Implement `EMCInstallBuilder`
- [ ] Add EMC payload assembly
- [ ] Test EMC ISO generation
- [ ] Verify artifact sizes and caching

**Phase 3: Installation (Week 5-6)**

- [ ] Implement EMC installation logic
- [ ] Add Gitea setup
- [ ] Add ArgoCD setup
- [ ] Add observability stack setup

**Phase 4: Integration (Week 7-8)**

- [ ] Test EMC + control node integration
- [ ] Test GitOps workflows
- [ ] Test observability dashboards
- [ ] Document end-to-end procedures

**Phase 5: Polish (Week 9-10)**

- [ ] Add comprehensive error handling
- [ ] Implement rollback logic
- [ ] Write user documentation
- [ ] Create tutorial videos/demos

### 14. Potential Challenges

**Challenge 1: Image Size**

- EMC ISO will be significantly larger (15-20 GB with all components)
- **Mitigation:** Implement multi-stage download or pull-on-boot for some components

**Challenge 2: Installation Time**

- Full stack installation could take 20-30 minutes
- **Mitigation:** Add progress indicators, optimize service startup order

**Challenge 3: Resource Requirements**

- EMC needs substantial resources (CPU, memory, disk)
- **Mitigation:** Document minimum requirements, add pre-flight checks

**Challenge 4: Integration Complexity**

- Coordinating control nodes + EMC + gateways is complex
- **Mitigation:** Phased rollout, comprehensive testing, good error messages

**Challenge 5: Upstream Divergence**

- Long-lived fork may diverge from upstream
- **Mitigation:** Regular rebasing, upstream PRs for generic improvements

---

## Appendix: File Reference Index

### Core Build Pipeline

| File | Description | Key Lines |
|------|-------------|-----------|
| `cmd/hhfab/main.go` | CLI entry point | 731-762 (build command) |
| `pkg/hhfab/cmdbuild.go` | Build orchestration | 29-137 (Build function) |
| `pkg/fab/recipe/build.go` | Core build engine | 65-173 (buildInstall), 194-397 (buildUSBImage) |
| `pkg/fab/recipe/control_build.go` | Control builder | 70-87 (Build), 89-231 (addPayload), 236-275 (buildIgnition) |
| `pkg/fab/recipe/node_build.go` | Gateway/node builder | Similar pattern to control |

### Configuration and Types

| File | Description |
|------|-------------|
| `api/fabricator/v1beta1/fabricator_types.go` | Fabricator CRD definition |
| `api/fabricator/v1beta1/control_types.go` | Control node types |
| `pkg/fab/versions.go` | Version source of truth |
| `pkg/hhfab/cmdconfig.go` | Config loading and validation |

### Components

| File | Description |
|------|-------------|
| `pkg/fab/comp/comp.go` | Component interface |
| `pkg/fab/comp/k3s/k3s.go` | K3s component |
| `pkg/fab/comp/zot/zot.go` | Zot registry component |
| `pkg/fab/comp/certmanager/certmanager.go` | Cert-manager component |
| `pkg/fab/comp/fabric/fabric.go` | Fabric controller component |
| `pkg/fab/comp/gateway/gateway.go` | Gateway controller component |

### Artifact Management

| File | Description |
|------|-------------|
| `pkg/artificer/downloader.go` | OCI artifact downloader |
| `pkg/artificer/oci.go` | OCI registry operations |

### Installation

| File | Description |
|------|-------------|
| `cmd/hhfab-recipe/main.go` | Installation binary entry point |
| `pkg/fab/recipe/install_upgrade.go` | Install/upgrade orchestration |
| `pkg/fab/recipe/control_install.go` | Control node installation |
| `pkg/fab/recipe/node_install.go` | Gateway/node installation |

### Templates

| File | Description |
|------|-------------|
| `pkg/fab/recipe/control_butane.tmpl.yaml` | Control ignition template |
| `pkg/fab/recipe/node_butane.tmpl.yaml` | Node ignition template |

---

## Summary

This research provides a complete understanding of the current hhfab controller build pipeline:

### What We Know

✅ Build process creates bootable ISO/USB images, not OVAs
✅ Uses Flatcar Linux with Ignition for provisioning
✅ Embeds all dependencies in installation media
✅ Automated installation via systemd service on first boot
✅ Supports airgap deployments with full offline capability
✅ Hash-based build caching for fast incremental builds
✅ OCI registry for artifact distribution
✅ Version management via single source of truth

### What to Clarify

❓ OVA requirement: Do we need actual VMware OVAs or are ISOs sufficient?
❓ Multi-control HA: How is cluster formation coordinated?
❓ Upgrade constraints: What can be upgraded in place vs. requires reinstall?
❓ EMC integration: Standalone cluster or integrated with control nodes?

### Next Steps

1. **Decision:** Confirm ISO-based approach is acceptable for EMC or if OVA generation needed
2. **Design:** Create EMC architecture decision record (ADR)
3. **Prototype:** Build minimal EMC installer with Gitea + ArgoCD
4. **Test:** Validate EMC + control node integration
5. **Document:** Create comprehensive EMC build and user guides

---

**Research Completed:** 2025-11-07
**Researcher:** AI Dev Agent
**Review Status:** Ready for technical review
**Related Issues:** #1 (Epic), #2 (This research)

---

## Questions for Review

1. Is ISO-based delivery acceptable or do we need to generate actual OVA files?
2. Should EMC run on a standalone K3s cluster or integrate with control nodes?
3. What is the minimum viable EMC for v1? (Gitea + ArgoCD only, or full observability stack?)
4. Do we need to support upgrading from fabricator without EMC to fabricator with EMC?
5. Should we upstream EMC support or keep it fork-only initially?
