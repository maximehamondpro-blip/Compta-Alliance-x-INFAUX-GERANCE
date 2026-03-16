# INFAUX Gérance N6 — Infrastructure Compta-Alliance

> **MSPR — Conception et déploiement d'une infrastructure virtualisée**  
> Groupe : INFAUX Gérance N6 | Client : Compta-Alliance | Avril 2026

---

## Sommaire

- [Contexte](#contexte)
- [Architecture](#architecture)
- [Prérequis](#prérequis)
- [Structure du dépôt](#structure-du-dépôt)
- [Déploiement pas à pas](#déploiement-pas-à-pas)
- [Plan d'adressage IP](#plan-dadressage-ip)
- [Adaptation à un autre client](#adaptation-à-un-autre-client)
- [Continuité de service](#continuité-de-service)
- [Points ouverts](#points-ouverts)

---

## Contexte

Compta-Alliance est un cabinet multi-sites (Expertise comptable, Juridique, RH) de 38 collaborateurs répartis sur 2 agences (Pessac et Paris) + 8 télétravailleurs.

**Problèmes résolus par ce projet :**

| Problème AS-IS | Solution TO-BE |
|----------------|----------------|
| Réseau à plat — tout le monde voit tout | Segmentation VLAN Corp/Invités/Management |
| Sage SaaS coûteux (~3 600€/an) | Sage On-Premise sur VM Windows Server |
| OneDrive non maîtrisé (1,8 To) | Nextcloud auto-hébergé avec ACL par profil |
| Pas d'accès distant sécurisé | VPN IPsec + OpenVPN SSL |
| Données RGPD exposées | Active Directory + LDAPS + ACL Samba |
| Hébergement web OVH ~100€/mois | VM WEB01 auto-hébergée |

---

## Architecture

### Infrastructure cible

```
                         INTERNET
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │   PESSAC    │   │  OVH (FR)   │   │    PARIS    │
    │ PFS01-PES   │◄──│  PFS01-OVH  │──►│  PFS01-PAR  │
    │ pfSense     │   │  pfSense    │   │  pfSense    │
    └─────────────┘   └─────────────┘   └─────────────┘
     IPsec AES-256         │              IPsec AES-256
                    ┌──────┴──────┐
                    │  Proxmox VE │
                    │  (bare-metal│
                    └──────┬──────┘
            ┌──────────────┼──────────────┐
            │              │              │
         DC01-OVH     SAGE01-OVH      NXT01-OVH
         (AD+LDAPS)   (Sage ERP)     (Nextcloud)
            │
         PBS01-OVH   SIP01-OVH   PRINT01-OVH
         (Backup)    (FreePBX)   (CUPS)
```

### VMs déployées

| VM | OS | vCPU | RAM | Disque | Rôle |
|----|-----|------|-----|--------|------|
| PFS01-OVH | pfSense (BSD) | 2 | 2 Go | 16 Go | Pare-feu / VPN Gateway |
| DC01-OVH | Windows Server 2022 | 2 | 4 Go | 60 Go | Active Directory + LDAPS |
| SAGE01-OVH | Windows Server 2022 | 4 | 16 Go | 200 Go | Sage Comptabilité On-Premise |
| NXT01-OVH | Debian 12 | 4 | 8 Go | 2 To | Nextcloud (fichiers) |
| SIP01-OVH | Debian 12 / FreePBX | 2 | 2 Go | 32 Go | IPBX VoIP |
| PRINT01-OVH | Debian 12 / CUPS | 2 | 2 Go | 16 Go | Impression centralisée |
| PBS01-OVH | Proxmox Backup Server | 2 | 4 Go | 500 Go | Sauvegarde VMs |
| WEB01-OVH (LXC) | Debian 12 / Nginx | 2 | 2 Go | 20 Go | Site vitrine + espace client |

---

## Prérequis

### Matériel
- Serveur dédié OVH Advance-3 (ou équivalent) : AMD EPYC, 128 Go RAM, 2×3,84 To NVMe
- OU : VPS avec minimum 8 vCPU / 16 Go RAM / 200 Go SSD pour une démo réduite

### Logiciels (tous open source / gratuits)
- [Proxmox VE](https://www.proxmox.com/en/proxmox-ve) ≥ 8.x
- [pfSense CE](https://www.pfsense.org/) ≥ 2.7
- [Proxmox Backup Server](https://www.proxmox.com/en/proxmox-backup-server) ≥ 3.x
- Windows Server 2022 (licence requise) + Sage i7

### Réseau
- 1 IP publique pour le serveur OVH (WAN PFS01-OVH)
- 1 IP publique additionnelle pour la DMZ (WEB01-OVH)
- Connexions internet FAI Orange sur les deux sites

---

## Structure du dépôt

```
INFAUX-GERANCE/
├── README.md                          ← Ce fichier
├── INFAUX GERANCE/
│   ├── 01_INFRA-SYS/
│   │   ├── system_architecture_global.png    ← Schéma Proxmox + VMs
│   │   ├── system_architecture_global.drawio ← Source draw.io
│   │   └── site.html                         ← Export interactif
│   ├── 02_INFRA-RZO/
│   │   ├── Schéma Réseau 0.0.2.png           ← Topologie réseau globale
│   │   └── Equipment URL.docx                ← URLs d'accès équipements
│   ├── 03_INFRA-BCK/
│   │   ├── backup_architecture_global_v2.png ← Architecture sauvegarde
│   │   └── backup_architecture_global_v2.drawio
│   ├── 05_RENDU_FINAL/
│   │   ├── 00_INDEX.md                       ← Index des documents
│   │   ├── 01_Presentation_Projet.md         ← Contexte AS-IS / TO-BE
│   │   ├── 02_Devis_Materiel.md              ← Budget CAPEX/OPEX
│   │   ├── 03_Architecture_Systeme.md        ← VMs, Proxmox, plan IP
│   │   ├── 04_Architecture_Reseau.md         ← VLANs, VPN, règles FW
│   │   ├── 05_Strategie_Sauvegarde.md        ← Backup 3-2-1
│   │   ├── 06_Planning_Deploiement.md        ← 4 phases, 4 semaines
│   │   └── 07_Matrice_RACI.md               ← Responsabilités projet
│   ├── Règle.md                              ← Cahier des charges client
│   └── RESSOURCE/
│       └── 2025-07-28_-_MSPR_-_Sujet_apprenant.pdf
```

---

## Déploiement pas à pas

### Étape 1 — Serveur OVH : Installation Proxmox VE

```bash
# 1. Commander le serveur OVH Advance-3 depuis le manager OVH
# 2. Booter sur l'ISO Proxmox VE depuis le KVM OVH
# 3. Installation standard Proxmox — choisir le disque NVMe OS
# 4. Configurer l'IP publique OVH sur vmbr0 (interface WAN)

# Accès interface web Proxmox après installation :
https://<IP_PUBLIQUE_OVH>:8006
```

### Étape 2 — Création des bridges réseau Proxmox

Dans l'interface Proxmox → Datacenter → Node → Network :

```
vmbr0   → Bridge WAN (IP publique OVH)
vmbr1   → Bridge interne Pessac  (aucune IP sur le bridge)
vmbr2   → Bridge interne Paris   (aucune IP sur le bridge)
vmbr3   → Bridge interne OVH     (VLANs 12/22/32)
```

### Étape 3 — Déploiement pfSense (PFS01-OVH)

```
VM ID : 100
Interfaces réseau :
  - vtnet0 → vmbr0 (WAN - IP publique OVH)
  - vtnet1 → vmbr3 tag VLAN 12  (LAN Serveurs  10.0.10.0/24)
  - vtnet2 → vmbr3 tag VLAN 22  (DMZ           10.0.20.0/24)
  - vtnet3 → vmbr3 tag VLAN 32  (MGMT/Bastion  10.0.30.0/24)
  - vtnet4 → vmbr1               (Tunnel IPsec Pessac)
  - vtnet5 → vmbr2               (Tunnel IPsec Paris)

Configuration WAN IPsec (à adapter) :
  allow UDP/500  from <IP_WAN_PESSAC> to WAN address
  allow UDP/4500 from <IP_WAN_PESSAC> to WAN address
  allow ESP       from <IP_WAN_PESSAC> to WAN address
  allow UDP/500  from <IP_WAN_PARIS>  to WAN address
  allow UDP/4500 from <IP_WAN_PARIS>  to WAN address
  allow ESP       from <IP_WAN_PARIS>  to WAN address
```

### Étape 4 — Active Directory et LDAPS (DC01-OVH)

```powershell
# 1. Installer Windows Server 2022 sur la VM DC01-OVH (IP : 10.0.10.101)
# 2. Installer le rôle AD DS
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# 3. Promouvoir en contrôleur de domaine
Install-ADDSForest -DomainName "comptaalliance.local" -InstallDns

# 4. Installer ADCS pour les certificats LDAPS
Install-WindowsFeature -Name AD-Certificate -IncludeManagementTools

# 5. Créer l'enregistrement DNS LDAPS
Add-DnsServerResourceRecordA -Name "ldaps" -ZoneName "compta-alliance.local" -IPv4Address "10.0.10.101"

# 6. Générer et déployer le certificat LDAPS
# (via la console ADCS - voir documentation 03_Architecture_Systeme.md)
```

### Étape 5 — Nextcloud + intégration LDAPS (NXT01-OVH)

```bash
# Sur NXT01-OVH (Debian 12 - IP : 10.0.10.12)

# Installer le certificat ADCS
cp /tmp/ca-comptaalliance.crt /usr/local/share/ca-certificates/
update-ca-certificates

# Configurer le plugin LDAP Nextcloud
# Admin → Paramètres → LDAP/AD :
#   Serveur : ldaps.compta-alliance.local
#   Port    : 636
#   DN base : DC=comptaalliance,DC=local
```

### Étape 6 — Tunnels VPN IPsec

Sur **PFS01-PES** (Remote GW : IP_WAN_OVH) :

| Phase 2 | Local | Remote |
|---------|-------|--------|
| 1 | 192.168.10.0/24 | 10.0.10.0/24 |
| 2 | 192.168.30.0/24 | 10.0.30.0/24 |
| 3 | 192.168.30.0/24 | 192.168.130.0/24 |
| 4 | 192.168.10.0/24 | 192.168.110.0/24 |
| 5 | 192.168.10.0/24 | 10.0.15.0/24 |

Paramètres Phase 1 : **IKEv2 / AES-256-GCM / SHA256 / DH-14 / PSK**

### Étape 7 — OpenVPN SSL (nomades)

```
Interface      : WAN
Protocol/Port  : UDP4/1195
Tunnel Network : 10.0.15.0/24
Mode           : Remote Access (User Auth)
Auth           : LDAPS → ldaps.compta-alliance.local:636
Cipher         : AES-256-GCM, AES-128-GCM, CHACHA20-POLY1305
```

### Étape 8 — Proxmox Backup Server (PBS01-OVH)

```bash
# Sur PBS01-OVH (IP : 10.0.10.15)
# Interface web : https://10.0.10.15:8007

# Configurer le datastore vm-backups
# Schedule quotidien : 02:00
# Retention : keep-daily=30, keep-weekly=4, keep-monthly=6, keep-yearly=1

# Depuis Proxmox VE, ajouter PBS comme storage :
# Datacenter → Storage → Add → Proxmox Backup Server
#   Server : 10.0.10.15
#   Datastore : vm-backups
```

---

## Plan d'adressage IP

### OVH — Datacenter France

| VLAN | Réseau | Gateway | Usage |
|------|--------|---------|-------|
| 12 — Serveurs | 10.0.10.0/24 | 10.0.10.1 (PFS01-OVH) | VMs internes |
| 22 — DMZ | 10.0.20.0/24 | 10.0.20.1 | Services exposés |
| 32 — Management | 10.0.30.0/24 | 10.0.30.1 | Bastion admin |
| OpenVPN | 10.0.15.0/24 | 10.0.15.1 | Nomades |

### Site Pessac

| VLAN | Réseau | Gateway |
|------|--------|---------|
| 10 — Corp | 192.168.10.0/24 | 192.168.10.1 (PFS01-PES) |
| 20 — Invités | 192.168.20.0/24 | 192.168.20.1 |
| 30 — Management | 192.168.30.0/24 | 192.168.30.1 |

### Site Paris

| VLAN | Réseau | Gateway |
|------|--------|---------|
| 10 — Corp | 192.168.110.0/24 | 192.168.110.1 (PFS01-PAR) |
| 20 — Invités | 192.168.120.0/24 | 192.168.120.1 |
| 30 — Management | 192.168.130.0/24 | 192.168.130.1 |

---

## Adaptation à un autre client

Pour adapter cette infrastructure à un autre client, modifier les éléments suivants :

### Variables à changer

```
DOMAINE_AD          = "comptaalliance.local"     → "votreclient.local"
PREFIXE_PESSAC      = "192.168.10"               → "10.10.10"
PREFIXE_PARIS       = "192.168.110"              → "10.10.20"
PREFIXE_OVH_LAN     = "10.0.10"                 → "172.16.10"
PREFIXE_OPENVPN     = "10.0.15"                 → "172.16.15"
NOM_BUCKET_S3       = "comptaalliance-backup"    → "votreclient-backup"
REGION_OVH          = "GRA"                     → "SBG" (selon datacenter)
```

### Parties génériques réutilisables sans modification

- Architecture Proxmox (bridges + pVLANs)
- Règles pfSense par type de VLAN (Corp / Invités / MGMT)
- Structure IPsec (phases 1 & 2 — seules les IPs et PSK changent)
- Configuration OpenVPN SSL (seul le réseau tunnel change)
- Politique de backup PBS (rotation GFS identique pour tout client)
- Structure Active Directory (seul le nom de domaine change)

### Parties spécifiques à adapter

- Licences Windows Server + Sage (selon le nombre d'utilisateurs)
- Dimensionnement des VMs (vCPU/RAM selon charge applicative)
- Plages IP (éviter les conflits avec l'existant client)
- PSK (Pre-Shared Keys) des tunnels IPsec — toujours uniques par déploiement
- Certificats ADCS (générés pour le domaine du client)

---

## Continuité de service

### Procédure de restauration rapide

```
Scénario : VM inaccessible ou corrompue

1. Connexion PBS01-OVH : https://10.0.10.15:8007
2. Datastore → vm-backups → Sélectionner la VM
3. Choisir le backup J-1 (ou J-N selon besoin)
4. Actions → Restore → Sélectionner le nœud Proxmox
5. Démarrer la VM restaurée
6. Validation fonctionnelle avec IT Client

Durée estimée : 30-60 min par VM
```

### Priorités de restauration en cas de panne totale

```
Priorité 1 (0-2h)  : DC01-OVH + PFS01-OVH → Authentification + VPN
Priorité 2 (2-4h)  : SAGE01-OVH + NXT01-OVH → Métier + Fichiers
Priorité 3 (4-8h)  : SIP01-OVH + PRINT01-OVH → Téléphonie + Impression
Priorité 4 (>8h)   : WEB01-OVH → Site vitrine
```

### Tests de restauration recommandés

| Fréquence | Test | Responsable |
|-----------|------|-------------|
| Mensuel | Restauration d'un fichier unique depuis PBS | IT Client |
| Trimestriel | Restauration VM complète (hors production) | TECH INFAUX |
| Annuel | Simulation panne totale + reprise complète | CDP + DIR |

---

## Points ouverts

| Élément | Statut | Priorité |
|---------|--------|----------|
| FreePBX — trunk SIP OVH VoIP | 🟡 En cours | Haute |
| MFA sur VPN (TOTP/App) | 🔵 Planifié | Moyenne |
| Monitoring Prometheus + Grafana | 🔵 Planifié | Moyenne |
| Cluster Proxmox HA (2ème nœud) | 🔵 Roadmap v2 | Basse |
| DHCP Relay inter-sites | ⛔ Annulé | — |

---

## Résumé financier

| Type | Montant HT |
|------|-----------|
| **CAPEX (investissement initial)** | **7 919 €** |
| **OPEX (mensuel récurrent)** | **132,49 €/mois** |
| **Économie annuelle estimée** | **~4 100 €/an** |
| **Retour sur investissement** | **~2,3 ans** |

---

## Contact

**INFAUX Gérance N6** — Solutions IT sur mesure  
Projet MSPR — Avril 2026

---

*Dossier de rendu — Version 1.0 — CONFIDENTIEL*
