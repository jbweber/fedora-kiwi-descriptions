# Custom Fedora Cloud Images

This is a custom fork of fedora-kiwi-descriptions to build modified Fedora cloud images.

## What We're Doing

We're creating custom Fedora cloud images based on the official Fedora KIWI descriptions, but with modifications for our specific use cases.

## Current Customizations

### Cloud-Base-Generic-EXT4 Profile

**What it is:** A standard Fedora cloud image using EXT4 filesystem instead of btrfs.

**Why:**
- EXT4 is more universally compatible
- Simpler filesystem without subvolumes
- Better compatibility with various cloud environments and tools

**Changes from official Cloud-Base-Generic:**
- Filesystem: `ext4` (instead of `btrfs`)
- No btrfs subvolumes or compression
- Includes `e2fsprogs` package for EXT4 tools
- Optimized for hybrid cloud/bare metal deployment
- Package installation uses `onlyRequired` pattern (no weak dependencies)
- Excludes unnecessary packages: `podman*`, `criu*`, `qemu-user-static*`, `sssd*`, `zram-generator*`
- Includes hardware support: `linux-firmware`, `microcode_ctl`, `logrotate`
- CloudCore modified to allow firmware packages (removed `*-firmware` exclusion)

**What you get:**
- Fedora 43 cloud image (~1.56 GB installed, ~704 MB qcow2)
- 5GB virtual disk (qcow2 format, sparse/optimized)
- UEFI boot with GRUB2
- cloud-init support
- qemu-guest-agent for VM integration
- Hardware firmware support for bare metal (linux-firmware included)
- CPU microcode updates (Intel/AMD)
- Log rotation support (logrotate)
- Optimized package selection excluding container/virtualization tools

## Building Images

### Requirements

- Fedora 43 (or latest stable Fedora)
- Root/sudo access
- Internet connection

### Install Build Tools

```bash
sudo dnf install -y kiwi kiwi-systemdeps distribution-gpg-keys
```

### Build an Image

```bash
# Clone this repo
git clone https://github.com/jbweber/fedora-kiwi-descriptions.git
cd fedora-kiwi-descriptions

# Switch to custom branch
git checkout f43-custom

# Build the EXT4 cloud image
sudo ./kiwi-build \
  --kiwi-file=Fedora.kiwi \
  --image-type=oem \
  --image-profile=Cloud-Base-Generic-EXT4 \
  --output-dir=./output
```

### Build Output

After successful build, you'll find:
- `./output/` - Final image files
- `./output-build/` - Build artifacts (can be deleted)

The final qcow2 image will be ready to use in your cloud environment.

## Image Details

### Disk Layout (Cloud-Base-Generic-EXT4)

```
/dev/sda1 - EFI System Partition (100MB, FAT32)
/dev/sda2 - /boot (2GB, ext4)
/dev/sda3 - / root (remaining space, ext4)
```

### Included Features

- UEFI Secure Boot compatible
- Serial console enabled
- cloud-init for initial configuration
- qemu-guest-agent for hypervisor integration
- Optimized qcow2 format (sparse, compressed)

## Branch Strategy

- `f43-custom` - Fedora 43 with our customizations
- Track upstream `f43` branch from fedora-kiwi-descriptions
- Rebase periodically to get upstream updates

## Development Notes

### Build Environment

KIWI image builds require:
- A proper Linux environment (not a container) with full `/proc` access
- Root/sudo privileges for mounting filesystems
- Fedora Linux host system (tested on Fedora 43)

**Note:** Builds will fail in containerized environments due to kernel filesystem mount restrictions. Use a VM or bare metal system.

### Build Time

Typical build time for Cloud-Base-Generic-EXT4:
- Package installation: ~1-2 minutes
- Image creation and formatting: ~1 minute
- Total: ~2-3 minutes on modern hardware

### Testing Your Image

You can test the built image locally with QEMU/KVM:

