## Inhoud

0. Overzicht en planning van resources
1. AlmaLinux 10 installeren
2. Basisconfiguratie van het OS
3. RKE2 (Kubernetes) installeren
4. Storage: local-path-provisioner
5. Splunk Operator installeren
6. Splunk-cluster uitrollen (ClusterManager, 3 indexers, SearchHeadCluster, Monitoring Console)
7. Toegang tot Splunk Web
8. Docker Engine installeren
9. Portainer CE installeren
10. (Optioneel) RKE2-cluster koppelen aan Portainer
11. Verificatie-checklist
12. Beheer, back-up en upgrades
13. Troubleshooting

---

## 0. Overzicht en planning van resources

### 0.1 Wat er straks draait

```
AlmaLinux 10 (bare metal)
├── RKE2 (rke2-server, embedded containerd, Canal CNI, ingress-nginx)
│   ├── namespace splunk-operator
│   │   ├── splunk-operator-controller-manager
│   │   ├── splunk-cm-cluster-manager-0
│   │   ├── splunk-idx-indexer-0/1/2
│   │   ├── splunk-shc-deployer-0
│   │   ├── splunk-shc-search-head-0/1/2
│   │   └── splunk-mc-monitoring-console-0
│   ├── namespace local-path-storage
│   └── (optioneel) namespace portainer  → Portainer Kubernetes-agent
└── Docker Engine
    └── portainer (web-UI op https://<ip>:9443)
```

### 0.2 Geheugenbudget (64 GB)

| Onderdeel | Aantal | RAM-limiet per pod | Totaal |
|---|---|---|---|
| Indexers | 3 | 8 Gi | 24 Gi |
| Search heads | 3 | 4 Gi | 12 Gi |
| Cluster Manager | 1 | 3 Gi | 3 Gi |
| Deployer | 1 | 2 Gi | 2 Gi |
| Monitoring Console | 1 | 2 Gi | 2 Gi |
| Splunk Operator | 1 | ~0,5 Gi | 0,5 Gi |
| RKE2 control plane + Canal + ingress | – | – | ~4 Gi |
| Docker + Portainer | – | – | ~1 Gi |
| OS + marge | – | – | ~15 Gi |
| **Totaal** | | | **~64 Gi** |

Dit is krap maar werkbaar voor een lab. Wil je meer lucht, dan is de eenvoudigste besparing: de SearchHeadCluster (3 SH + deployer = 14 Gi) vervangen door één `Standalone` search head met `clusterManagerRef` (4 Gi). Beide varianten staan in hoofdstuk 6.

### 0.3 Disk-indeling (2 TB)

Aanbevolen partitionering tijdens de installatie (LVM, XFS):

| Mountpoint | Grootte | Doel |
|---|---|---|
| `/boot` | 1 GB | |
| `/boot/efi` | 600 MB | (bij UEFI) |
| `/` | 100 GB | OS, `/opt`, `/usr` |
| `/var/lib/rancher` | 200 GB | RKE2, containerd-images, etcd |
| `/var/lib/docker` | 100 GB | Docker-images en -volumes |
| `/opt/local-path-provisioner` | rest (~1,5 TB) | PersistentVolumes voor Splunk |
| swap | geen | Kubernetes vereist swap uit |

Het is ook prima om alles op `/` te zetten; dan is de bovenstaande indeling alleen een advies om te voorkomen dat Splunk-data de root-partitie volschrijft.

### 0.4 Wat je klaar moet hebben liggen

- AlmaLinux 10 ISO (`AlmaLinux-10-latest-x86_64-dvd.iso` of `-minimal.iso`) van https://almalinux.org/get-almalinux/
- USB-stick (8 GB+) en een tool als Rufus, balenaEtcher of `dd`
- Een vast IP-adres (of DHCP-reservering) en een hostnaam, bijv. `splunk-k8s.lan` / `192.168.1.50`
- Een werkstation met SSH-client en browser
- (Optioneel) een Splunk Enterprise-licentiebestand; zonder licentie draait Splunk 60 dagen in trial-modus

---

## 1. AlmaLinux 10 installeren

1. Schrijf de ISO naar de USB-stick en boot de machine ervan (UEFI-modus, Secure Boot mag aan blijven — AlmaLinux is gesigneerd).
2. Kies **Install AlmaLinux 10**.
3. **Taal/toetsenbord:** English (US) als systeemtaal is het handigst voor foutmeldingen; toetsenbord naar wens (`nl` of `us`).
4. **Time & Date:** Europe/Amsterdam, NTP aan.
5. **Software Selection:** *Server* (zonder GUI) of *Minimal Install*. Geen "Container Management"-groep aanvinken (die installeert Podman, wat we niet nodig hebben).
6. **Installation Destination:** kies de 2 TB-disk → *Custom* → maak de indeling uit §0.3 (LVM, XFS). Voeg **geen** swap toe.
7. **Network & Host Name:**
   - Zet de netwerkinterface aan.
   - Hostnaam: `splunk-k8s` (of eigen keuze; **geen** hoofdletters of underscores — Kubernetes-nodenamen moeten DNS-conform zijn).
   - Configure → IPv4 → *Manual* → vast IP, gateway, DNS. (Of laat DHCP staan als je een reservering hebt gemaakt.)
