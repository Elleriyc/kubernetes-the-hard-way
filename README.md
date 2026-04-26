# Kubernetes The Hard Way — Home Lab sur Beelink SER5 MAX

> Installation complète d'un cluster Kubernetes from scratch sur un serveur physique, en suivant le guide [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) de Kelsey Hightower.

---

## 🎯 Introduction

Ce projet documente l'installation manuelle d'un cluster Kubernetes sans aucun outil d'automatisation (pas de kubeadm, pas de k3s). L'objectif était de comprendre en profondeur chaque composant de Kubernetes en le configurant à la main.

L'infrastructure tourne sur un **Beelink SER5 MAX** (mini-PC AMD Ryzen) faisant office de serveur home lab, avec 4 machines virtuelles KVM simulant un vrai cluster.

---

## 🖥️ Matériel

| Composant | Détails |
|---|---|
| Machine physique | Beelink SER5 MAX |
| CPU | AMD Ryzen 7 6800U |
| RAM | 32 GB LPDDR5 |
| Stockage | 1 TB NVMe PCIe 4.0 |
| OS hôte | Debian 13 (Trixie) |
| Réseau | RTL8125 2.5GbE (driver r8125-dkms) |

---

## 🏗️ Architecture

### Réseau physique
| Machine | IP fixe |
|---|---|
| `kitana-master` (hôte) | `<HOST_IP>` |

### Machines virtuelles (KVM)
| VM | IP | Rôle | Pod Subnet |
|---|---|---|---|
| `jumpbox` | `<JUMPBOX_IP>` | Administration | - |
| `server` | `<SERVER_IP>` | Control Plane | - |
| `node-0` | `<NODE_0_IP>` | Worker Node | `10.200.0.0/24` |
| `node-1` | `<NODE_1_IP>` | Worker Node | `10.200.1.0/24` |

### Composants Kubernetes installés
```
Control Plane (server)          Worker Nodes (node-0, node-1)
├── kube-apiserver              ├── kubelet
├── kube-controller-manager     ├── kube-proxy
├── kube-scheduler              ├── containerd
└── etcd                        ├── runc
                                └── cni-plugins
```

---

## ⚙️ Prérequis

- Debian 12+ installé sur les VMs
- `kubectl` installé sur la jumpbox
- `openssl` pour la génération des certificats
- `KVM/libvirt` sur l'hôte pour les VMs

---

## 🚀 Installation

### 1. Configuration de l'hôte (kitana-master)

#### Driver Ethernet RTL8125
Le Beelink SER5 MAX utilise un chipset Realtek RTL8125 2.5GbE qui nécessite le driver `r8125` spécifique — le driver `r8169` intégré au kernel est incompatible.

```bash
# Activer le dépôt non-free
sudo nano /etc/apt/sources.list
# Ajouter : contrib non-free non-free-firmware

sudo apt update
sudo apt install linux-headers-$(uname -r) r8125-dkms

# Blacklister l'ancien driver incompatible
echo "blacklist r8169" | sudo tee /etc/modprobe.d/blacklist-r8169.conf
sudo update-initramfs -u
sudo reboot
```

#### IP fixe
```bash
# /etc/network/interfaces
auto eno1
iface eno1 inet static
    address 192.168.1.200
    netmask 255.255.255.0
    gateway 192.168.1.254
    dns-nameservers 192.168.1.254 1.1.1.1 8.8.8.8
```

#### Désactivation du swap (obligatoire pour Kubernetes)
```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

---

### 2. Infrastructure KVM

#### Création des images disque
```bash
# Image de base Debian 12
sudo qemu-img create -f qcow2 -b debian-12-generic-amd64.qcow2 -F qcow2 jumpbox.qcow2 10G
sudo qemu-img create -f qcow2 -b debian-12-generic-amd64.qcow2 -F qcow2 server.qcow2 20G
sudo qemu-img create -f qcow2 -b debian-12-generic-amd64.qcow2 -F qcow2 node-0.qcow2 20G
sudo qemu-img create -f qcow2 -b debian-12-generic-amd64.qcow2 -F qcow2 node-1.qcow2 20G
```

#### Personnalisation des images avec virt-customize
```bash
for vm in jumpbox server node-0 node-1; do
  sudo virt-customize \
    -a /var/lib/libvirt/images/${vm}.qcow2 \
    --root-password password:debian \
    --hostname ${vm} \
    --run-command 'useradd -m -s /bin/bash debian' \
    --run-command 'echo "debian:debian" | chpasswd' \
    --run-command 'usermod -aG sudo debian' \
    --ssh-inject debian:file:/home/elleriyc/.ssh/id_ed25519.pub
