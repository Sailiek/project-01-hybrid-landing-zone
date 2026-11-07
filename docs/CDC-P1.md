

# Cahier des charges — Projet 1

## Hybrid Landing Zone automatisée — Proxmox / Terraform / Ansible / OPNsense

**Version : 1.0 — Architecture cible**

---

# 1. Vision globale du portfolio

Le portfolio complet sera constitué de trois projets indépendants mais interconnectables.

```text
┌──────────────────────────────────────────────────────────────┐
│                  PORTFOLIO CLOUD / DEVSECOPS                 │
└──────────────────────────────────────────────────────────────┘

       PROJECT 1                         PROJECT 2
┌─────────────────────┐           ┌──────────────────────┐
│ Hybrid Landing Zone │           │ DevSecOps / GitOps   │
│                     │           │                      │
│ Proxmox             │           │ Git                  │
│ Terraform           │──────────►│ CI/CD                │
│ Ansible             │           │ Security scanning    │
│ OPNsense            │           │ GitOps               │
│ Network segmentation│           │ Containers/K8s       │
└──────────┬──────────┘           └──────────┬───────────┘
           │                                  │
           │                                  │
           └────────────────┬─────────────────┘
                            ▼
                 ┌──────────────────────┐
                 │ PROJECT 3            │
                 │ Detection-as-Code    │
                 │ & SOAR               │
                 │                      │
                 │ Logs                 │
                 │ Detection rules      │
                 │ SIEM                 │
                 │ Alerting             │
                 │ Response automation  │
                 └──────────────────────┘
```

Le **Projet 1 est le socle infrastructurel**.

Il doit donc être conçu avant tout pour fournir :

* réseau ;
* segmentation ;
* machines ;
* contrôle d'accès ;
* automatisation ;
* observabilité minimale ;
* une base propre pour les projets suivants.

---

# 2. Objectif du Projet 1

L'objectif n'est pas simplement :

> « créer des VM avec Terraform ».

L'objectif est de construire une **mini landing zone hybride/on-premise automatisée**, représentant certains principes que l'on retrouverait dans Azure/AWS :

* séparation des environnements ;
* segmentation réseau ;
* principe du moindre privilège ;
* réseau de management ;
* DMZ ;
* réseau applicatif ;
* infrastructure déclarative ;
* configuration automatisée ;
* reproductibilité ;
* documentation des décisions d'architecture.

Le projet devra être suffisamment propre pour pouvoir dire en entretien :

> « J'ai conçu et automatisé une petite landing zone sécurisée sur Proxmox, avec une séparation réseau DMZ / administration / applications, Terraform pour le provisioning et Ansible pour la configuration. »

---

# 3. Contraintes matérielles

## PVE-01 — serveur physique

Ton premier nœud est un **HP ProBook 430 G5**.

```text
CPU
Intel Core i5-8250U
4 cores / 8 threads
1 socket

RAM
8 GB

SSD
256 GB commercial
≈238.5 GiB disponibles

Proxmox
Debian GNU/Linux 13
Proxmox VE

IP actuelle
192.168.137.10/24
```

Stockage :

```text
/dev/sda
≈238.5 GiB

PVE LVM
├── root       ≈69 GB
├── swap       ≈7.6 GB
└── data       ≈141 GB
```

Il reste actuellement environ :

```text
16 GB libres dans le VG
```

et le thin pool est très peu utilisé.

**Conclusion : stockage suffisant pour le projet.**

La contrainte principale est la RAM.

---

# 4. PVE-02 — second nœud virtuel

Ton PC Windows est :

```text
HP ZBOOK 14UG6
Windows 11
32 GB RAM
512 GB SSD
≈192 GB disponibles
VMware Workstation 17 Pro
17.6.2
```

Nous allons créer :

```text
Windows
   │
   └── VMware Workstation
          │
          └── PVE-02
```

Dimension initiale :

```text
PVE-02
4 vCPU
8 GB RAM
≈80 GB disque virtuel
```

Ce sera du **nested virtualization**.

Il ne faut pas présenter cela comme un cluster de production.

Dans le portfolio, on pourra explicitement expliquer :

> PVE-02 is a nested virtualization node used to emulate a multi-node infrastructure environment under constrained lab resources.

C'est une limitation assumée, pas un défaut caché.

---

# 5. Pourquoi deux nodes ?

Le deuxième nœud permet d'aller plus loin que :

```text
Proxmox
└── 5 VM
```

On pourra montrer :