8. **User Creation:** maak een gebruiker aan, bijv. `eric`, vink *Make this user administrator* aan (sudo). Zet een sterk root-wachtwoord.
9. **Security Profile:** geen profiel selecteren.
10. Begin de installatie, reboot en verwijder de USB-stick.

Log daarna in via SSH vanaf je werkstation:

```bash
ssh eric@192.168.1.50
```

Alle volgende commando's draai je als root, tenzij anders aangegeven:

```bash
sudo -i
```

---

## 2. Basisconfiguratie van het OS

### 2.1 Updaten en basistools

```bash
dnf -y update
dnf -y install vim curl wget tar git jq bash-completion policycoreutils-python-utils \
               container-selinux iptables-nft nfs-utils chrony tmux htop
systemctl enable --now chronyd
reboot   # als er een nieuwe kernel is geïnstalleerd
```

### 2.2 Hostnaam en /etc/hosts

```bash
hostnamectl set-hostname splunk-k8s
echo "192.168.1.50  splunk-k8s splunk-k8s.lan" >> /etc/hosts
```

### 2.3 Firewalld uitschakelen

RKE2's standaard-CNI (Canal = Calico + Flannel) conflicteert met firewalld. De officiële RKE2-documentatie schrijft voor firewalld uit te zetten; de kernel-firewall (nftables) blijft actief en wordt door Canal beheerd.

```bash
systemctl disable --now firewalld
systemctl mask firewalld
```

> Sta je de machine bloot aan een onvertrouwd netwerk, beperk dan de toegang op je router/perimeter-firewall. Relevante poorten op deze host: 22 (SSH), 6443 (Kubernetes API), 9443 (Portainer), 80/443 (ingress), 30000–32767 (NodePorts).

### 2.4 NetworkManager: CNI-interfaces met rust laten

```bash
cat > /etc/NetworkManager/conf.d/rke2-canal.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:flannel*;interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
EOF
systemctl reload NetworkManager

# nm-cloud-setup bestaat alleen op cloud-images; uitzetten kan geen kwaad
systemctl disable --now nm-cloud-setup.service nm-cloud-setup.timer 2>/dev/null || true
```

### 2.5 Kernelinstellingen

```bash
cat > /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

cat > /etc/sysctl.d/90-kubernetes.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
# Splunk-aanbevelingen
vm.max_map_count                    = 262144
fs.inotify.max_user_instances       = 8192
fs.inotify.max_user_watches         = 1048576
EOF
sysctl --system
```

### 2.6 Swap controleren

```bash
swapon --show        # moet leeg zijn
sed -i '/ swap / s/^/#/' /etc/fstab   # alleen nodig als er toch swap in staat
```

### 2.7 Transparent Huge Pages uit (Splunk-aanbeveling)

```bash
cat > /etc/systemd/system/disable-thp.service <<'EOF'
[Unit]
Description=Disable Transparent Huge Pages
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled; echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF
systemctl daemon-reload
systemctl enable --now disable-thp.service
cat /sys/kernel/mm/transparent_hugepage/enabled   # verwacht: always madvise [never]
```

### 2.8 SELinux

Laat SELinux op **Enforcing** staan. RKE2 installeert via de RPM-methode automatisch het `rke2-selinux`-beleid. Controleer:

```bash
getenforce   # Enforcing
```

---

## 3. RKE2 (Kubernetes) installeren

### 3.1 Configuratiebestand aanmaken (vóór de installatie)

```bash
mkdir -p /etc/rancher/rke2
cat > /etc/rancher/rke2/config.yaml <<'EOF'
# Kubeconfig leesbaar voor niet-root gebruikers
write-kubeconfig-mode: "0644"

# Extra namen/IP's in het API-server-certificaat
tls-san:
  - splunk-k8s
  - splunk-k8s.lan
  - 192.168.1.50

# Single node: geen taint op de control-plane, zodat Splunk-pods hier mogen draaien
node-label:
  - "node-role.kubernetes.io/splunk=true"

# Standaardwaarden expliciet (mag je weglaten)
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16

# Kubelet: reserveer wat resources voor het OS
kubelet-arg:
  - "system-reserved=cpu=1,memory=2Gi"
  - "kube-reserved=cpu=500m,memory=1Gi"
  - "max-pods=110"
EOF
```

