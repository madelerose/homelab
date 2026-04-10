# homelab
[Projet Personnel] Gestion de mon infrastructure hybride (GCP &amp; Local) via Ansible, Docker et Traefik
# 🏠 Hybrid Cloud Homelab Infrastructure

> Une infrastructure hybride résiliente liant des nœuds locaux (Homelab) et un bastion Cloud (GCP) via un maillage VPN sécurisé.

---

## 📊 Architecture & Inventaire

Mon parc est composé de trois nœuds sous **Rocky Linux 10**, orchestrés pour la haute disponibilité et le monitoring.

| Nœud | Localisation | CPU | RAM | Rôle Principal |
| :--- | :--- | :--- | :--- | :--- |
| **net-svc-01** | Cloud (GCP) | 2 vCPU (E2-micro) | 1 Go | Vigie / Traefik Reverse Proxy |
| **srv-01** | Local | Ryzen 5 (8 vCPU) | 8 Go | Compute / NAS / Immich |
| **srv-02** | Local | Intel i5 (4 vCPU) | 6 Go | DNS (Pi-hole) / Backup |

---

## 🛠️ Stack Technologique & Sécurité

### Réseau & Connectivité
- **VPN Mesh :** Utilisation de **Tailscale** pour interconnecter le cloud et le local sans ouverture de ports.
- **Routage :** **Traefik v3** gère le trafic entrant avec challenge DNS Cloudflare pour le SSL.

### Hardening & Provisioning
- **Sécurité SSH :** Authentification exclusive par **clés ED25519**.
- **OS Hardening :** Configuration de **SELinux** et durcissement du démon SSH.
- **Laptop-as-Server :** Gestion spécifique du *Lid Switch* pour transformer des ordinateurs portables en nœuds stables.

---

## 🚀 Services Déployés

- **Proxying :** Traefik v3 (avec Dashboard sécurisé).
- **Monitoring :** Stack Uptime Kuma et Glance pour la surveillance du parc.
- **Sécurité réseau :** Pi-hole pour le DNS menteur local.
- **Gestionnaire de mots de passe :** Vaultwarden auto-hébergé.

---

## ⚠️ Notes Techniques & Optimisations (Post-Mortem)

Pour garantir la stabilité sur une instance GCP `e2-micro` (1 Go RAM) :
- **Gestion de la mémoire :** Mise en place d'un **Swap de 2 Go** pour éviter les plantages lors des challenges ACME de Traefik.
- **Docker Networking :** Utilisation du `network_mode: host` pour Traefik afin de permettre la résolution MagicDNS via Tailscale.

---

## 📂 Organisation du dépôt

- `/ansible` : Configuration automatisée des nœuds.
- `/docker` : Fichiers `docker-compose.yml` et configurations des services.
- `/docs` : [Guides détaillés de mise en prod](./docs) et fiches d'infrastructure.