```text
                    Infrastructure
                          │
              ┌───────────┴───────────┐
              │                       │
           PVE-01                  PVE-02
          physique                virtualisé
              │                       │
          security                 workloads
          services                 applicatifs
```

Cela ouvre également la porte à l'étude de :

* clustering ;
* quorum ;
* migration ;
* HA ;
* contraintes d'un cluster 2-node ;
* stockage local vs partagé ;
* architecture distribuée.

Mais **nous ne dépendrons pas d'un cluster Proxmox pour la V1**.

---

# 6. Pourquoi pas Ceph ?

Important.

Nous n'allons pas faire :

```text
PVE-01 ── Ceph ── PVE-02
```

juste pour avoir un joli schéma.

Ceph nécessite normalement davantage de nœuds, de disques et de ressources.

Avec :

```text
8 GB + 8 GB
```

ce serait artificiel et contre-productif.

Dans ton README, on pourra même avoir un ADR :

> **ADR — Why Ceph is not used**

C'est une excellente question d'architecture à savoir expliquer en entretien.

---

# 7. Architecture réseau globale

Le réseau physique actuel est :

```text
                     INTERNET
                         │
                         ▼
                  Windows / Home NAT
                   192.168.137.1
                         │
                         │ Ethernet
                         ▼
                   PVE-01
                192.168.137.10
```

Tu n'as pas de switch manageable.

Nous allons donc créer la segmentation **virtuellement**.

---

# 8. Réseau PVE-01

Actuellement :

```text
vmbr0
bridge-ports nic0
192.168.137.10/24
gateway 192.168.137.1
```

C'est notre réseau externe / underlay.

Puis :

```text
vmbr1
bridge-ports none
```

est un réseau interne virtuel.

Nous allons exploiter cette architecture.

---

# 9. Architecture OPNsense

Actuellement :

```text
OPNsense
│
├── net0 → vmbr0
└── net1 → vmbr1
```

OPNsense possède :

```text
LAN = 10.0.0.1/24
WAN = actuellement non renseigné
```

Nous allons faire évoluer cette VM en **routeur/firewall central du lab**.

Architecture :

```text
                       HOME NETWORK
                      192.168.137.0/24
                             │
                           vmbr0
                             │
                         OPNsense
                     ┌───────┴───────┐
                     │               │
                    WAN             LAN/TRUNK
                                     │
                                   vmbr1
```

---

# 10. Segmentation logique

Sur le LAN interne :

```text
                   OPNsense
                       │
                  802.1Q trunk
                       │
                     vmbr1
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      VLAN 10        VLAN 20        VLAN 30
        │              │              │
       DMZ            ADMIN           APP
```

Et :

```text
VLAN 99
Management
```

---

# 11. Plan d'adressage

## VLAN 10 — DMZ

```text
Network: 10.10.10.0/24
Gateway: 10.10.10.1
```

Rôle :

* reverse proxy ;
* web server ;
* services éventuellement exposés.

---

## VLAN 20 — ADMIN

```text
Network: 10.10.20.0/24
Gateway: 10.10.20.1
```

Rôle :

* bastion ;
* Ansible ;
* administration ;
* éventuellement monitoring.

---

## VLAN 30 — APP

```text
Network: 10.10.30.0/24
Gateway: 10.10.30.1
```

Rôle :

* application ;
* backend ;
* base de données.

---

## VLAN 99 — MANAGEMENT

```text
Network: 10.10.99.0/24
Gateway: 10.10.99.1
```

Rôle :

* management des workloads ;
* éventuellement interfaces d'administration internes.

---

# 12. Attention : management Proxmox

Je ne veux pas mettre immédiatement le management physique de Proxmox derrière OPNsense.

Actuellement :

```text
PVE-01
192.168.137.10
```

reste accessible via le réseau hôte.

C'est notre **break-glass / underlay management path**.

Donc :

```text
192.168.137.0/24
       │
       └── PVE-01
```

et :

```text
10.10.99.0/24
       │
       └── management des workloads
```

Cette distinction est volontaire.

---

# 13. Architecture des workloads

## PVE-01

Au départ :

```text
PVE-01
│
├── OPNsense
│
└── Bastion
```

Puis éventuellement :

```text
└── Automation / Ansible
```

mais nous pourrons garder Terraform/Ansible directement sur le host au début pour économiser la RAM.

---

# 14. PVE-02

PVE-02 sera orienté workloads :

```text
PVE-02
│
├── Web
├── Application
└── Database
```