> RKE2 zet standaard **geen** NoSchedule-taint op server-nodes, dus workloads mogen op deze single node landen. Er is dus niets extra's nodig.

### 3.2 Installeren

Het installatiescript detecteert een RPM-gebaseerd systeem en gebruikt automatisch de RPM-methode (inclusief `rke2-selinux`). We pinnen op het `v1.34`-kanaal omdat Splunk Operator 3.1.0 tot en met Kubernetes 1.34 ondersteunt.

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.34 INSTALL_RKE2_TYPE=server sh -
```

Controleer wat er is geïnstalleerd:

```bash
rpm -qa | grep rke2      # rke2-server, rke2-common, rke2-selinux
cat /etc/sysconfig/rke2-server   # bevat RKE2_SELINUX=true
```

### 3.3 Starten

```bash
systemctl enable --now rke2-server.service
journalctl -u rke2-server -f     # volg de logs; Ctrl-C als 'rke2 is up and running' verschijnt
```

De eerste start duurt 2–5 minuten (images worden gedownload).

### 3.4 kubectl instellen

```bash
# Voor root
mkdir -p ~/.kube
ln -sf /etc/rancher/rke2/rke2.yaml ~/.kube/config
ln -sf /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
ln -sf /var/lib/rancher/rke2/bin/crictl  /usr/local/bin/crictl
ln -sf /var/lib/rancher/rke2/bin/ctr     /usr/local/bin/ctr

cat > /etc/profile.d/rke2.sh <<'EOF'
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin
source <(kubectl completion bash) 2>/dev/null
EOF
source /etc/profile.d/rke2.sh
```

Voor je gewone gebruiker (`eric`) werkt hetzelfde profile-script, omdat de kubeconfig met `0644` is weggeschreven.

### 3.5 Helm installeren (handig voor later)

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 3.6 Controle

```bash
kubectl get nodes -o wide
# NAME         STATUS   ROLES                       VERSION
# splunk-k8s   Ready    control-plane,etcd,master   v1.34.x+rke2r1

kubectl get pods -A
# Alle pods in kube-system Running/Completed:
# rke2-canal-*, rke2-coredns-*, rke2-ingress-nginx-controller-*, rke2-metrics-server-*, etcd-*, kube-apiserver-* ...
```

Wil je de kubeconfig op je werkstation gebruiken:

```bash
# op je werkstation
scp eric@192.168.1.50:/etc/rancher/rke2/rke2.yaml ~/.kube/splunk-k8s.yaml
sed -i 's/127.0.0.1/192.168.1.50/' ~/.kube/splunk-k8s.yaml
export KUBECONFIG=~/.kube/splunk-k8s.yaml
kubectl get nodes
```

---

## 4. Storage: local-path-provisioner

RKE2 levert (anders dan k3s) geen storage-provisioner mee. Splunk Operator heeft een StorageClass nodig voor de `etc`- en `var`-volumes van elke pod. Voor een single node is Rancher's local-path-provisioner de eenvoudigste keuze: elke PVC wordt een map onder `/opt/local-path-provisioner`.

```bash
mkdir -p /opt/local-path-provisioner
# SELinux-context zodat containers erin mogen schrijven
semanage fcontext -a -t container_file_t "/opt/local-path-provisioner(/.*)?"
restorecon -Rv /opt/local-path-provisioner

kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.36/deploy/local-path-storage.yaml

# Maak 'local-path' de standaard StorageClass
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass
# NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer
```

Snelle test:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: sh
    image: busybox
    command: ["sh","-c","echo ok > /data/ok && sleep 3600"]
    volumeMounts: [{ name: data, mountPath: /data }]
  volumes:
  - name: data
    persistentVolumeClaim: { claimName: test-pvc }
EOF
sleep 20
kubectl exec test-pod -- cat /data/ok      # ok
kubectl delete pod test-pod; kubectl delete pvc test-pvc
```

> **Let op:** local-path-provisioner heeft `reclaimPolicy: Delete`. Verwijder je een Splunk-CR (met de `enterprise.splunk.com/delete-pvc`-finalizer), dan is de data weg. Zie §12 voor back-up.

---

## 5. Splunk Operator installeren

### 5.1 Operator-manifest downloaden en aanpassen

Sinds Operator 3.0.0 **moet** je de Splunk General Terms accepteren via de omgevingsvariabele `SPLUNK_GENERAL_TERMS`, anders starten Splunk 10.x-containers niet. Bovendien zetten we de Splunk-image op 10.4.3 (vereist voor Kubernetes 1.34).