```bash
# Create a cloud-init config (optional)
cat > user-data.yaml <<EOF
#cloud-config
users:
  - name: fedora
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-rsa YOUR_SSH_KEY_HERE
EOF

# Create cloud-init ISO
cloud-localds seed.img user-data.yaml

# Boot with QEMU
qemu-system-x86_64 \
  -m 2048 \
  -smp 2 \
  -drive file=Fedora-43-Cloud-EXT4.qcow2,format=qcow2 \
  -drive file=seed.img,format=raw \
  -net nic -net user,hostfwd=tcp::2222-:22 \
  -nographic
```

### Cloud-Base-K8s-EXT4 Profile

**What it is:** Kubernetes-ready cloud image with EXT4, pre-configured for immediate cluster deployment.

**Includes:**
- All Cloud-Base-Generic-EXT4 features (EXT4, hardware support, optimized packages)
- **Kubernetes 1.34**: kubelet, kubeadm, kubectl
- **containerd 1.34**: Container runtime
- **Pre-configured** kernel modules, sysctl, CNI paths
- **Enabled services**: containerd ready at boot

**Kubernetes Configuration (applied at build):**
- Kernel modules: `overlay`, `br_netfilter` (`/etc/modules-load.d/k8s.conf`)
- Sysctl: bridge netfilter, IP forwarding (`/etc/sysctl.d/k8s.conf`)
- containerd config: Custom config with CNI plugin paths (`/etc/containerd/config.toml`)
  - CNI bin_dirs: `/usr/libexec/cni` (system), `/var/lib/cni/bin` (writable)
  - CNI conf_dir: `/etc/cni/net.d`
  - Symlink: `/opt/cni/bin` → `/var/lib/cni/bin` (compatibility)
- Kubelet volume plugins: `/var/lib/kubelet/volumeplugins`

**Ready to deploy:** Just run `kubeadm init` after first boot!

## Building Images

### Build Cloud-Base-Generic-EXT4
```bash
cd fedora-kiwi-descriptions
sudo ./kiwi-build \
  --kiwi-file=Fedora.kiwi \
  --image-type=oem \
  --image-profile=Cloud-Base-Generic-EXT4 \
  --output-dir=./output
```

### Build Cloud-Base-K8s-EXT4
```bash
cd fedora-kiwi-descriptions
sudo ./kiwi-build \
  --kiwi-file=Fedora.kiwi \
  --image-type=oem \
  --image-profile=Cloud-Base-K8s-EXT4 \
  --output-dir=./output
```

**Output:** `./output-build/Fedora.x86_64-43.qcow2` (~900 MB)

**Build time:** ~2 minutes on modern hardware

## Implementation Notes

The Kubernetes configuration is based on:
- **kubernetes-sigs/image-builder** - Official CAPI image builder patterns
- **Fedora best practices** - Standard CNI paths (`/etc/cni/net.d`)
- **bootc compatibility** - Writable `/var/lib/cni/bin` for additional plugins

### Container Runtime: containerd

We use **containerd** (instead of CRI-O) as the container runtime:

**Configuration approach:**
- Custom `/etc/containerd/config.toml` provided via KIWI root overlay (`root/etc/containerd/config.toml`)
- Uses containerd v2 plugin paths: `plugins.'io.containerd.cri.v1.runtime'.cni`
- Minimal config overrides only what's needed (containerd uses internal defaults for everything else)

**CNI plugin paths:**
- `/usr/libexec/cni` - System-provided immutable plugins (from `containernetworking-plugins` RPM)
- `/var/lib/cni/bin` - Writable location for additional CNI plugins (e.g., Flannel, Calico, Cilium)
- `/opt/cni/bin` - Compatibility symlink to `/var/lib/cni/bin`

The dual-path approach allows:
- System updates to manage base CNI plugins
- CNI installers (like Flannel DaemonSet) to add plugins to writable location
- Legacy tools expecting `/opt/cni/bin` to work via symlink

Configuration applied via KIWI `config.sh` during image build, triggered by profile name matching `*K8s*`.

## Future Plans

- Add additional package sets for specific workloads
- Consider additional filesystem options as needed

## Original Project

This is based on the official Fedora KIWI descriptions:
- https://pagure.io/fedora-kiwi-descriptions
- Licensed under GPL v3

## License

GPL v3 (same as upstream project)