Puis éventuellement :

```text
├── Monitoring
└── autres services
```

---

# 15. Flux réseau cible

C'est ici que le projet commence réellement à devenir un projet **Cloud + Security**.

Politique initiale :

| Source   | Destination | Politique                    |
| -------- | ----------- | ---------------------------- |
| Internet | DMZ         | HTTP/HTTPS autorisés         |
| Internet | ADMIN       | DENY                         |
| Internet | APP         | DENY                         |
| Internet | MANAGEMENT  | DENY                         |
| ADMIN    | DMZ         | SSH/HTTPS selon besoin       |
| ADMIN    | APP         | management autorisé          |
| DMZ      | APP         | uniquement ports applicatifs |
| DMZ      | ADMIN       | DENY                         |
| APP      | DB          | port DB uniquement           |
| APP      | ADMIN       | DENY par défaut              |
| ADMIN    | MANAGEMENT  | autorisé                     |
| DMZ      | MANAGEMENT  | DENY                         |
| APP      | MANAGEMENT  | DENY                         |

Le principe général sera :

> **Deny by default, explicit allow.**

---

# 16. Terraform

Terraform sera responsable de l'**infrastructure**.

Il devra gérer notamment :

```text
Proxmox
│
├── VM
├── CPU
├── RAM
├── disks
├── network interfaces
├── cloud-init
└── éventuellement configuration réseau
```

Le provider choisi sera :

**BPG Proxmox Provider**

plutôt que de partir sur l'ancien provider Telmate.

---

# 17. Ansible

Ansible sera responsable de la **configuration**.

Exemples :

```text
Ansible
│
├── users
├── SSH hardening
├── packages
├── firewall host
├── NTP
├── logging
├── Docker/Podman éventuellement
├── services
└── configuration applicative
```

Donc :

```text
Terraform
    │
    │ "Cette machine doit exister"
    ▼
Proxmox
    │
    │
    ▼
VM Debian
    │
    │ "Cette machine doit être configurée ainsi"
    ▼
Ansible
```

C'est une distinction que tu dois être capable d'expliquer parfaitement en entretien.

---

# 18. Cloud-init

Nous utiliserons probablement **Cloud-init** entre Terraform et Ansible.

Pipeline :

```text
Terraform
   │
   ▼
Clone Debian template
   │
   ▼
Cloud-init
   │
   ├── hostname
   ├── IP
   ├── SSH key
   └── initial user
   │
   ▼
VM prête
   │
   ▼
Ansible
```

Cela permet d'éviter de configurer manuellement chaque VM.

---

# 19. Structure GitHub

Le dépôt final devrait ressembler à :

```text
project-01-hybrid-landing-zone/
│
├── README.md
│
├── docs/
│   │
│   ├── architecture.md
│   ├── network.md
│   ├── security.md
│   ├── deployment.md
│   │
│   └── decisions/
│       ├── ADR-001-network-segmentation.md
│       ├── ADR-002-proxmox-topology.md
│       ├── ADR-003-no-ceph.md
│       └── ADR-004-terraform-ansible-separation.md
│
├── terraform/
│   │
│   ├── versions.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── locals.tf
│   ├── main.tf
│   ├── outputs.tf
│   │
│   ├── modules/
│   │   ├── network/
│   │   ├── vm/
│   │   └── ...
│   │
│   └── environments/
│       └── lab/
│
├── ansible/
│   │
│   ├── ansible.cfg
│   ├── inventory/
│   ├── playbooks/
│   └── roles/
│       ├── common/
│       ├── hardening/
│       └── ...
│
├── diagrams/
│
└── .gitignore
```

---

# 20. Phases de réalisation

Voilà le véritable **plan de travail**.

---

## PHASE 0 — Architecture & documentation

### Objectif

Figer l'architecture avant de coder.

### Travail

Documenter :

* architecture physique ;
* architecture logique ;
* plan IP ;
* VLAN ;
* flux ;
* responsabilités Terraform/Ansible/OPNsense ;
* contraintes matérielles.

### Livrables

```text
docs/architecture.md
docs/network.md
docs/security.md
ADR-001...
```

### Concepts à étudier

Cherche notamment :

* Landing Zone
* Underlay / Overlay
* VLAN
* 802.1Q
* Network segmentation
* DMZ
* East-West / North-South traffic
* Zero Trust
* Defense in Depth

---

# PHASE 1 — Git & Terraform bootstrap

### Objectif

