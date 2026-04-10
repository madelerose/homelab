<div align="center">

# 🖥️ Hybrid Cloud Homelab

**Infrastructure personnelle hybride — GCP + Local — sous Rocky Linux 10**

[![Rocky Linux](https://img.shields.io/badge/OS-Rocky%20Linux%2010-10B981?style=for-the-badge&logo=rockylinux&logoColor=white)](https://rockylinux.org/)
[![Docker](https://img.shields.io/badge/Runtime-Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![Tailscale](https://img.shields.io/badge/Network-Tailscale-242424?style=for-the-badge&logo=tailscale&logoColor=white)](https://tailscale.com)
[![Traefik](https://img.shields.io/badge/Proxy-Traefik%20v3-24A1C1?style=for-the-badge&logo=traefikproxy&logoColor=white)](https://traefik.io)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)](https://terraform.io)

</div>

---

## 🧭 Vision

> Parc hybride composé d'un nœud **Cloud (GCP)** servant de point d'entrée public et de deux machines **locales (Homelab)** fournissant la puissance de calcul, le stockage et les services critiques.

Le flux de trafic est le suivant :

```
Utilisateur ➔ Cloudflare (DNS/WAF) ➔ GCP net-svc-01 (Traefik) ➔ Tailscale Mesh ➔ Services (srv-01 / srv-02)
```

---

## 📊 Inventaire du Parc

| Nœud | Rôle | Localisation | CPU | RAM | Stockage |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **net-svc-01** | Edge Proxy / Bastion | ☁️ GCP (Free Tier) | 2 vCPU (e2-micro) | 1 Go | 20 Go SSD |
| **srv-01** | Compute / NAS | 🏠 Homelab (Local) | AMD Ryzen 5 2500U (8 vCPU) | 8 Go | 120 Go SSD + 1 To HDD |
| **srv-02** | DNS / Backup | 🏠 Homelab (Local) | Intel i5-4210U (4 vCPU) | 6 Go | 700 Go HDD |

---

## 🏗️ Architecture

### 🌐 net-svc-01 — « La Vigie » (GCP)
- **Traefik v3** : Reverse Proxy — intercepte le trafic Cloudflare et le route via Tailscale
- **Glance** : Dashboard de monitoring léger
- **Uptime Kuma** : Surveillance externe des nœuds locaux
- ⚠️ RAM limitée à 1 Go — Swap de 2 Go provisionné obligatoirement

### 🚀 srv-01 — Compute / NAS (Local)
- **Stack Monitoring** : Grafana + Prometheus (métriques sur disque 1 To)
- **Immich** : Gestion de photos auto-hébergée
- **Plex** : Serveur media
- **Docker Workers** : Conteneurs gourmands en CPU

### 🛡️ srv-02 — DNS / Backup (Local)
- **Pi-hole** : DNS + bloqueur de publicités réseau *(en cours)*
- **Vaultwarden** : Gestionnaire de mots de passe auto-hébergé *(en cours)*
- **Backup** : Réplication des données critiques depuis srv-01 *(en cours)*

---

## 🔐 Sécurité & Réseau

| Composant | Rôle |
| :--- | :--- |
| **Tailscale Mesh** | Tunnel Zero Trust entre GCP et le homelab — aucun port ouvert sur la box |
| **MagicDNS** | Résolution stable par FQDN (`srv-01.tail*.ts.net`) — plus d'IPs hardcodées |
| **SSH ED25519** | Authentification exclusivement par clé sur l'ensemble du parc |
| **SELinux** | Activé et configuré (contexte `ssh_home_t`) |
| **Cloudflare** | DNS + WAF en frontal |
| **Let's Encrypt** | Certificats TLS via DNS Challenge Cloudflare |

---

## 🛠️ Stack Technique

| Catégorie | Outils |
| :--- | :--- |
| **OS** | Rocky Linux 10 (Bare Metal & GCP) |
| **Conteneurs** | Docker, Docker Compose |
| **Réseau** | Tailscale, Traefik v3, Cloudflare |
| **IaC** | Terraform (provisioning GCP), Ansible (hardening & MCO) |
| **Monitoring** | Grafana, Prometheus, Uptime Kuma, Glance |
| **Scripting** | Bash |

---

## 📁 Structure du Dépôt

```
homelab/
├── gcp/
│   ├── terraform/              # Provisioning net-svc-01 (GCP Free Tier)
│   └── traefik/
│       ├── traefik.yml         # Config statique Traefik
│       ├── dynamic_conf.yml    # Routing vers les services (via MagicDNS)
│       ├── docker-compose.yml
│       └── .env.example        # Template — ne jamais commiter le vrai .env
├── homelab/
│   ├── ansible/                # Playbooks de hardening & MCO
│   └── docker/                 # Stacks Docker des services locaux
└── docs/
    ├── 00-vue-ensemble.md
    ├── 11-guide-mise-en-prod.md
    ├── 12-provisioning-gcp.md
    ├── 13-provisioning-homelab.md
    └── 14-traefik-reverse-proxy.md
```

---

## 🚀 Mise en Production

### Prérequis
- Compte GCP avec Free Tier activé
- Domaine géré par Cloudflare
- Machines locales sous Rocky Linux 10

### 1. Provisionner net-svc-01 (GCP)

```bash
gcloud compute instances create net-svc-01 \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --image-family=rocky-linux-10 \
    --image-project=rocky-linux-cloud \
    --boot-disk-size=20GB \
    --boot-disk-type=pd-standard \
    --tags=http-server,https-server
```

### 2. Appliquer le Guide de Mise en Prod

Sur chaque nœud (net-svc-01, srv-01, srv-02) :

```bash
# Tailscale
sudo dnf config-manager --add-repo https://pkgs.tailscale.com/stable/rhel/9/tailscale.repo
sudo dnf install -y tailscale && sudo systemctl enable --now tailscaled
sudo tailscale up

# Docker
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker && sudo usermod -aG docker sysops
```

### 3. Déployer Traefik

```bash
# Swap obligatoire sur e2-micro (1 Go RAM)
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Initialisation acme.json
touch acme.json && chmod 600 acme.json

# Lancement
docker compose up -d --force-recreate
```

---

## ⚠️ Retours d'Expérience (Post-Mortem)

Erreurs réelles rencontrées en production, documentées pour éviter toute régression.

| # | Symptôme | Cause | Leçon |
| :--- | :--- | :--- | :--- |
| 1 | VM inaccessible SSH, services 502 | OOM sans Swap sur e2-micro | Toujours provisionner **2 Go de Swap** avant Docker |
| 2 | Routage cassé après redémarrage | IPs Tailscale hardcodées | Utiliser exclusivement le **MagicDNS** |
| 3 | Swap repassé à 0 après reboot | Ligne `/etc/fstab` manquante | Vérifier `free -h` après chaque redémarrage |
| 4 | Erreur 502 persistante | Inversion srv-01 / srv-02 dans la config | Valider avec `curl -I` depuis le bastion |
| 5 | MagicDNS non résolu dans Traefik | Mode réseau `bridge` Docker | Passer en **`network_mode: host`** |
| 6 | "directory not found" au démarrage | Fichiers de config inexistants → Docker crée des dossiers vides | S'assurer que tous les fichiers montés **existent avant** `docker compose up` |

---

## ✅ État du Parc

- [x] **net-svc-01** — Tailscale ✓ · Docker ✓ · SSH Key ✓ · Traefik ✓
- [x] **srv-01** — Tailscale ✓ · Docker ✓ · SSH Key ✓
- [x] **srv-02** — Tailscale ✓ · Docker ✓ · SSH Key ✓

---

## 📬 Contact

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-madelerose-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/madelerose)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Mathieu-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/mathieu-adele-rose)

</div>

---

<div align="center">

*Infrastructure is code. Code is documentation. Documentation is this repo.* 🚀

</div>