```bash
mkdir -p ~/splunk && cd ~/splunk
wget -O splunk-operator-cluster.yaml \
  https://github.com/splunk/splunk-operator/releases/download/3.1.0/splunk-operator-cluster.yaml

# 1) General Terms accepteren
sed -i '/name: SPLUNK_GENERAL_TERMS/{n;s|value: ""|value: "--accept-sgt-current-at-splunk-com"|}' splunk-operator-cluster.yaml

# 2) Splunk Enterprise image op 10.4.3
sed -i 's|value: splunk/splunk:10.2.0|value: splunk/splunk:10.4.3|' splunk-operator-cluster.yaml

# Controle
grep -n -A1 'name: SPLUNK_GENERAL_TERMS\|name: RELATED_IMAGE_SPLUNK_ENTERPRISE' splunk-operator-cluster.yaml
```

Verwacht:

```
- name: RELATED_IMAGE_SPLUNK_ENTERPRISE
  value: splunk/splunk:10.4.3
- name: SPLUNK_GENERAL_TERMS
  value: "--accept-sgt-current-at-splunk-com"
```

> Lees de Splunk General Terms op https://www.splunk.com/en_us/legal/splunk-general-terms.html voordat je dit accepteert.

### 5.2 Toepassen

De CRD's zijn groot; gebruik daarom `--server-side` (zoals de Splunk-documentatie voorschrijft).

```bash
kubectl apply -f splunk-operator-cluster.yaml --server-side
kubectl -n splunk-operator get pods -w
# splunk-operator-controller-manager-xxxx   1/1   Running
```

```bash
kubectl get crd | grep enterprise.splunk.com
# clustermanagers, indexerclusters, licensemanagers, monitoringconsoles,
# searchheadclusters, standalones, ... (11 CRD's)
```

### 5.3 Splunk-image alvast binnenhalen (optioneel, scheelt wachttijd)

```bash
crictl pull docker.io/splunk/splunk:10.4.3
```

---

## 6. Splunk-cluster uitrollen

We rollen uit in namespace `splunk-operator` (waar de operator al draait; cluster-wide mag het ook in een eigen namespace). Volgorde is belangrijk: eerst Monitoring Console en ClusterManager, dan de IndexerCluster, dan de search-laag.

### 6.1 (Optioneel) Licentie

Sla dit over als je de 60-daagse trial wilt gebruiken.

```bash
kubectl -n splunk-operator create configmap splunk-licenses --from-file=enterprise.lic

cat <<'EOF' | kubectl apply -f -
apiVersion: enterprise.splunk.com/v4
kind: LicenseManager
metadata:
  name: lm
  namespace: splunk-operator
  finalizers:
  - enterprise.splunk.com/delete-pvc
spec:
  volumes:
    - name: licenses
      configMap:
        name: splunk-licenses
  licenseUrl: /mnt/licenses/enterprise.lic
  etcVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 5Gi }
  varVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 10Gi }
  resources:
    requests: { cpu: 250m, memory: 1Gi }
    limits:   { cpu: "1",  memory: 2Gi }
EOF
```

Voeg dan aan elk CR hieronder `licenseManagerRef: { name: lm }` toe in de `spec`.

### 6.2 Monitoring Console + ClusterManager

```bash
cat > 01-mc-cm.yaml <<'EOF'
apiVersion: enterprise.splunk.com/v4
kind: MonitoringConsole
metadata:
  name: mc
  namespace: splunk-operator
  finalizers:
  - enterprise.splunk.com/delete-pvc
spec:
  etcVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 5Gi }
  varVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 20Gi }
  resources:
    requests: { cpu: 250m, memory: 1Gi }
    limits:   { cpu: "1",  memory: 2Gi }
---
apiVersion: enterprise.splunk.com/v4
kind: ClusterManager
metadata:
  name: cm
  namespace: splunk-operator
  finalizers:
  - enterprise.splunk.com/delete-pvc
spec:
  monitoringConsoleRef:
    name: mc
  etcVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 10Gi }
  varVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 30Gi }
  resources:
    requests: { cpu: 500m, memory: 1Gi }
    limits:   { cpu: "2",  memory: 3Gi }
  defaults: |-
    splunk:
      conf:
        - key: server
          value:
            directory: /opt/splunk/etc/system/local
            content:
              clustering:
                replication_factor: 2
                search_factor: 2
EOF
kubectl apply -f 01-mc-cm.yaml
kubectl -n splunk-operator get pods -w
```

Wacht tot `splunk-mc-monitoring-console-0` en `splunk-cm-cluster-manager-0` op `1/1 Running` staan (eerste keer 3–6 minuten). Controleer de fase:

```bash
kubectl -n splunk-operator get clustermanager cm
# NAME   PHASE   ...
# cm     Ready
```