Obtenir :

```text
terraform init
terraform validate
terraform plan
```

sans erreur.

### Travail

Créer :

```text
Git repository
        │
        ▼
Terraform
        │
        ▼
Proxmox API
```

Créer un utilisateur/token Proxmox dédié.

Pas de `root`.

### Concepts

* Terraform provider
* API
* authentication
* provider
* resource
* data source
* variable
* output
* state
* idempotence

---

# PHASE 2 — Réseau

### Objectif

Construire le réseau logique.

```text
vmbr1
 │
 └── VLAN
      ├── 10 DMZ
      ├── 20 ADMIN
      ├── 30 APP
      └── 99 MGMT
```

Configurer OPNsense.

### Livrable

Une machine de test doit pouvoir être attachée à un VLAN et communiquer correctement.

### Concepts

* VLAN tagging
* trunk
* access VLAN
* routing
* inter-VLAN routing
* gateway
* DHCP
* DNS
* NAT

---

# PHASE 3 — Firewall

### Objectif

Appliquer la segmentation.

Exemple :

```text
DMZ ──X──> ADMIN
DMZ ──►──> APP:443
ADMIN ──►──> DMZ:22
ADMIN ──►──> APP:22
APP ──►──> DB:5432
```

### Livrable

Matrice de flux documentée + règles OPNsense.

### Concepts

* stateful firewall
* ACL
* least privilege
* default deny
* NAT
* ingress / egress
* east-west traffic
* north-south traffic

---

# PHASE 4 — Compute provisioning

### Objectif

Terraformer les premières VM.

```text
Terraform
    │
    ├── clone Debian
    ├── CPU
    ├── RAM
    ├── disk
    ├── network
    └── cloud-init
```

Premier objectif :

```text
Debian VM
```

Puis :

```text
Bastion
Web
App
DB
```

progressivement.

---

# PHASE 5 — Ansible

### Objectif

Automatiser le système.

Pipeline :

```text
Terraform
    ↓
VM
    ↓
Cloud-init
    ↓
Ansible
    ↓
Hardening
```

Exemples :

* SSH keys ;
* désactivation login root ;
* SSH hardening ;
* utilisateurs ;
* sudo ;
* packages ;
* NTP ;
* audit ;
* firewall local.

---

# PHASE 6 — PVE-02

### Objectif

Introduire le deuxième nœud.

```text
PVE-01
│
├── OPNsense
└── Bastion

PVE-02
│
├── Web
├── App
└── DB
```

À ce stade, nous étudierons :

* réseau entre nodes ;
* routage ;
* nested virtualization ;
* stockage local ;
* limites du design.

---

# PHASE 7 — Clustering expérimental

**Optionnelle.**

Seulement une fois la V1 fonctionnelle.

Nous pourrons créer un cluster :

```text
PVE-01
   │
   ├── Corosync
   │
   └────── PVE-02
```

et étudier :

* quorum ;
* vote ;
* HA ;
* migration ;
* fencing ;
* split-brain.

Mais le cluster ne sera **pas une dépendance du projet**.

Cela peut devenir une section :

> Advanced / Experimental Architecture

---

# PHASE 8 — Observabilité

Avant de passer au Projet 2, on pourra ajouter :

```text
Logs
Metrics
Health checks
```

Mais sans déployer une énorme stack.

On pourra éventuellement intégrer quelque chose de léger.

L'objectif est surtout de préparer :

```text
Project 1
     │
     └── logs/events
             │
             ▼
         Project 3
```

---

# 21. Points de vigilance — Projet 1

## 1. RAM

C'est notre contrainte principale.

Ne pas transformer le lab en :

```text
OPNsense
Kubernetes
GitLab
Jenkins
Wazuh
Elastic
Prometheus
Grafana
Postgres
...
```

simultanément.

---

## 2. Nested virtualization

PVE-02 est une simulation de deuxième serveur.

Il faudra surveiller :

* CPU virtualization ;
* performance ;
* réseau VMware ;
* bridge/trunk ;
* MAC spoofing/promiscuous mode selon le design.

---

## 3. Réseau Wi-Fi / NAT

Ton réseau physique est :

```text
Windows
  │
Wi-Fi
  │
192.168.137.1
```

Ce n'est **pas** un véritable réseau d'entreprise.

Donc il faut distinguer :

```text
Physical underlay
        ≠
Logical lab network
```

Le lab simule le second.

---

## 4. Secrets

Jamais :

