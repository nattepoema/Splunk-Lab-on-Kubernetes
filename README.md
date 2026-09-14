# Splunk Lab on Kubernetes

Deze repository beschrijft de installatie van Splunk Operator for Kubernetes op een NUC.
De specificaties van de gebruikte NUC zijn:
- Minisforum UM790 Pro
- AMD Risen 9 7940HS (8 Core / 16 Threads)
- 64GB DDR5 RAM
- 2 x 1TB NVMe

**Gebruikte software**
| Component | Versie | Opmerking |
|---|---|---|
| AlmaLinux | 10.x (laatste point-release) | RHEL 10-compatibel; wordt door RKE2 als gevalideerd OS vermeld |
| RKE2 | v1.34.x (kanaal `v1.34`) | Hoogste Kubernetes-minor die Splunk Operator 3.1.0 ondersteunt |
| Splunk Operator | 3.1.0 | Ondersteunt Kubernetes 1.25 – 1.34 |
| Splunk Enterprise image | `splunk/splunk:10.4.3` | Vereist voor K8s 1.34 (10.0.4+ of 10.4+) |
| local-path-provisioner | v0.0.36 | Dynamische PersistentVolumes op lokale disk |
| Docker Engine | laatste `docker-ce` uit de officiële repo | |
| Portainer CE | `portainer/portainer-ce:lts` | Beheert Docker én (optioneel) het RKE2-cluster |

> **Ontwerpkeuze — twee container-runtimes op één host.** RKE2 gebruikt zijn eigen ingebouwde containerd (`/var/lib/rancher/rke2`) en raakt Docker niet aan. Docker Engine draait er los naast, met Portainer als web-UI. Beide gebruiken verschillende netwerken (Docker: `172.17.0.0/16`, RKE2: pods `10.42.0.0/16`, services `10.43.0.0/16`), dus er is geen overlap.