> Replication factor en search factor staan op 2 (in plaats van de Splunk-standaard 3 op 3 peers) om schijfgebruik en I/O op één fysieke disk te beperken. Alle drie de indexers staan toch op dezelfde disk, dus redundantie is hier vooral functioneel (clustergedrag oefenen), niet voor dataveiligheid.

### 6.3 IndexerCluster (3 peers)

```bash
cat > 02-idx.yaml <<'EOF'
apiVersion: enterprise.splunk.com/v4
kind: IndexerCluster
metadata:
  name: idx
  namespace: splunk-operator
  finalizers:
  - enterprise.splunk.com/delete-pvc
spec:
  replicas: 3
  clusterManagerRef:
    name: cm
  monitoringConsoleRef:
    name: mc
  etcVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 10Gi }
  varVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 250Gi }
  resources:
    requests: { cpu: "1", memory: 4Gi }
    limits:   { cpu: "4", memory: 8Gi }
  # HEC/S2S via NodePort bereikbaar maken (optioneel)
  serviceTemplate:
    spec:
      type: NodePort
EOF
kubectl apply -f 02-idx.yaml
kubectl -n splunk-operator get pods -l app.kubernetes.io/instance=splunk-idx-indexer -w
```

Wacht op `splunk-idx-indexer-0/1/2` Running en:

```bash
kubectl -n splunk-operator get indexercluster idx
# NAME   PHASE   MANAGER   DESIRED   READY   ...
# idx    Ready   Ready     3         3
```

### 6.4 Search-laag — kies A of B

**Variant A: SearchHeadCluster (3 search heads + deployer, ~14 Gi RAM)**

```bash
cat > 03-shc.yaml <<'EOF'
apiVersion: enterprise.splunk.com/v4
kind: SearchHeadCluster
metadata:
  name: shc
  namespace: splunk-operator
  finalizers:
  - enterprise.splunk.com/delete-pvc
spec:
  replicas: 3
  clusterManagerRef:
    name: cm
  monitoringConsoleRef:
    name: mc
  etcVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 10Gi }
  varVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 50Gi }
  resources:
    requests: { cpu: 500m, memory: 2Gi }
    limits:   { cpu: "2",  memory: 4Gi }
EOF
kubectl apply -f 03-shc.yaml
```

De operator maakt automatisch een deployer (`splunk-shc-deployer-0`) en drie search heads (`splunk-shc-search-head-0/1/2`). Het SHC-bootstrappen duurt ~5–10 minuten.

**Variant B: één Standalone search head (~4 Gi RAM, lichter)**

```bash
cat > 03-sh.yaml <<'EOF'
apiVersion: enterprise.splunk.com/v4
kind: Standalone
metadata:
  name: sh
  namespace: splunk-operator
  finalizers:
  - enterprise.splunk.com/delete-pvc
spec:
  clusterManagerRef:
    name: cm
  monitoringConsoleRef:
    name: mc
  etcVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 10Gi }
  varVolumeStorageConfig: { storageClassName: local-path, storageCapacity: 50Gi }
  resources:
    requests: { cpu: 500m, memory: 2Gi }
    limits:   { cpu: "2",  memory: 4Gi }
EOF
kubectl apply -f 03-sh.yaml
```

### 6.5 Eindcontrole

```bash
kubectl -n splunk-operator get pods
kubectl -n splunk-operator get clustermanager,indexercluster,searchheadcluster,standalone,monitoringconsole
kubectl -n splunk-operator get pvc
kubectl top pods -n splunk-operator      # via rke2-metrics-server
```

Alle CR's moeten `PHASE: Ready` melden.

### 6.6 Admin-wachtwoord ophalen

De operator genereert de wachtwoorden en zet ze in één namespace-breed secret:

```bash
kubectl -n splunk-operator get secret splunk-splunk-operator-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

(Naamgeving: `splunk-<namespace>-secret`.) Gebruikersnaam is `admin`.

---

## 7. Toegang tot Splunk Web

### 7.1 Snel: port-forward

```bash
# Search head cluster (variant A)
kubectl -n splunk-operator port-forward --address 0.0.0.0 svc/splunk-shc-search-head-service 8000:8000
# of Standalone (variant B)
kubectl -n splunk-operator port-forward --address 0.0.0.0 svc/splunk-sh-standalone-service 8000:8000
```

Open http://192.168.1.50:8000 en log in als `admin`.

### 7.2 Blijvend: Ingress via de meegeleverde ingress-nginx

Voeg op je werkstation (of in je lokale DNS) toe:

```
192.168.1.50  splunk.lan cm.splunk.lan mc.splunk.lan
```

```bash
cat > 04-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: splunk-web
  namespace: splunk-operator
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/affinity: "cookie"     # sticky sessions voor SHC
spec:
  ingressClassName: nginx
  rules:
  - host: splunk.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: splunk-shc-search-head-service   # variant B: splunk-sh-standalone-service
            port: { number: 8000 }
  - host: cm.splunk.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: splunk-cm-cluster-manager-service
            port: { number: 8000 }
  - host: mc.splunk.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: splunk-mc-monitoring-console-service
            port: { number: 8000 }