```text
API token
password
SSH private key
```

dans Git.

---

## 5. Terraform State

Le state peut contenir des informations sensibles.

Donc :

```text
terraform.tfstate
```

ne doit pas être publié.

À terme, on pourra réfléchir à un backend distant/chiffré, mais **ne complexifions pas V1 inutilement**.

---

## 6. Ne pas mélanger Terraform et Ansible

Mauvais :

```text
Terraform
 └── exécute 50 scripts shell
      └── configure Linux
```

Bon :

```text
Terraform → infrastructure

Ansible → configuration
```

---

# 22. Points de vigilance pour le Projet 2

Le Projet 2 devra être construit **au-dessus** du Projet 1.

On pourra avoir :

```text
Git
 │
 ▼
CI
 │
 ├── SAST
 ├── SCA
 ├── secret scanning
 ├── IaC scanning
 └── container scanning
 │
 ▼
Build
 │
 ▼
Registry
 │
 ▼
Deployment
 │
 ▼
Infrastructure Project 1
```

Il faudra donc dès maintenant prévoir :

* naming cohérent ;
* IP management propre ;
* SSH ;
* DNS ;
* certificats éventuellement ;
* séparation management/workloads.

Et surtout :

**ne pas faire du Projet 1 un cluster Kubernetes déguisé.**

Kubernetes sera beaucoup plus intéressant dans le Projet 2.

---

# 23. Points de vigilance pour le Projet 3

Le Projet 3 devra exploiter les événements générés par le Projet 1 et 2.

Par exemple :

```text
OPNsense
   │
   ├── firewall logs
   └── VPN/auth logs
          │
          ▼
       Detection

Linux
   │
   ├── SSH logs
   ├── auth logs
   └── audit
          │
          ▼
       Detection
```

Puis :

```text
Detection
    │
    ▼
Alert
    │
    ▼
Enrichment
    │
    ▼
SOAR
    │
    ▼
Response
```

Donc dès le Projet 1, il faudra éviter de construire une architecture où les logs sont impossibles à récupérer.

---

# 24. Les concepts que je veux que tu maîtrises réellement

Tu n'as pas besoin de connaître tout ça aujourd'hui.

Mais le projet doit te servir à les apprendre.

### Infrastructure

* Proxmox
* hypervisor
* VM
* template
* cloud-init
* Terraform
* Ansible
* IaC
* idempotence

### Networking

* subnet
* CIDR
* VLAN
* 802.1Q
* trunk
* bridge
* routing
* NAT
* firewall
* DMZ
* DNS
* DHCP

### Architecture

* landing zone
* management plane
* data plane
* control plane
* segmentation
* least privilege
* defense in depth
* zero trust
* north-south
* east-west

### Security

* bastion
* SSH hardening
* network ACL
* host firewall
* secrets management
* PKI
* TLS
* logging
* audit

### DevOps

* Git
* CI/CD
* immutable infrastructure
* configuration management
* GitOps
* containers
* Kubernetes

### Cloud

Il faudra progressivement faire le mapping :

```text
Proxmox             Azure/AWS
──────────────────────────────────
VLAN/network        VNet/VPC
Subnet              Subnet
Firewall            NSG / Security Group
VM                  EC2 / Azure VM
Terraform           Terraform
Ansible             Ansible
Landing Zone        Landing Zone
Management network  Management subnet
Bastion             Azure Bastion / bastion host
```

**C'est particulièrement intéressant pour tes candidatures Cloud**, parce que tu pourras dire :

> « J'ai appris les concepts de landing zone et de segmentation sur une infrastructure on-premise automatisée, puis je les ai mappés aux primitives Azure/AWS. »

---

# 25. Ce que le recruteur doit voir dans le GitHub

Je ne veux pas que le dépôt soit uniquement :

```text
main.tf
variables.tf
outputs.tf
```

Je veux qu'il raconte une **démarche d'architecte**.

Le README devra éventuellement avoir :

```text
# Hybrid Landing Zone

## Architecture

## Business / Technical Goals

## Constraints

## Architecture Decision Records

## Network Architecture

## Security Model

## Infrastructure as Code

## Configuration Management

## Deployment

## Validation

## Failure Scenarios

## Lessons Learned

## Cloud Mapping
```

Et surtout :

### Architecture Decision Records

Exemple :

> **ADR-003 — Why no Ceph?**

Contexte :

```text
2 nodes
8 GB RAM/node
local SSD
homelab
```

Décision :

```text
No Ceph in V1.
```