done
```

---

### 3. PKI et Certificats TLS

Génération d'une Certificate Authority (CA) auto-signée et des certificats pour chaque composant Kubernetes :

- `ca.crt` / `ca.key` — Certificate Authority du cluster
- `kube-api-server.crt` — Certificat de l'API Server
- `kubelet.crt` — Certificat par nœud worker
- `kube-proxy.crt` — Certificat du kube-proxy
- `kube-controller-manager.crt` — Certificat du controller manager
- `kube-scheduler.crt` — Certificat du scheduler
- `admin.crt` — Certificat administrateur
- `service-accounts.crt` — Certificat pour les service accounts

---

### 4. Kubeconfigs

Génération des fichiers kubeconfig pour chaque composant permettant l'authentification mutuelle via certificats TLS :

```bash
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig
  # ...
done
```

---

### 5. Chiffrement des données au repos

Génération d'une clé de chiffrement AES pour les Secrets Kubernetes stockés dans etcd :

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
envsubst < configs/encryption-config.yaml > encryption-config.yaml
```

---

### 6. etcd

Installation et configuration de la base de données clé-valeur du cluster sur `server` :

```bash
systemctl enable etcd
systemctl start etcd
etcdctl member list
```

---

### 7. Control Plane

Installation sur `server` des 3 composants du control plane :

```bash
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
systemctl start kube-apiserver kube-controller-manager kube-scheduler
kubectl cluster-info --kubeconfig admin.kubeconfig
```

---

### 8. Worker Nodes

Installation sur `node-0` et `node-1` des composants workers :

```bash
systemctl enable containerd kubelet kube-proxy
systemctl start containerd kubelet kube-proxy
```

---

### 9. Routes réseau des pods

Configuration du routage inter-nœuds pour permettre la communication entre pods sur des nœuds différents :

```bash
# Sur server
ip route add 10.200.0.0/24 via <NODE_0_IP>
ip route add 10.200.1.0/24 via <NODE_1_IP>

# Sur node-0
ip route add 10.200.1.0/24 via <NODE_1_IP>

# Sur node-1
ip route add 10.200.0.0/24 via <NODE_0_IP>
```

---

## 🐛 Difficultés rencontrées

### 1. Driver Ethernet RTL8125 non fonctionnel
**Symptôme** : Interface réseau en `NO-CARRIER` puis disparition de l'interface.  
**Cause** : Le driver `r8169` du kernel est incompatible avec le chipset RTL8125.  
**Solution** : Installation du driver `r8125-dkms` depuis les dépôts `non-free` de Debian et blacklist de `r8169`.

### 2. VMs sans DNS configuré
**Symptôme** : `No DNS servers known` dans `/etc/resolv.conf` des VMs, impossibilité de télécharger les images de conteneurs.  
**Cause** : Les images cloud Debian n'ont pas de DNS configuré par défaut.  
**Solution** : Configuration manuelle de `nameserver 192.168.122.1` dans `/etc/resolv.conf`.

### 3. Image `pause:3.10` non téléchargeable
**Symptôme** : Pods bloqués en `ContainerCreating`.  
**Cause** : Problème DNS (voir point 2) empêchant la résolution de `registry.k8s.io`.  
**Solution** : Résolution du problème DNS.

### 4. VMs sans accès SSH root
**Symptôme** : `Permission denied (publickey)` lors de la connexion root.  
**Cause** : `PasswordAuthentication no` et pas de clé SSH root configurée.  
**Solution** : Injection manuelle de la clé publique via `virsh console` et activation de `PasswordAuthentication yes`.

---

## ✅ Smoke Test

Vérification du bon fonctionnement du cluster :

```bash
# Vérification des nœuds
kubectl get nodes
# NAME     STATUS   ROLES    AGE    VERSION
# node-0   Ready    <none>   1m     v1.32.3
# node-1   Ready    <none>   10s    v1.32.3

# Déploiement nginx
kubectl create deployment nginx --image=nginx
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-xxxx               1/1     Running   0          1m

# Vérification de l'API
curl --cacert ca.crt https://server.kubernetes.local:6443/version
```

---

## 📚 Références

- [Kubernetes The Hard Way — Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [realtek-r8125-dkms](https://github.com/awesometic/realtek-r8125-dkms)
- [Documentation Debian](https://www.debian.org/doc/)
- [Documentation Kubernetes](https://kubernetes.io/docs/)

---

## 👤 Auteur

Projet réalisé dans le cadre d'un apprentissage approfondi de Kubernetes et de l'administration système Linux.