EOF
kubectl apply -f 04-ingress.yaml
kubectl -n splunk-operator get ingress
```

Nu bereikbaar op http://splunk.lan, http://cm.splunk.lan en http://mc.splunk.lan. Wil je TLS, installeer dan cert-manager en voeg een `tls:`-sectie toe.

### 7.3 Data insturen (HEC)

Met `serviceTemplate.type: NodePort` op de IndexerCluster:

```bash
kubectl -n splunk-operator get svc splunk-idx-indexer-service
# Zoek de NodePort die op 8088 (HEC) is gemapt, bijv. 8088:31088

# HEC-token staat in hetzelfde secret
kubectl -n splunk-operator get secret splunk-splunk-operator-secret -o jsonpath='{.data.hec_token}' | base64 -d; echo

curl -k https://192.168.1.50:31088/services/collector/event \
  -H "Authorization: Splunk <hec_token>" \
  -d '{"event": "hello from k8s", "sourcetype": "manual"}'
```

Zoek in Splunk Web: `index=main sourcetype=manual`.

---

## 8. Docker Engine installeren

Docker publiceert geen aparte AlmaLinux-repo; de CentOS-repo (met `$releasever` = 10) werkt voor AlmaLinux 10.

```bash
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Voordat je Docker start: zorg dat Docker's iptables-regels de Kubernetes-verkeer niet hinderen en dat Docker de `nftables`-backend van EL10 gebruikt. Een minimale daemon-config:

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" },
  "default-address-pools": [
    { "base": "172.17.0.0/16", "size": 24 }
  ],
  "live-restore": true
}
EOF
```

> `default-address-pools` houdt Docker weg van `10.42.0.0/16` en `10.43.0.0/16` (RKE2). Docker's standaardpool `172.17.0.0/16` overlapt hier niet mee.

```bash
systemctl enable --now docker
docker version
docker run --rm hello-world
```

Niet-root gebruik:

```bash
usermod -aG docker eric
# opnieuw inloggen (of: newgrp docker)
```

Controleer na het starten van Docker dat Kubernetes nog gezond is (Docker voegt iptables/nftables-regels toe, Canal moet daar niet van schrikken):

```bash
kubectl get pods -A | grep -v Running | grep -v Completed   # verwacht: leeg
kubectl -n splunk-operator exec splunk-cm-cluster-manager-0 -- curl -s -o /dev/null -w '%{http_code}\n' http://splunk-idx-indexer-0.splunk-idx-indexer-headless:8000   # 200/303
```

---

## 9. Portainer CE installeren

```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts

docker ps --filter name=portainer
```

> Poort 8000 is alleen nodig voor Portainer Edge-agents; je mag `-p 8000:8000` weglaten. Let op: dit is dezelfde poort als Splunk Web bij een port-forward (§7.1) — gebruik dan een andere lokale poort voor de port-forward (bijv. `8001:8000`), of laat 8000 bij Portainer weg.

Open binnen **5 minuten** na de start https://192.168.1.50:9443 (zelfondertekend certificaat — accepteer de waarschuwing) en maak het admin-account aan. Ben je te laat, dan vergrendelt Portainer zich uit veiligheid; herstart dan de container:

```bash
docker restart portainer
```

Na het aanmaken van het account: kies **Get Started** → de lokale Docker-omgeving (`local`) is direct beschikbaar. Vanaf hier beheer je containers, images, volumes, netwerken en Compose-stacks via de browser.

### 9.1 Portainer-updates

```bash
docker stop portainer && docker rm portainer
docker pull portainer/portainer-ce:lts
# herhaal het docker run-commando van hierboven; de data staat in het volume
```

---

## 10. (Optioneel) RKE2-cluster koppelen aan Portainer

Portainer kan ook je Kubernetes-cluster tonen en beheren (namespaces, workloads, logs, shell in pods).

1. In Portainer: **Environments → Add environment → Kubernetes → Start Wizard**.
2. Kies **Agent** → tab **Kubernetes via node port**.
3. Portainer toont een `kubectl apply -f https://downloads.portainer.io/...portainer-agent-k8s-nodeport.yaml`-commando. Voer dat uit op de server:

```bash
kubectl apply -f <de URL uit de wizard>
kubectl -n portainer get pods    # portainer-agent-xxxx Running
kubectl -n portainer get svc     # NodePort 30778
```

4. Terug in de wizard: naam `rke2-splunk`, Environment address `192.168.1.50:30778` → **Connect**.