Conséquence :

```text
Local storage
No distributed storage
No HA storage
```

C'est **beaucoup plus intéressant intellectuellement** qu'un README qui prétend avoir une architecture « enterprise-grade ».

---

# 26. Les scénarios de validation

Le projet ne sera pas terminé lorsque :

```bash
terraform apply
```

retourne `Apply complete`.

Il faudra tester :

### Test 1

```text
ADMIN → APP
```

autorisé.

### Test 2

```text
DMZ → ADMIN
```

bloqué.

### Test 3

```text
APP → DB
```

port approprié uniquement.

### Test 4

```text
Internet → APP
```

bloqué.

### Test 5

Détruire une VM :

```text
terraform destroy
```

puis :

```text
terraform apply
```

et obtenir la même infrastructure.

### Test 6

Ansible doit être idempotent :

```text
ansible-playbook ...
```

première exécution :

```text
changed
```

deuxième :

```text
changed = 0
```

Ça, c'est du **vrai IaC**.

---

# 27. Architecture finale du Projet 1

En résumé :

```text
                              INTERNET
                                  │
                         Home Network/NAT
                         192.168.137.1
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                 PVE-01                      Windows
             192.168.137.10                     │
                    │                       VMware
                    │                         PVE-02
                    │                           │
              ┌─────▼──────┐                    │
              │  OPNsense   │                    │
              │             │                    │
              │ FW / NAT    │                    │
              │ DNS / DHCP  │                    │
              └──────┬──────┘                    │
                     │                            │
                  vmbr1                           │
                     │                            │
                VLAN trunk                         │
                     │                            │
       ┌─────────────┼─────────────┐              │
       │             │             │              │
      DMZ           ADMIN          APP            │
 10.10.10.0/24 10.10.20.0/24 10.10.30.0/24       │
       │             │             │              │
      Web          Bastion         App            │
                                    │             │
                                    ▼             │
                                    DB ◄───────────┘

              MANAGEMENT
              10.10.99.0/24

                    │
             Infrastructure
                    │
        ┌───────────┴───────────┐
        │                       │
    Terraform                Ansible
        │                       │
        └──────────┬────────────┘
                   │
                Proxmox
```

---

# 28. La philosophie du projet

Il y a une chose que je veux conserver tout au long de sa réalisation.

**Nous ne cherchons pas à reproduire une entreprise entière avec 8 Go de RAM.**

Nous cherchons à reproduire **les principes architecturaux d'une entreprise**, avec les ressources disponibles.

Donc :

```text
Entreprise réelle
        │
        ▼
Principes
        │
        ▼
Architecture simplifiée
        │
        ▼
Homelab
```

et non :

```text
Homelab
   │
   ▼
On ajoute 40 VM
   │
   ▼
"Enterprise architecture"
```

La première approche est beaucoup plus défendable devant un recruteur senior.

---

## 29. Roadmap globale figée

Je te propose donc de suivre exactement cet ordre :

```text
┌────────────────────────────────────┐
│ 0. Architecture & ADR              │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 1. Git + Terraform + Proxmox API  │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 2. OPNsense + VLAN + Routing      │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 3. Firewall + Network Security     │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 4. Terraform VM + Cloud-init       │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 5. Ansible + Hardening             │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 6. PVE-02 + Distributed Workloads │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 7. Validation / Failure scenarios  │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 8. Documentation + Cloud mapping   │
└────────────────┬───────────────────┘
                 ▼
          PROJECT 1 COMPLETE
                 │
        ┌────────┴────────┐
        ▼                 ▼
   PROJECT 2          PROJECT 3
   DevSecOps          Detection/SOAR
```

**Et surtout : on ne passe pas directement à la phase 1.** Pour ton objectif de montée en compétence, la phase 0 doit d'abord te permettre de comprendre les notions que tu vas implémenter. Tu peux donc utiliser ce document comme **cahier des charges + syllabus de recherche** : à chaque notion que tu rencontres (`VLAN trunk`, `landing zone`, `underlay`, `cloud-init`, `state`, `quorum`, etc.), tu peux la travailler avant de l'implémenter.

La prochaine étape logique sera donc de transformer ce cahier des charges en **architecture documentaire du dépôt** — README, diagrammes, ADR-001, plan IP et matrice des flux — **sans encore écrire le Terraform**. C'est la meilleure façon de construire le projet comme un vrai travail d'architecte plutôt qu'un exercice de syntaxe Terraform.
