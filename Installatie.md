# splunk-10.6-sok-3.2.0-k3s-almalinux-10.2

# Splunk Enterprise 10.6 on K3s with Splunk Operator for Kubernetes 3.2.0

**Single-host lab build on AlmaLinux 10.2**

| Item | Value |
| --- | --- |
| Host | AMD Ryzen 9 7940HS (8C/16T), 64 GB DDR4, 4 TB NVMe |
| OS | AlmaLinux OS 10.2 “Lavender Lion” (x86_64) |
| Kubernetes | K3s `v1.36.5+k3s1` (Kubernetes 1.36) |
| Operator | Splunk Operator for Kubernetes (SOK) `3.2.0` |
| Splunk Enterprise | `splunk/splunk:10.6.0` |
| Universal Forwarder | `splunk/universalforwarder:10.6.0` via the SOK 3.2.0 UF Helm chart |

> **Compatibility check.** SOK 3.2.0 (released 2026-09-08) lists Splunk Enterprise **10.6.0** as its supported Splunk version and Kubernetes **1.32 – 1.36** as supported Kubernetes versions. K3s v1.36.x is inside that range. Always re-check the [SOK change log](https://splunk.github.io/splunk-operator/reference/ChangeLog.html) before deploying.
> 

---

## Table of contents

1. [Target architecture and resource plan](about:blank#0-target-architecture-and-resource-plan)
2. [Install AlmaLinux 10.2](about:blank#1-install-almalinux-102)
3. [Prepare the OS for Kubernetes and Splunk](about:blank#2-prepare-the-os-for-kubernetes-and-splunk)
4. [Install Kubernetes (K3s)](about:blank#3-install-kubernetes-k3s)
5. [Install Splunk Operator for Kubernetes 3.2.0](about:blank#4-install-splunk-operator-for-kubernetes-320)
6. [Deploy the Splunk 10.6 environment](about:blank#5-deploy-the-splunk-106-environment)
7. [Deploy the Universal Forwarder container](about:blank#6-deploy-the-universal-forwarder-container)
8. [Access and verification](about:blank#7-access-and-verification)
9. [Operations: scale, upgrade, tear down](about:blank#8-operations-scale-upgrade-tear-down)
10. [Troubleshooting](about:blank#9-troubleshooting)
11. [References](about:blank#10-references)

---

## 0. Target architecture and resource plan

Everything runs on one physical host as a single-node K3s cluster. All Splunk components live in namespace `splunk`; the operator runs in namespace `splunk-operator`.

```
                       ┌─────────────────────── AlmaLinux 10.2 host ────────────────────────┐
                       │  K3s (single node: control plane + worker)                          │
                       │                                                                     │
  Browser ──NodePort──▶│  [sh1 Standalone]   [sh2 Standalone]   [Monitoring Console]         │
   :30801 / :30802     │         │  search          │  search                                │
   :30800 (CM)         │         ▼                  ▼                                        │
   :30803 (MC)         │  ┌──────── Indexer cluster "idxc" (3 peers) ────────┐               │
                       │  │ idxc-indexer-0   idxc-indexer-1   idxc-indexer-2 │◀── S2S 9997 ──┤
                       │  └──────────────────────┬───────────────────────────┘               │
                       │                         │ managed by                                │
                       │                  [Cluster Manager "cm"]    [License Manager "lm"]   │
                       │                                                                     │
  syslog UDP 514 ─────▶│  [Universal Forwarder Deployment "uf"] ──── tcpout (useACK) ────────┘
                       │  Storage: local-path provisioner on /var/lib/rancher (XFS, ~3.6 TB) │
                       └─────────────────────────────────────────────────────────────────────┘
```

| Component | CR kind / chart | Pods | CPU req / limit | Memory req / limit | Persistent storage (etc / var) |
| --- | --- | --- | --- | --- | --- |
| License Manager `lm` | `LicenseManager` | 1 | 0.5 / 1 | 1 Gi / 2 Gi | 10 Gi / 20 Gi |
| Cluster Manager `cm` | `ClusterManager` | 1 | 1 / 2 | 2 Gi / 4 Gi | 10 Gi / 50 Gi |
| Indexer cluster `idxc` | `IndexerCluster` | 3 | 2 / 4 each | 6 Gi / 10 Gi each | 10 Gi / 900 Gi each |
| Search head `sh1` | `Standalone` + `clusterManagerRef` | 1 | 2 / 4 | 4 Gi / 8 Gi | 10 Gi / 100 Gi |
| Search head `sh2` | `Standalone` + `clusterManagerRef` | 1 | 2 / 4 | 4 Gi / 8 Gi | 10 Gi / 100 Gi |
| Monitoring Console `mc` (optional) | `MonitoringConsole` | 1 | 0.5 / 2 | 2 Gi / 4 Gi | 10 Gi / 20 Gi |
| Universal Forwarder `uf` | Helm `splunk-universalforwarder` | 1 | 0.1 / 0.5 | 256 Mi / 1 Gi | emptyDir (stateless) |
| **Total** |  | **9** | **≈ 12.1 / 27.5** | **≈ 31 Gi / 57 Gi** | **≈ 3 TB** |

Notes on sizing:

- 16 hardware threads and ~60 GiB allocatable memory (after OS and kubelet reservations) are enough for the requests above. Limits are deliberately over-committed on CPU; memory limits stay under physical RAM.
- The two search heads are **independent** (not a Search Head Cluster). A SHC needs at least 3 members plus a deployer; you asked for 2 separate search heads, so two `Standalone` CRs that both point at the cluster manager is the correct pattern.
- The K3s `local-path` provisioner **does not enforce** PVC capacity. The numbers are a budget; actual limits come from Splunk index settings (`maxTotalDataSizeMB`) and free disk.
- Indexer replication factor 3 / search factor 2 means every bucket is stored three times on the same physical disk. That is fine for a lab and functionally realistic, but it is not real redundancy.
- A **License Manager** is mandatory: every instance is a licence peer of `lm`. You need a licence file (a free Splunk Developer License is fine) before §5 — see [§5.2](about:blank#52-licence-file).

---

## 1. Install AlmaLinux 10.2

### 1.1 Firmware (BIOS/UEFI) settings on the 7940HS

1. Update the mini-PC/laptop firmware to the latest version.
2. **UMA frame buffer / iGPU memory** (Radeon 780M): set to the minimum (e.g. 512 MB). Every GB reserved for the iGPU is taken from the 64 GB system RAM.
3. **SVM (AMD-V)**: enable (harmless, useful if you later run VMs).
4. **Boot mode**: UEFI only. Secure Boot may stay enabled — AlmaLinux ships signed shim/kernel.
5. Power: set “Restore on AC power loss” to *On* if this box should behave like a server.

### 1.2 Download and verify the ISO

The Ryzen 7940HS (Zen 4) supports x86-64-v3/v4, so use the standard `x86_64` image (not `x86_64_v2`).

```bash
# On any Linux workstation
ARCH=x86_64
wget https://repo.almalinux.org/almalinux/10.2/isos/${ARCH}/AlmaLinux-10.2-${ARCH}-dvd.iso
wget https://repo.almalinux.org/almalinux/10.2/isos/${ARCH}/CHECKSUM
wget https://repo.almalinux.org/almalinux/RPM-GPG-KEY-AlmaLinux-10

gpg --import RPM-GPG-KEY-AlmaLinux-10
gpg --verify CHECKSUM                     # expect: Good signature from "AlmaLinux OS 10"
sha256sum AlmaLinux-10.2-${ARCH}-dvd.iso
grep -E "SHA256.*AlmaLinux-10.2-${ARCH}-dvd.iso" CHECKSUM   # hashes must match
```

Write the ISO to a USB stick (≥ 16 GB). **This erases the stick** — double-check the device name:

```bash
lsblk                                     # identify the USB stick, e.g. /dev/sdX
sudo dd if=AlmaLinux-10.2-x86_64-dvd.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

(On Windows use Rufus in “DD image” mode or Fedora Media Writer.)

### 1.3 Anaconda installer choices

Boot from the USB stick and choose **Install AlmaLinux 10.2**.

| Screen | Setting |
| --- | --- |
| Language / Keyboard | As required (e.g. English (US), keyboard `us` or `nl`) |
| Time & Date | Region **Europe / Amsterdam**, Network Time **ON** |
| Software Selection | **Minimal Install** (add *Standard* and *Headless Management* if you want Cockpit) |
| Installation Destination | NVMe 4 TB → **Custom** partitioning (see 1.4) |
| KDUMP | Disabled (saves ~512 MB RAM) |
| Network & Host Name | Hostname `splunk-k3s.lab.local`; enable the wired NIC, set a **static IPv4** (example used below: `192.168.1.50/24`, gateway `192.168.1.1`, DNS `192.168.1.1`) |
| Security Profile | None (a CIS profile can break K3s networking) |
| Root password | Set a strong one; leave “Allow root SSH login with password” **off** |
| User creation | Create user `splunkadm`, tick **Make this user administrator** (wheel) |

### 1.4 Partition layout (GPT, UEFI)

Choose **Custom**, scheme **LVM**, then create:

| Mount point | Size | Filesystem | Notes |
| --- | --- | --- | --- |
| `/boot/efi` | 1 GiB | EFI System Partition |  |
| `/boot` | 2 GiB | xfs |  |
| `/` | 150 GiB | xfs (LVM `almalinux/root`) | OS, `/var/log`, packages |
| `/var/lib/rancher` | **all remaining space (~3.5 TiB)** | xfs (LVM `almalinux/rancher`) | K3s data, container images, **all Splunk PVCs** |
| swap | **none** | — | Kubernetes expects swap off |

If Anaconda insists on swap, create a small one and disable it in §2.3.

Click **Begin Installation**, wait, then **Reboot System** and remove the USB stick.

### 1.5 First login

```bash
ssh splunkadm@192.168.1.50
cat /etc/almalinux-release          # AlmaLinux release 10.2 (Lavender Lion)
uname -r                            # 6.12.0-*.el10_2.x86_64
lsblk -f                            # confirm /var/lib/rancher is the large XFS volume
df -h /var/lib/rancher
```

---

## 2. Prepare the OS for Kubernetes and Splunk

All commands in this section run as root (`sudo -i`).

### 2.1 Update and base packages

```bash
dnf -y upgrade --refresh
dnf -y install curl wget tar git jq vim bash-completion \
  container-selinux policycoreutils-python-utils \
  iptables-nft nftables kernel-modules-extra \
  chrony tuned
systemctl enable --now chronyd
reboot
```

`kernel-modules-extra` provides netfilter modules that container networking may need on EL10 kernels.

### 2.2 Hostname resolution

```bash
hostnamectl set-hostname splunk-k3s.lab.local
echo "192.168.1.50  splunk-k3s.lab.local splunk-k3s" >> /etc/hosts
```

### 2.3 Disable swap (including zram, if present)

```bash
swapoff -a
sed -ri '/\sswap\s/s/^/#/' /etc/fstab
dnf -y remove zram-generator-defaults 2>/dev/null || true
free -h        # Swap: 0B
```

### 2.4 Kernel modules and sysctls

```bash
cat > /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

cat > /etc/sysctl.d/90-k8s-splunk.conf <<'EOF'
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
# Many pods + Splunk file monitoring
fs.inotify.max_user_instances       = 8192
fs.inotify.max_user_watches         = 1048576
fs.file-max                         = 6553600
vm.swappiness                       = 0
EOF
sysctl --system
```

### 2.5 Disable Transparent Huge Pages (Splunk recommendation)

```bash
grubby --update-kernel=ALL --args="transparent_hugepage=never"
tuned-adm profile throughput-performance
reboot
# after reboot:
cat /sys/kernel/mm/transparent_hugepage/enabled    # ...[never]
```

THP is a kernel-wide setting, so setting it on the host also applies to all Splunk containers.

### 2.6 Firewall (firewalld stays enabled)

```bash
# Kubernetes API (only needed from your admin workstation)
firewall-cmd --permanent --add-port=6443/tcp
# Trust the K3s pod and service CIDRs
firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16
# Splunk Web NodePorts (cm, sh1, sh2, mc, lm) – defined in §5.8
firewall-cmd --permanent --add-port=30800-30804/tcp
# Optional: syslog into the UF from the LAN (NodePort defined in §6.4)
firewall-cmd --permanent --add-port=30514/udp
firewall-cmd --reload
firewall-cmd --list-all
```

### 2.7 SELinux

Leave SELinux in **enforcing** mode; K3s is configured with `selinux: true` in §3.1.

```bash
getenforce          # Enforcing
```

---

## 3. Install Kubernetes (K3s)

### 3.1 K3s configuration file

Create the config **before** running the installer so the first start already uses it.

```bash
mkdir -p /etc/rancher/k3s
cat > /etc/rancher/k3s/config.yaml <<'EOF'
write-kubeconfig-mode: "0600"
node-name: splunk-k3s
selinux: true
# Traefik is not needed (we use NodePorts); keep servicelb, local-storage, metrics-server
disable:
  - traefik
# Keep resources back for the OS and K3s itself
kubelet-arg:
  - "system-reserved=cpu=1,memory=2Gi"
  - "kube-reserved=cpu=1,memory=2Gi"
  - "eviction-hard=memory.available<1Gi,nodefs.available<5%,imagefs.available<5%"
  - "max-pods=110"
# Default data dir is /var/lib/rancher/k3s (on the large XFS volume)
EOF
```

### 3.2 Install K3s (pinned to a version supported by SOK 3.2.0)

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.36.5+k3s1" sh -
```

The installer detects SELinux and tries to install the `k3s-selinux` policy RPM from `rpm.rancher.io` for your EL major version.

> **If the installer fails on the SELinux RPM** (no EL10 build available for your date), you have two lab-acceptable options:
> 
> 1. Re-run with `INSTALL_K3S_SKIP_SELINUX_RPM=true`, then set `selinux: false` in `config.yaml` and run SELinux in permissive mode (`setenforce 0` and `SELINUX=permissive` in `/etc/selinux/config`), **or**
> 2. Check `https://rpm.rancher.io/k3s/stable/common/` for an `el10` (or `centos/10`) repo, add it manually, `dnf install k3s-selinux`, and re-run the installer.

### 3.3 Verify and configure `kubectl` for the admin user

```bash
systemctl status k3s --no-pager
k3s kubectl get nodes -o wide         # STATUS Ready, VERSION v1.36.5+k3s1
k3s kubectl get pods -A               # coredns, local-path-provisioner, metrics-server Running
k3s kubectl get storageclass          # local-path (default)

# kubeconfig for splunkadm
exit                                  # back to splunkadm
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl; complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

### 3.4 Install Helm (needed for the UF chart)

```bash
dnf -y install openssl tar
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 3.5 (Optional) Pre-pull the large Splunk images

Pre-pulling avoids long `ContainerCreating` phases and Docker Hub rate-limit surprises:

```bash
sudo k3s ctr images pull docker.io/splunk/splunk:10.6.0
sudo k3s ctr images pull docker.io/splunk/universalforwarder:10.6.0
sudo k3s ctr images pull docker.io/splunk/splunk-operator:3.2.0
```

> **Image tags.** Starting with 10.6, Splunk uses four-segment versions (e.g. `10.6.0.x`). Confirm the exact tag on Docker Hub (`splunk/splunk`, `splunk/universalforwarder`) and use it consistently in every manifest below. This guide uses `10.6.0`, the version listed as supported by SOK 3.2.0.
> 

---

## 4. Install Splunk Operator for Kubernetes 3.2.0

The cluster-wide manifest installs the CRDs, RBAC, the validation webhook and the operator Deployment into namespace `splunk-operator`, watching all namespaces.

### 4.1 Download and apply the manifest

```bash
mkdir -p ~/splunk-k8s && cd ~/splunk-k8s
wget -O splunk-operator-cluster.yaml \
  https://github.com/splunk/splunk-operator/releases/download/3.2.0/splunk-operator-cluster.yaml
kubectl apply -f splunk-operator-cluster.yaml --server-side
```

### 4.2 Accept the Splunk General Terms and pin the Splunk image

Since SOK 3.0.0 / Splunk 10.x, Splunk containers will not start unless the operator passes `SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com`.

> **Legal:** Read the [Splunk General Terms](https://www.splunk.com/en_us/legal/splunk-general-terms.html) first. Setting this value means you have read and accept the current version of those terms.
> 

```bash
kubectl -n splunk-operator set env deployment/splunk-operator-controller-manager \
  SPLUNK_GENERAL_TERMS="--accept-sgt-current-at-splunk-com" \
  RELATED_IMAGE_SPLUNK_ENTERPRISE="splunk/splunk:10.6.0"
```

### 4.3 Verify the operator

```bash
kubectl -n splunk-operator rollout status deployment/splunk-operator-controller-manager
kubectl -n splunk-operator get pods
kubectl get crd | grep enterprise.splunk.com
# Expect at least: clustermanagers, indexerclusters, licensemanagers, monitoringconsoles,
#                  searchheadclusters, standalones  (all .enterprise.splunk.com)
kubectl -n splunk-operator get deploy splunk-operator-controller-manager \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'     # splunk/splunk-operator:3.2.0
kubectl -n splunk-operator logs deploy/splunk-operator-controller-manager --tail=30
```

---

## 5. Deploy the Splunk 10.6 environment

Deployment order: **License Manager → Cluster Manager → Indexer cluster → Search heads + Monitoring Console**. Every Splunk instance references the License Manager through `licenseManagerRef`, so it must exist first.

> **Tip:** create each file with `cat > file.yaml <<'EOF' … EOF` (spaces only, no tabs) and validate it first with `kubectl apply --dry-run=server -f file.yaml`.
> 

### 5.1 Namespace

```bash
kubectl create namespace splunk
```

SOK automatically generates a namespace-wide secret, `splunk-splunk-secret` (pattern `splunk-<namespace>-secret`), holding the admin password, `pass4SymmKey`, `idxc_secret` and HEC token shared by all Splunk pods in the namespace.

### 5.2 Licence file

A Splunk Enterprise licence file is **required** for this build. Options:

- **Splunk Developer License** (free, 10 GB/day, 6 months, renewable) — request it at [https://dev.splunk.com/enterprise/dev_license/](https://dev.splunk.com/enterprise/dev_license/)
- **Enterprise licence** from your Splunk account team.

Copy the licence file (an XML file, usually `*.lic` or `Splunk.License`) to the host, e.g. `~/splunk-k8s/enterprise.lic`, and check it:

```bash
cd ~/splunk-k8s
head -c 300 enterprise.lic; echo        # must start with <license>
grep -oE '<(expiration_time|quota|label)>[^<]+' enterprise.lic
```

Store it in a ConfigMap. SOK mounts every entry under `spec.volumes` at `/mnt/<volume-name>/` inside the pod:

```bash
kubectl -n splunk create configmap splunk-licenses --from-file=enterprise.lic=./enterprise.lic
kubectl -n splunk get configmap splunk-licenses
```

> Multiple licence files (stacking): add each with an extra `--from-file=<name>.lic=./<file>` and list them comma-separated in `licenseUrl`, e.g. `/mnt/licenses/enterprise.lic,/mnt/licenses/extra.lic`.
> 

### 5.3 License Manager — `00-license-manager.yaml`

```yaml
apiVersion: enterprise.splunk.com/v4
kind: LicenseManager
metadata:
  name: lm
  namespace: splunk
  finalizers:
    - enterprise.splunk.com/delete-pvc
spec:
  image: splunk/splunk:10.6.0
  imagePullPolicy: IfNotPresent
  volumes:
    - name: licenses
      configMap:
        name: splunk-licenses
  licenseUrl: /mnt/licenses/enterprise.lic
  monitoringConsoleRef:
    name: mc
  resources:
    requests:
      cpu: "500m"
      memory: 1Gi
    limits:
      cpu: "1"
      memory: 2Gi
  etcVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 10Gi
  varVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 20Gi
```

### 5.4 Cluster Manager — `01-cluster-manager.yaml`

```yaml
apiVersion: enterprise.splunk.com/v4
kind: ClusterManager
metadata:
  name: cm
  namespace: splunk
  finalizers:
    - enterprise.splunk.com/delete-pvc
spec:
  image: splunk/splunk:10.6.0
  imagePullPolicy: IfNotPresent
  licenseManagerRef:
    name: lm
  monitoringConsoleRef:
    name: mc
  # Single-line form: immune to copy/paste indentation damage
  defaults: "splunk:\n  idxc:\n    replication_factor: 3\n    search_factor: 2\n"
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "2"
      memory: 4Gi
  etcVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 10Gi
  varVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 50Gi
```

### 5.5 Indexer cluster (3 peers) — `02-indexer-cluster.yaml`

```yaml
apiVersion: enterprise.splunk.com/v4
kind: IndexerCluster
metadata:
  name: idxc
  namespace: splunk
  finalizers:
    - enterprise.splunk.com/delete-pvc
spec:
  replicas: 3
  image: splunk/splunk:10.6.0
  imagePullPolicy: IfNotPresent
  clusterManagerRef:
    name: cm
  licenseManagerRef:
    name: lm
  monitoringConsoleRef:
    name: mc
  resources:
    requests:
      cpu: "2"
      memory: 6Gi
    limits:
      cpu: "4"
      memory: 10Gi
  etcVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 10Gi
  varVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 900Gi
```

### 5.6 Two separate search heads — `03-search-heads.yaml`

Each search head is a `Standalone` instance with `clusterManagerRef`, which makes it a search head of the indexer cluster.

```yaml
apiVersion: enterprise.splunk.com/v4
kind: Standalone
metadata:
  name: sh1
  namespace: splunk
  finalizers:
    - enterprise.splunk.com/delete-pvc
spec:
  replicas: 1
  image: splunk/splunk:10.6.0
  imagePullPolicy: IfNotPresent
  clusterManagerRef:
    name: cm
  licenseManagerRef:
    name: lm
  monitoringConsoleRef:
    name: mc
  resources:
    requests:
      cpu: "2"
      memory: 4Gi
    limits:
      cpu: "4"
      memory: 8Gi
  etcVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 10Gi
  varVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 100Gi
---
apiVersion: enterprise.splunk.com/v4
kind: Standalone
metadata:
  name: sh2
  namespace: splunk
  finalizers:
    - enterprise.splunk.com/delete-pvc
spec:
  replicas: 1
  image: splunk/splunk:10.6.0
  imagePullPolicy: IfNotPresent
  clusterManagerRef:
    name: cm
  licenseManagerRef:
    name: lm
  monitoringConsoleRef:
    name: mc
  resources:
    requests:
      cpu: "2"
      memory: 4Gi
    limits:
      cpu: "4"
      memory: 8Gi
  etcVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 10Gi
  varVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 100Gi
```

### 5.7 Monitoring Console (optional but recommended) — `04-monitoring-console.yaml`

```yaml
apiVersion: enterprise.splunk.com/v4
kind: MonitoringConsole
metadata:
  name: mc
  namespace: splunk
  finalizers:
    - enterprise.splunk.com/delete-pvc
spec:
  image: splunk/splunk:10.6.0
  imagePullPolicy: IfNotPresent
  licenseManagerRef:
    name: lm
  resources:
    requests:
      cpu: "500m"
      memory: 2Gi
    limits:
      cpu: "2"
      memory: 4Gi
  etcVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 10Gi
  varVolumeStorageConfig:
    storageClassName: local-path
    storageCapacity: 20Gi
```

If you skip the Monitoring Console, remove every `monitoringConsoleRef` block from the files above (including the License Manager).

### 5.8 Web access via NodePort — `05-nodeport-services.yaml`

These Services select the operator-created pods by the standard StatefulSet pod-name label, so they do not depend on operator-internal labels.

```yaml
apiVersion: v1
kind: Service
metadata: { name: web-cm, namespace: splunk }
spec:
  type: NodePort
  selector: { statefulset.kubernetes.io/pod-name: splunk-cm-cluster-manager-0 }
  ports: [ { name: web, port: 8000, targetPort: 8000, nodePort: 30800 } ]
---
apiVersion: v1
kind: Service
metadata: { name: web-sh1, namespace: splunk }
spec:
  type: NodePort
  selector: { statefulset.kubernetes.io/pod-name: splunk-sh1-standalone-0 }
  ports: [ { name: web, port: 8000, targetPort: 8000, nodePort: 30801 } ]
---
apiVersion: v1
kind: Service
metadata: { name: web-sh2, namespace: splunk }
spec:
  type: NodePort
  selector: { statefulset.kubernetes.io/pod-name: splunk-sh2-standalone-0 }
  ports: [ { name: web, port: 8000, targetPort: 8000, nodePort: 30802 } ]
---
apiVersion: v1
kind: Service
metadata: { name: web-mc, namespace: splunk }
spec:
  type: NodePort
  selector: { statefulset.kubernetes.io/pod-name: splunk-mc-monitoring-console-0 }
  ports: [ { name: web, port: 8000, targetPort: 8000, nodePort: 30803 } ]
---
apiVersion: v1
kind: Service
metadata: { name: web-lm, namespace: splunk }
spec:
  type: NodePort
  selector: { statefulset.kubernetes.io/pod-name: splunk-lm-license-manager-0 }
  ports: [ { name: web, port: 8000, targetPort: 8000, nodePort: 30804 } ]
```

### 5.9 Apply in order and watch

```bash
cd ~/splunk-k8s
kubectl apply -f 00-license-manager.yaml
kubectl -n splunk get licensemanager lm -w          # wait for PHASE Ready, Ctrl-C

kubectl apply -f 01-cluster-manager.yaml
kubectl -n splunk get clustermanager cm -w          # wait for PHASE Ready (5–10 min first time), Ctrl-C

kubectl apply -f 02-indexer-cluster.yaml
kubectl -n splunk get indexercluster idxc -w        # wait for Ready, 3/3

kubectl apply -f 03-search-heads.yaml -f 04-monitoring-console.yaml -f 05-nodeport-services.yaml
kubectl -n splunk get standalone,monitoringconsole -w
```

Full status:

```bash
kubectl -n splunk get licensemanager,clustermanager,indexercluster,standalone,monitoringconsole
kubectl -n splunk get pods -o wide
kubectl -n splunk get pvc
kubectl -n splunk get svc
```

Expected pods:

```
splunk-lm-license-manager-0         1/1 Running
splunk-cm-cluster-manager-0         1/1 Running
splunk-idxc-indexer-0               1/1 Running
splunk-idxc-indexer-1               1/1 Running
splunk-idxc-indexer-2               1/1 Running
splunk-sh1-standalone-0             1/1 Running
splunk-sh2-standalone-0             1/1 Running
splunk-mc-monitoring-console-0      1/1 Running
```

### 5.10 Verify licensing

```bash
PW=$(kubectl -n splunk get secret splunk-splunk-secret -o jsonpath='{.data.password}' | base64 -d)

# The above command stores the automatically generated admin password in an environment variable.
# You might want to run 'echo $PW' and store the password somewhere in a safe place
echo $PW


# Licence installed on the License Manager
kubectl -n splunk exec splunk-lm-license-manager-0 -- \
  /opt/splunk/bin/splunk list licenses -auth "admin:${PW}"

# Every other instance must be a licence peer of the LM
for p in splunk-cm-cluster-manager-0 splunk-idxc-indexer-0 splunk-sh1-standalone-0 splunk-sh2-standalone-0; do
  echo "==$p"
  kubectl -n splunk exec $p -- /opt/splunk/bin/splunk list licenser-localpeer -auth "admin:${PW}" \
    | grep -E "manager_uri|master_uri|guid" | head -2
done
```

In the License Manager UI (`http://10.10.99.10:30804`) → **Settings → Licensing**: the licence stack shows your licence, and the peer list shows the CM, three indexers, both search heads and the MC.

**Renewing / replacing the licence** (e.g. a Developer License after 6 months):

```bash
kubectl -n splunk create configmap splunk-licenses --from-file=enterprise.lic=./new.lic \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n splunk delete pod splunk-lm-license-manager-0      # LM restarts and loads the new file
```

---

## 6. Deploy the Universal Forwarder container

SOK 3.2.0 introduced a dedicated **Universal Forwarder Helm chart** (stateless by default, Deployment-only). The UF is *not* a SOK custom resource; the operator’s CRDs do not support the forwarder image, so the Helm chart is the supported route.

### 6.1 Get the chart from the 3.2.0 release

```bash
cd ~/splunk-k8s
git clone --branch 3.2.0 --depth 1 https://github.com/splunk/splunk-operator.git sok-src
ls sok-src/helm-chart/splunk-universalforwarder
```

### 6.2 Values file — `uf-values.yaml`

Replace `<UF_ADMIN_PASSWORD>` with a strong password (the chart refuses `changeme`).

```yaml
image:
  repository: docker.io/splunk/universalforwarder
  tag: "10.6.0"            # chart default is 10.4.0; UF 10.6 is certified with Splunk 10.6
  pullPolicy: IfNotPresent

replicaCount: 1

splunkConfig:
  password: "W5R10XvgfiepDlK4Cv544rxt"
  # Read the Splunk General Terms first; this value records your acceptance
  splunkGeneralTerms: "--accept-sgt-current-at-splunk-com"
  # Required by chart validation; the full load-balanced list is in conf.outputs below
  forwardServers:
    - splunk-idxc-indexer-0.splunk-idxc-indexer-headless.splunk.svc.cluster.local:9997
  # Inputs created at start-up (comma-joined into SPLUNK_ADD)
  add:
    - "udp 514"

conf:
  outputs: |
    [tcpout]
    defaultGroup = idxc

    [tcpout:idxc]
    server = splunk-idxc-indexer-0.splunk-idxc-indexer-headless.splunk.svc.cluster.local:9997,splunk-idxc-indexer-1.splunk-idxc-indexer-headless.splunk.svc.cluster.local:9997,splunk-idxc-indexer-2.splunk-idxc-indexer-headless.splunk.svc.cluster.local:9997
    autoLBFrequency = 30
    useACK = true

storage:
  emptyDir: true           # stateless; set false + updateStrategy Recreate for durable fishbucket

service:
  enabled: true
  type: ClusterIP

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 1Gi
    ephemeral-storage: 2Gi
```

> Pointing the UF at the three indexer pods individually (headless DNS) gives true Splunk auto-load-balancing with indexer acknowledgement. Using the `splunk-idxc-indexer-service` ClusterIP would also work, but Kubernetes rather than Splunk would decide which indexer receives each connection.
> 

### 6.3 Install

```bash
dnf -y install openssl tar
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
helm install uf ./sok-src/helm-chart/splunk-universalforwarder \
  --namespace splunk \
  --values uf-values.yaml

kubectl -n splunk get pods -l app.kubernetes.io/name=splunk-universalforwarder -w
```

### 6.4 (Optional) Receive syslog from the LAN on UDP 30514

```bash
cat > 06-uf-syslog-nodeport.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata: { name: uf-syslog-nodeport, namespace: splunk }
spec:
  type: NodePort
  selector: { app.kubernetes.io/name: splunk-universalforwarder }
  ports: [ { name: syslog-udp, port: 514, targetPort: 514, protocol: UDP, nodePort: 30514 } ]
EOF
kubectl apply -f 06-uf-syslog-nodeport.yaml
```

### 6.5 Verify forwarding

```bash
UF_POD=$(kubectl -n splunk get pod -l app.kubernetes.io/name=splunk-universalforwarder -o jsonpath='{.items[0].metadata.name}')

kubectl -n splunk exec $UF_POD -- cat /opt/splunkforwarder/etc/system/local/outputs.conf
kubectl -n splunk exec $UF_POD -- \
  grep -E "Connected to idx|AutoLoadBalanced" /opt/splunkforwarder/var/log/splunk/splunkd.log | tail -5

# Send a test syslog event to the UF from inside the cluster
UF_SVC=$(kubectl -n splunk get svc -l app.kubernetes.io/name=splunk-universalforwarder \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n splunk run sendlog --rm -it --restart=Never --image=busybox:1.36.1 -- \
  sh -c "echo '<14>hello from busybox via UF' | nc -u -w1${UF_SVC} 514"
```

---

## 7. Access and verification

### 7.1 Admin password

```bash
kubectl -n splunk get secret splunk-splunk-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

User: `admin`. The same password is valid on the LM, CM, indexers, both search heads and the MC.

### 7.2 Web UIs

| Component | URL |
| --- | --- |
| Cluster Manager | `http://10.10.99.10:30800` |
| Search head 1 | `http://10.10.99.10:30801` |
| Search head 2 | `http://10.10.99.10:30802` |
| Monitoring Console | `http://10.10.99.10:30803` |
| License Manager | `http://10.10.99.10:30804` |

Alternative without NodePorts (from a workstation with kubeconfig):

```bash
kubectl -n splunk port-forward svc/splunk-sh1-standalone-service 8000:8000
```

### 7.3 Indexer cluster health

```bash
PW=$(kubectl -n splunk get secret splunk-splunk-secret -o jsonpath='{.data.password}' | base64 -d)
kubectl -n splunk exec splunk-cm-cluster-manager-0 -- \
  /opt/splunk/bin/splunk show cluster-status --verbose -auth "admin:${PW}"
```

Expect 3 peers `Up`, *Replication factor met* and *Search factor met*.

In the Cluster Manager UI: **Settings → Indexer Clustering** → three peers, and both search heads listed on the *Search Heads* tab.

### 7.4 End-to-end search

On **sh1** and on **sh2** run:

```
index=main source="udp:514" "hello from busybox"
```

and

```
| tstats count where index=_internal by splunk_server
```

The second search should return all three indexer pods (`splunk-idxc-indexer-0/1/2`), proving both search heads are distributed across the cluster.

### 7.5 Versions sanity check

```bash
kubectl -n splunk exec splunk-sh1-standalone-0 -- /opt/splunk/bin/splunk version
kubectl -n splunk exec $UF_POD -- /opt/splunkforwarder/bin/splunk version
kubectl version
```

---

## 8. Operations: scale, upgrade, tear down

**Scale indexers** (watch RAM: each peer requests 6 Gi):

```bash
kubectl -n splunk patch indexercluster idxc --type merge -p '{"spec":{"replicas":4}}'
```

Remember to add the new peer to `conf.outputs` in `uf-values.yaml` and run `helm upgrade uf ... --values uf-values.yaml`.

**Deploy apps / indexes** to the indexer cluster: use the SOK *App Framework* (S3-compatible storage, e.g. MinIO) or place apps in `manager-apps` on the CM and push the bundle.

**Upgrade Splunk** (e.g. to a later 10.6.0.x tag): change `spec.image` in each CR (LM first, then CM, then indexers, then search heads and MC) and re-apply; SOK handles the rolling order.

**Upgrade the UF:**

```bash
helm upgrade uf ./sok-src/helm-chart/splunk-universalforwarder -n splunk \
  --values uf-values.yaml --set image.tag=<new-tag>
```

**Tear down** (the `delete-pvc` finalizer removes the data volumes — **this destroys all indexed data**):

```bash
helm uninstall uf -n splunk
kubectl -n splunk delete -f 05-nodeport-services.yaml -f 04-monitoring-console.yaml \
  -f 03-search-heads.yaml -f 02-indexer-cluster.yaml -f 01-cluster-manager.yaml \
  -f 00-license-manager.yaml
kubectl -n splunk delete configmap splunk-licenses
kubectl delete namespace splunk
kubectl delete -f splunk-operator-cluster.yaml
# Remove K3s completely:
sudo /usr/local/bin/k3s-uninstall.sh
```

---

## 9. Troubleshooting

| Symptom | Check / fix |
| --- | --- |
| Splunk pods exit immediately, logs mention licence/terms | `SPLUNK_GENERAL_TERMS` not set on the operator (§4.2): `kubectl -n splunk-operator get deploy splunk-operator-controller-manager -o yaml \| grep -A1 SPLUNK_GENERAL_TERMS` |
| `lm` stays in `Error` / licence not loaded | `kubectl -n splunk logs splunk-lm-license-manager-0 \| grep -i licen`; check the file is mounted: `kubectl -n splunk exec splunk-lm-license-manager-0 -- ls -l /mnt/licenses/`; `licenseUrl` must match the ConfigMap key; licence must not be expired |
| Peers show licence warnings / not connected to LM | `licenseManagerRef: { name: lm }` missing in a CR; re-apply it. Check §5.10 output |
| Pods stuck `Pending` | `kubectl -n splunk describe pod <pod>` → usually *Insufficient memory/cpu*; reduce requests or replicas. `kubectl describe node splunk-k3s` shows allocations. |
| Pods stuck `ContainerCreating` for a long time | First pull of the ~2–3 GB Splunk image; see §3.5. Docker Hub rate limits: `kubectl -n splunk get events --sort-by=.lastTimestamp` |
| CR phase stays `Error` | `kubectl -n splunk describe <kind> <name>` (3.2.0 adds standard status conditions incl. `TerminalFailure`) and `kubectl -n splunk-operator logs deploy/splunk-operator-controller-manager` |
| Ansible start-up errors inside a Splunk pod | `kubectl -n splunk logs splunk-idxc-indexer-0` (splunk-ansible output) |
| SELinux denials (volumes not writable) | `sudo ausearch -m avc -ts recent`; confirm `k3s-selinux` is installed (`rpm -q k3s-selinux`) and `selinux: true` is set in K3s config |
| Search heads see no peers | `kubectl -n splunk exec splunk-sh1-standalone-0 -- /opt/splunk/bin/splunk list search-server -auth admin:$PW`; confirm `clusterManagerRef` is set |
| UF not forwarding | `outputs.conf` content and `splunkd.log` (§6.5); test DNS: `kubectl -n splunk run t --rm -it --image=busybox:1.36.1 --restart=Never -- nslookup splunk-idxc-indexer-0.splunk-idxc-indexer-headless.splunk.svc.cluster.local` |
| NodePort unreachable from LAN | `sudo firewall-cmd --list-ports`; `kubectl -n splunk get endpoints web-sh1` must list the pod IP |
| Disk filling up | `df -h /var/lib/rancher`; `sudo du -sh /var/lib/rancher/k3s/storage/*`; tune index retention (`maxTotalDataSizeMB`, `frozenTimePeriodInSecs`) via a CM app bundle |

---

## 10. References

- SOK change log (3.2.0 supported versions): [https://splunk.github.io/splunk-operator/reference/ChangeLog.html](https://splunk.github.io/splunk-operator/reference/ChangeLog.html)
- SOK installation (General Terms acceptance, manifests): [https://splunk.github.io/splunk-operator/deploy/Install.html](https://splunk.github.io/splunk-operator/deploy/Install.html)
- SOK Universal Forwarder Helm chart: [https://splunk.github.io/splunk-operator/deploy/uf-helm-chart.html](https://splunk.github.io/splunk-operator/deploy/uf-helm-chart.html)
- SOK custom resources and examples: [https://splunk.github.io/splunk-operator/docs/CustomResources.html](https://splunk.github.io/splunk-operator/docs/CustomResources.html) , [https://splunk.github.io/splunk-operator/docs/Examples.html](https://splunk.github.io/splunk-operator/docs/Examples.html)
- SOK guide on Splunk Docs (3.2): [https://help.splunk.com/en/splunk-enterprise/splunk-operator-for-kubernetes-guide/3.2](https://help.splunk.com/en/splunk-enterprise/splunk-operator-for-kubernetes-guide/3.2)
- Splunk Enterprise 10.6 release notes: [https://help.splunk.com/en/splunk-enterprise/release-notes-and-updates/release-notes/10.6](https://help.splunk.com/en/splunk-enterprise/release-notes-and-updates/release-notes/10.6)
- AlmaLinux 10.2 release notes: [https://wiki.almalinux.org/release-notes/10.2.html](https://wiki.almalinux.org/release-notes/10.2.html)
- K3s v1.36 release notes: [https://docs.k3s.io/release-notes/v1.36.X](https://docs.k3s.io/release-notes/v1.36.X)
- K3s requirements and SELinux: [https://docs.k3s.io/installation/requirements](https://docs.k3s.io/installation/requirements) , [https://docs.k3s.io/advanced#selinux-support](https://docs.k3s.io/advanced#selinux-support)
- Splunk General Terms: [https://www.splunk.com/en_us/legal/splunk-general-terms.html](https://www.splunk.com/en_us/legal/splunk-general-terms.html)