---

## 11. Verificatie-checklist

```bash
# OS
getenforce                                  # Enforcing
systemctl is-active firewalld               # inactive
swapon --show                               # leeg
cat /sys/kernel/mm/transparent_hugepage/enabled   # [never]

# RKE2
systemctl is-active rke2-server             # active
kubectl get nodes                           # Ready
kubectl get pods -A | grep -vE 'Running|Completed'   # leeg
kubectl get storageclass                    # local-path (default)

# Splunk
kubectl -n splunk-operator get pods         # allemaal 1/1 Running
kubectl -n splunk-operator get clustermanager,indexercluster,searchheadcluster,monitoringconsole
                                            # allemaal Ready
kubectl -n splunk-operator exec splunk-cm-cluster-manager-0 -- \
  /opt/splunk/bin/splunk show cluster-status -auth admin:"$(kubectl -n splunk-operator get secret splunk-splunk-operator-secret -o jsonpath='{.data.password}' | base64 -d)"
                                            # 3 peers Up, RF/SF met

# Docker / Portainer
docker info | grep -E 'Server Version|Cgroup|Storage Driver'
docker ps                                   # portainer Up
curl -k -s -o /dev/null -w '%{http_code}\n' https://127.0.0.1:9443/   # 200
```

Functionele test: stuur een event via HEC (§7.3), zoek het op in Splunk Web, en bekijk in de Monitoring Console (mc.splunk.lan → Settings → Monitoring Console → Indexing → Indexer Clustering) of alle peers zichtbaar zijn.

---

## 12. Beheer, back-up en upgrades

### 12.1 Dagelijkse commando's

```bash
kubectl -n splunk-operator logs -f splunk-idx-indexer-0            # Splunk-logs
kubectl -n splunk-operator exec -it splunk-shc-search-head-0 -- bash
kubectl -n splunk-operator logs -f deploy/splunk-operator-controller-manager
journalctl -u rke2-server -f
crictl ps ; crictl images                                          # RKE2's containerd
docker ps ; docker images                                          # Docker
```

### 12.2 Splunk netjes stoppen/starten (bijv. voor onderhoud)

De operator beheert StatefulSets; schaal niet handmatig. Wil je de hele node herstarten: `reboot` volstaat — RKE2 en Docker starten automatisch, en de operator brengt Splunk terug in de laatste staat. Volgorde bij herstart: CM → indexers → SH; even geduld (5–10 min).

### 12.3 Back-up

- **etcd (RKE2):** automatische snapshots elke 12 uur in `/var/lib/rancher/rke2/server/db/snapshots/`. Handmatig: `rke2 etcd-snapshot save`.
- **Splunk-data:** staat in `/opt/local-path-provisioner/pvc-*_splunk-operator_*`. Maak een filesystem-back-up (rsync/borg/restic) naar een externe schijf of NAS. Voor consistente back-ups: gebruik Splunk's eigen mechanismen (frozen buckets naar een archiefpad, of SmartStore naar S3-compatibele storage zoals MinIO).
- **Manifesten:** bewaar `~/splunk/*.yaml` in git.
- **Portainer:** `docker run --rm -v portainer_data:/data -v $PWD:/backup busybox tar czf /backup/portainer_data.tgz /data` (of Settings → Backup in de UI).

### 12.4 Upgrades

| Wat | Hoe |
|---|---|
| AlmaLinux | `dnf update && reboot` (RKE2/Docker starten vanzelf) |
| RKE2 patch-release | `curl -sfL https://get.rke2.io \| INSTALL_RKE2_CHANNEL=v1.34 sh -` en `systemctl restart rke2-server` |
| RKE2 minor (bijv. 1.35) | **Eerst** controleren of de Splunk Operator die K8s-versie ondersteunt (ChangeLog) |
| Splunk Operator | Nieuwe manifest downloaden, dezelfde twee `sed`-aanpassingen doen, `kubectl apply --server-side`; de operator rolt de Splunk-pods vervolgens in de juiste volgorde (CM → indexers → SH) door |
| Splunk Enterprise | `RELATED_IMAGE_SPLUNK_ENTERPRISE` in het operator-deployment aanpassen, of per CR `spec.image` zetten |
| Docker | `dnf update docker-ce` |
| Portainer | §9.1 |

Splunk Operator: lees vóór elke upgrade de upgrade-notities op https://splunk.github.io/splunk-operator/ChangeLog.html; grote sprongen (bijv. 2.x → 3.x) vragen extra stappen.

---

## 13. Troubleshooting

| Symptoom | Oorzaak / oplossing |
|---|---|
| `rke2-server` start niet, log meldt iets over `firewalld` of CNI-routes | Firewalld nog actief of NetworkManager beheert `cali*`-interfaces → §2.3 / §2.4, daarna `reboot` |
| Pods blijven `ContainerCreating` met SELinux-denials (`ausearch -m avc -ts recent`) | `rke2-selinux` ontbreekt (tarball-install?) → `dnf install rke2-selinux`, of context van `/opt/local-path-provisioner` niet gezet → §4 |
| PVC blijft `Pending` | Geen default StorageClass, of local-path-provisioner niet Running → `kubectl -n local-path-storage get pods` |
| Splunk-pod crasht direct, log: *"You must accept the Splunk General Terms"* | `SPLUNK_GENERAL_TERMS` niet gezet in het operator-deployment → §5.1, daarna `kubectl -n splunk-operator rollout restart deploy/splunk-operator-controller-manager` en de Splunk-pods verwijderen zodat ze opnieuw starten |
| Splunk-pod `OOMKilled` | Verhoog `resources.limits.memory` in het CR (indexers hebben bij zwaar zoekwerk >8 Gi nodig) of kies variant B in §6.4 |
| IndexerCluster blijft in `Pending`/`ScalingUp` | Cluster Manager nog niet `Ready`; check `kubectl -n splunk-operator logs splunk-cm-cluster-manager-0` en of `replication_factor` ≤ aantal peers |
| SHC blijft in `Pending` | Deployer moet eerst klaar zijn; SHC-bootstrap kan 10 minuten duren. Log: `kubectl -n splunk-operator logs splunk-shc-search-head-0` |
| Splunk Web via ingress geeft 502/504 | Service-naam of poort klopt niet: `kubectl -n splunk-operator get svc` |
| Portainer: *"Your Portainer instance timed out for security purposes"* | `docker restart portainer` en binnen 5 min het admin-account aanmaken |
| Na starten van Docker: pods kunnen elkaar/DNS niet bereiken | Docker heeft `FORWARD`-policy op DROP gezet. Fix: `iptables -P FORWARD ACCEPT` en permanent: `"iptables": true` laten staan maar in `/etc/systemd/system/docker.service.d/override.conf` een `ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT` toevoegen; of `"ip-forward-no-drop": true` in `daemon.json` (Docker ≥ 28) |
| Schijf vol | `crictl rmi --prune` (RKE2-images), `docker system prune`, en controleer `du -sh /opt/local-path-provisioner/*`. Splunk: verlaag `maxTotalDataSizeMB` per index of zet `frozenTimePeriodInSecs` |
| Node `NotReady` na reboot | `journalctl -u rke2-server -b` ; meestal wachten (etcd + containerd herstart) of THP/sysctl niet toegepast |

Handige diagnose-commando's:

```bash
kubectl describe pod -n splunk-operator <pod>          # events onderaan
kubectl get events -n splunk-operator --sort-by=.lastTimestamp | tail -30
kubectl -n splunk-operator get <kind> <naam> -o yaml | yq '.status'
rke2 --version ; crictl info | head -40
```

---

## Bronnen

- AlmaLinux 10: https://almalinux.org/get-almalinux/ en https://wiki.almalinux.org/release-notes/
- RKE2 requirements: https://docs.rke2.io/install/requirements
- RKE2 quickstart en configuratie: https://docs.rke2.io/install/quickstart , https://docs.rke2.io/install/configuration
- RKE2 known issues (firewalld, NetworkManager): https://docs.rke2.io/known_issues
- RKE2 SELinux: https://docs.rke2.io/security/selinux
- RKE2 releases: https://github.com/rancher/rke2/releases
- local-path-provisioner: https://github.com/rancher/local-path-provisioner
- Splunk Operator (home): https://splunk.github.io/splunk-operator/
- Splunk Operator install: https://splunk.github.io/splunk-operator/Install.html
- Splunk Operator examples (CR-voorbeelden): https://splunk.github.io/splunk-operator/Examples.html
- Splunk Operator custom resources: https://splunk.github.io/splunk-operator/CustomResources.html
- Splunk Operator storage guidance: https://splunk.github.io/splunk-operator/StorageClass.html
- Splunk Operator 3.1.0 release notes (K8s/Splunk-compatibiliteit): https://github.com/splunk/splunk-operator/releases/tag/3.1.0
- Splunk Operator ChangeLog: https://splunk.github.io/splunk-operator/ChangeLog.html
- Splunk Enterprise image tags: https://hub.docker.com/r/splunk/splunk/tags
- Docker op AlmaLinux 10: https://linuxiac.com/how-to-install-docker-on-almalinux-10/ en https://docs.docker.com/engine/install/centos/
- Portainer CE installatie: https://docs.portainer.io/start/install-ce/server/docker/linux
- Portainer Kubernetes-agent: https://docs.portainer.io/admin/environments/add/kubernetes/agent
