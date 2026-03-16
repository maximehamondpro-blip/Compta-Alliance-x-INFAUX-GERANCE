
# Phase 1 — Recherche & Conception

**INFAUX Gérance N6 × Compta-Alliance | Avril 2026 | CONFIDENTIEL**

---

## Sommaire

1. [Comprendre la Virtualisation](#1-comprendre-la-virtualisation)
2. [Choix de l'Hyperviseur](#2-choix-de-lhyperviseur)
3. [Conception de l'Architecture Cible](#3-conception-de-larchitecture-cible)
4. [Limites Identifiées et Compromis Techniques](#4-limites-identifiées-et-compromis-techniques)
5. [Continuité de Service — Scénarios d'Incident](#5-continuité-de-service--scénarios-dincident)
6. [Conclusion — Retour Critique](#6-conclusion--retour-critique)

---

## 1. Comprendre la Virtualisation

### 1.1 Qu'est-ce que la virtualisation ?

La virtualisation est une technologie qui permet de faire fonctionner plusieurs systèmes d'exploitation ou environnements logiciels de manière isolée sur un seul et même matériel physique. Concrètement, au lieu d'avoir un serveur physique dédié par service (un pour la messagerie, un pour la comptabilité, un pour les fichiers…), la virtualisation permet de créer des **machines virtuelles** (VM) qui partagent les ressources d'un seul serveur tout en restant totalement indépendantes les unes des autres.

Le composant central qui rend cela possible s'appelle un **hyperviseur** : c'est le logiciel qui gère l'allocation des ressources (CPU, RAM, stockage, réseau) entre les différentes machines virtuelles. L'hyperviseur joue le rôle d'arbitre — il s'assure que chaque VM dispose de ce dont elle a besoin sans empiéter sur les autres.

---

### 1.2 Quels problèmes la virtualisation résout-elle en entreprise ?

#### Avant la virtualisation — le problème du « one app = one server »

Dans une infrastructure traditionnelle, chaque application critique (messagerie, ERP, serveur de fichiers…) tourne sur sa propre machine physique dédiée. Cette approche génère plusieurs problèmes majeurs :

- **Coût élevé** : multiplication des serveurs physiques, chacun sous-utilisé (souvent à 10-15 % de ses capacités)
- **Gaspillage énergétique** : des serveurs qui consomment de l'électricité même quand ils ne font rien
- **Complexité de maintenance** : intervention physique nécessaire pour chaque panne
- **Rigidité** : pour ajouter un service, il faut acheter un nouveau serveur, attendre la livraison, l'installer…
- **Manque de résilience** : si le serveur physique tombe en panne, le service est entièrement perdu

#### Après la virtualisation — les bénéfices concrets

| Problème résolu | Comment la virtualisation y répond |
|---|---|
| Sous-utilisation des serveurs | Plusieurs VMs partagent un seul serveur — taux d'utilisation typique : 60-80% |
| Coût d'infrastructure | 1 serveur puissant remplace 5 à 10 serveurs physiques séparés |
| Sauvegarde et restauration | Snapshots instantanés — restauration en quelques minutes vs plusieurs heures |
| Déploiement lent | Une nouvelle VM se crée en quelques minutes depuis un modèle (template) |
| Panne physique fatale | Migration rapide vers une autre machine physique — continuité de service |
| Administration dispersée | Un seul outil (Proxmox VE) pour gérer toute l'infrastructure |
| Tests risqués en production | Environnement de test isolé sur la même machine — zéro impact sur la prod |

---

### 1.3 Virtualisation, Émulation, Conteneurisation — Quelles différences ?

#### La Virtualisation

La virtualisation reproduit un ordinateur complet — processeur, mémoire, disque, carte réseau — de manière logicielle. Chaque VM embarque son propre système d'exploitation (Windows, Linux…) complet et fonctionne de façon totalement autonome. L'hyperviseur traduit les instructions de la VM vers le matériel physique.

> 💡 **Analogie** : c'est comme diviser un grand appartement en plusieurs studios indépendants — chacun a sa propre entrée, ses propres compteurs, son propre espace de vie.

- **Avantages** : isolation totale, compatibilité maximale, sécurité renforcée
- **Inconvénients** : consommation de ressources importante (chaque VM embarque un OS complet)

#### L'Émulation

L'émulation va plus loin : elle simule non seulement le logiciel mais aussi le matériel physique en entier — y compris le processeur. Cela permet de faire tourner des logiciels conçus pour une architecture matérielle différente (ex : faire tourner un jeu GameBoy sur un PC x86).

> ⚠️ **Analogie** : c'est comme apprendre une langue étrangère pour traduire chaque mot à la volée — ça fonctionne, mais c'est beaucoup plus lent.

- **Avantages** : compatibilité totale — fait tourner n'importe quel OS sur n'importe quel matériel
- **Inconvénients** : très lent (10 à 100x plus lent que le natif), usage réservé à des fins de tests ou de rétrocompatibilité, jamais utilisé en production

#### La Conteneurisation

La conteneurisation (Docker, Kubernetes) adopte une approche radicalement différente : les conteneurs ne virtualisent pas un ordinateur complet. Ils partagent le noyau du système d'exploitation hôte et isolent uniquement les processus applicatifs. Chaque conteneur embarque uniquement l'application et ses dépendances, pas un OS entier.

> 💡 **Analogie** : c'est comme des boxes de co-working dans un open-space — chacun a son espace de travail délimité, mais tout le monde partage la même infrastructure du bâtiment (électricité, internet, climatisation).

- **Avantages** : ultra-léger (démarrage en millisecondes), densité maximale, idéal pour les applications modernes
- **Inconvénients** : isolation moins forte que les VMs, pas adapté aux applications legacy (Windows Server, Sage)

#### Tableau comparatif synthétique

| Critère | Virtualisation | Émulation | Conteneurisation |
|---|---|---|---|
| Unité isolée | VM complète (OS + App) | Système entier + matériel | Application + dépendances |
| OS embarqué | Oui — OS complet par VM | Oui — OS + architecture simulés | Non — partage le kernel hôte |
| Performance | Bonne (5-10% overhead) | Très mauvaise (10-100x) | Excellente (<1% overhead) |
| Isolation | Forte | Très forte | Moyenne |
| Démarrage | Minutes (boot OS) | Minutes (boot OS simulé) | Secondes (pas de boot OS) |
| Usage typique | Serveurs, infra d'entreprise | Rétrocompatibilité, tests | Applications web, microservices |
| Notre projet | ✅ Solution retenue | ✗ Non pertinent | Hors périmètre MSPR |

> → **Pourquoi la virtualisation pour Compta-Alliance ?** La conteneurisation nécessite des compétences avancées et n'est pas adaptée aux applications métier legacy (Sage). La virtualisation offre le meilleur équilibre entre isolation, compatibilité et facilité d'administration pour une PME.

---

## 2. Choix de l'Hyperviseur

### 2.1 Contexte et contraintes

La contrainte principale de la MSPR est claire : le projet doit pouvoir être déployé et démontré sur un poste de travail personnel. Nous disposons cependant d'un VPS OVH dédié, ce qui ouvre la possibilité d'utiliser un hyperviseur de type 1 (bare-metal).

| Type | Définition | Exemples | Contexte d'usage |
|---|---|---|---|
| Type 1 (bare-metal) | S'installe directement sur le matériel, remplace l'OS | Proxmox VE, ESXi, XCP-ng | Serveurs de production, datacenter |
| Type 2 (hosted) | S'installe comme un logiciel sur un OS existant | VirtualBox, VMware Workstation | Poste de développement, tests, formation |

---

### 2.2 Comparatif des solutions

#### VirtualBox (Oracle)

| Critère | Note | Détail |
|---|---|---|
| Compatibilité OS hôte | ⭐⭐⭐⭐⭐ | Windows, macOS, Linux — meilleure compatibilité multi-plateforme |
| Compatibilité OS invité | ⭐⭐⭐⭐⭐ | Windows, toutes distributions Linux, BSD, Solaris |
| Performances | ⭐⭐⭐ | Overhead notable (~10-20%), VT-x/AMD-V supporté |
| Facilité d'administration | ⭐⭐⭐⭐ | Interface graphique intuitive + CLI VBoxManage |
| Réseau virtualisé | ⭐⭐⭐⭐ | NAT, Bridge, Host-Only, Internal Network — 4 modes |
| Snapshots | ⭐⭐⭐⭐ | Snapshots chaînés, restauration facile |
| Coût | ⭐⭐⭐⭐⭐ | 100% gratuit et open source (GPL) |
| Support professionnel | ⭐⭐ | Support communautaire principalement |
| **Limites** | — | Performance USB 3.0 limitée, accélération 3D partielle |

#### VMware Workstation Pro / Fusion (Broadcom)

| Critère | Note | Détail |
|---|---|---|
| Compatibilité OS hôte | ⭐⭐⭐⭐ | Windows et Linux (Workstation), macOS (Fusion) |
| Compatibilité OS invité | ⭐⭐⭐⭐⭐ | Windows, Linux, BSD — excellent support |
| Performances | ⭐⭐⭐⭐⭐ | Overhead minimal (~5%), optimisations poussées |
| Facilité d'administration | ⭐⭐⭐⭐⭐ | Interface soignée, intégration vSphere pour les pros |
| Réseau virtualisé | ⭐⭐⭐⭐⭐ | VMnet Editor — configuration avancée des segments réseau |
| Snapshots | ⭐⭐⭐⭐⭐ | Snapshots rapides, gestion en arbre, auto-snapshot |
| Coût | ⭐⭐ | Gratuit depuis 2024 (usage personnel) — payant en entreprise |
| Support professionnel | ⭐⭐⭐⭐⭐ | Broadcom — support enterprise de qualité |
| **Limites** | — | Changement de politique de licence en 2024, incertitude long terme |

#### Proxmox VE — Référence professionnelle (utilisé sur VPS OVH)

| Critère | Note | Détail |
|---|---|---|
| Type | — | Hyperviseur de **TYPE 1** (bare-metal) — nécessite matériel dédié |
| Compatibilité OS invité | ⭐⭐⭐⭐⭐ | KVM pour VMs complètes + LXC pour conteneurs Linux |
| Performances | ⭐⭐⭐⭐⭐ | Overhead quasi nul (<2%) — performances quasi-natives |
| Interface d'administration | ⭐⭐⭐⭐⭐ | Interface web puissante accessible depuis n'importe où |
| Réseau virtualisé | ⭐⭐⭐⭐⭐ | Linux bridges, VLANs, SDN — niveau datacenter |
| Snapshots & Backup | ⭐⭐⭐⭐⭐ | PBS intégré, snapshots ZFS, réplication distante |
| Coût | ⭐⭐⭐⭐ | Gratuit (sans support) — abonnement support optionnel |
| Cas d'usage pro | ⭐⭐⭐⭐⭐ | Référence PME/ETI — concurrence directe VMware ESXi |
| **Limites sur poste perso** | — | Ne s'installe pas sur un OS existant — matériel dédié obligatoire |

---

### 2.3 Tableau de décision

| Critère | Poids | VirtualBox | VMware Workstation | Proxmox VE |
|---|---|---|---|---|
| Fonctionnement sur PC perso | 30% | ✅ Oui | ✅ Oui | ❌ Impossible |
| Gratuité totale | 20% | ✅ Oui | ⚠️ Partiel | ✅ Oui |
| Performances | 20% | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Réseau virtualisé | 15% | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Disponible sur VPS OVH | 15% | ❌ Non | ❌ Non | ✅ Oui |

---

### 2.4 Choix argumenté

> → **Décision : Proxmox VE sur le VPS OVH dédié** (hyperviseur type 1) — VirtualBox sur poste de travail pour démonstration locale si nécessaire.

Proxmox VE a été retenu comme hyperviseur principal pour les raisons suivantes :

- Notre groupe dispose d'un VPS OVH dédié — la contrainte MSPR est donc respectée
- Proxmox VE est la **référence open source du marché PME** — utiliser la même solution qu'en production réelle renforce la valeur pédagogique et professionnelle du projet
- L'interface web de Proxmox permet une administration complète **depuis n'importe quel poste**, sans installation locale
- Les performances bare-metal sont indispensables pour héberger simultanément Sage (Windows Server), Active Directory, Nextcloud et pfSense sur le même serveur
- La gestion native des VLANs et bridges réseau dans Proxmox colle parfaitement aux besoins de segmentation de Compta-Alliance

VirtualBox reste pertinent pour les démonstrations locales et constitue une alternative valide pour les groupes ne disposant pas de VPS.

---

## 3. Conception de l'Architecture Cible

### 3.1 Traduction des besoins métiers en exigences techniques

Le client ne formule pas de besoins techniques — il exprime des **besoins métiers** que nous avons traduits :

| Besoin métier exprimé | Exigence technique retenue |
|---|---|
| « Tout le monde voit tous les dossiers — c'est dangereux » | Active Directory + ACL Samba — contrôle d'accès par profil métier |
| « Sage coûte trop cher en SaaS » | Migration Sage On-Premise sur VM Windows Server |
| « On ne peut pas imprimer depuis Paris » | Serveur d'impression CUPS centralisé + VPN |
| « Le site web OVH coûte 100€/mois » | VM WEB01 auto-hébergée — suppression de l'abonnement |
| « Les télétravailleurs n'ont pas accès aux outils » | VPN SSL OpenVPN — accès nomade depuis n'importe où |
| « Si un serveur tombe, on perd tout » | Stratégie backup 3-2-1 — PBS + NAS + S3 |
| « On ne peut pas accueillir des clients sur notre Wi-Fi » | VLAN 20 Invités — isolation totale du réseau entreprise |

---

### 3.2 Machines virtuelles — Rôles et dimensionnement

| VM | Rôle | OS | vCPU | RAM | Disque | Justification |
|---|---|---|---|---|---|---|
| PFS01-OVH | Pare-feu / VPN Gateway | pfSense (BSD) | 2 | 2 Go | 16 Go | Sécurité périmétrique — point d'entrée unique |
| DC01-OVH | Active Directory + LDAPS | Windows Server 2022 | 2 | 4 Go | 60 Go | Authentification centralisée — base de tout le SI |
| SAGE01-OVH | ERP Sage On-Premise | Windows Server 2022 | 4 | 16 Go | 200 Go | Cœur de métier — 10 utilisateurs simultanés via RDP |
| NXT01-OVH | Nextcloud (fichiers) | Debian 12 | 4 | 8 Go | 2 To | Remplace OneDrive — 1,8 To de données à migrer |
| SIP01-OVH | IPBX VoIP (FreePBX) | Debian 12 | 2 | 2 Go | 32 Go | 30 téléphones — centralisation de la téléphonie |
| PRINT01-OVH | Impression centralisée | Debian 12 / CUPS | 2 | 2 Go | 16 Go | Impression Paris ↔ Pessac via VPN |
| PBS01-OVH | Proxmox Backup Server | PBS OS | 2 | 4 Go | 500 Go | Sauvegarde quotidienne de toutes les VMs |
| WEB01-OVH (LXC) | Site vitrine + espace client | Debian 12 / Nginx | 2 | 2 Go | 20 Go | Supprime l'abonnement OVH à 100€/mois |

#### Bilan des ressources

| Ressource | Total alloué | Capacité serveur | Marge disponible |
|---|---|---|---|
| vCPU | 20 vCPUs | 12 cœurs / 24 threads (EPYC) | 4 threads de réserve |
| RAM | 40 Go | 128 Go DDR5 | 88 Go de réserve |
| Stockage VMs | ~2,9 To | 2× 3,84 To NVMe RAID1 = ~3,5 To utiles | ~600 Go de réserve |

---

### 3.3 Hypothèses retenues

**Hypothèse 1 — Trois VLANs par site (Corp / Invités / Management)**

La séparation Corp/Invités est une exigence de sécurité non négociable (données RGPD exposées à tous dans l'AS-IS). Le VLAN Management permet d'administrer les équipements réseau sans risque d'interférence avec le trafic utilisateur.

**Hypothèse 2 — Pare-feu pfSense par nœud (OVH + Pessac + Paris)**

Une instance pfSense par site garantit un filtrage local indépendant. En cas de panne du tunnel VPN, chaque site conserve un accès internet filtré. pfSense est open source, mature, et couvre tous nos besoins (IPsec, OpenVPN, VLAN, DHCP).

**Hypothèse 3 — VPN IPsec Site-à-Site AES-256-GCM**

IKEv2 avec AES-256-GCM est le standard actuel de chiffrement pour les VPNs d'entreprise. L'algorithme GCM (Galois/Counter Mode) offre à la fois chiffrement et intégrité en un seul passage — plus performant que AES-CBC + SHA.

**Hypothèse 4 — OpenVPN SSL pour les nomades (port UDP/1195)**

OpenVPN est plus simple à déployer pour les utilisateurs finaux (client multiplateforme). L'authentification via LDAPS garantit que les comptes désactivés dans l'AD perdent automatiquement l'accès VPN.

---

## 4. Limites Identifiées et Compromis Techniques

### 4.1 Limites liées à l'environnement

| Limite identifiée | Impact | Contexte MSPR | Production réelle |
|---|---|---|---|
| VPS unique — pas de redondance physique | Élevé | Un seul serveur OVH — SPOF matériel | Deux serveurs en cluster Proxmox HA |
| Bande passante internet variable | Moyen | VPN IPsec dépend de la connexion OVH | Liens dédiés ou MPLS entre sites |
| Adresses IP publiques dynamiques sur sites | Moyen | Les IPs WAN des pfSense peuvent changer | IP fixes dédiées par site (FAI Pro) |
| Pas de redondance alimentation/réseau | Élevé | Un seul câble réseau, une seule alimentation | Double alimentation, double uplink |
| Stockage RAID logiciel Proxmox | Moyen | Moins performant qu'un contrôleur RAID matériel | Contrôleur RAID HW + disques SAS |

---

### 4.2 Compromis techniques réalisés

**Compromis 1 — pfSense au lieu d'OPNsense**

Nous avons retenu pfSense (FreeBSD) plutôt qu'OPNsense pour sa documentation plus abondante et sa communauté plus large, facilitant la résolution de problèmes en contexte de MSPR. En production, OPNsense serait préférable pour son cycle de mises à jour plus fréquent et son interface plus moderne.

**Compromis 2 — LDAPS sans MFA**

L'authentification Multi-Facteurs (MFA/2FA) n'a pas été implémentée dans cette version. En contexte MSPR, LDAPS + certificats ADCS offre un niveau de sécurité satisfaisant. En production réelle, un MFA (Microsoft Authenticator, TOTP) serait indispensable sur tous les accès VPN et RDP.

**Compromis 3 — FreePBX non finalisé (SIP en cours)**

La configuration complète de FreePBX (trunk SIP OVH, provisioning téléphones Yealink) n'a pas été achevée dans les délais du projet. Le serveur est déployé et fonctionnel pour les appels internes, mais le raccordement au réseau téléphonique public reste à finaliser.

**Compromis 4 — Adressage VLAN sans IPAM formalisé**

Nous avons adopté les bonnes pratiques d'adressage (plages distinctes par site) mais sans outil IPAM. En production, un outil comme NetBox ou phpIPAM serait utilisé pour la gestion centralisée des adresses.

**Compromis 5 — Haute disponibilité absente**

Notre infrastructure repose sur un serveur unique. En cas de panne matérielle, tous les services sont interrompus. Ce risque est atténué par la stratégie de backup (RTO < 4h), mais pas éliminé.

---

### 4.3 Ce que nous ferions différemment en contexte professionnel réel

| Élément | Version MSPR (actuelle) | Version production réelle |
|---|---|---|
| Hyperviseur HA | 1 serveur Proxmox standalone | Cluster Proxmox HA 3 nœuds — migration live des VMs |
| Authentification | LDAPS + certificats ADCS | MFA (TOTP/App) sur VPN + accès RDP + portail web |
| Monitoring | Alertes email PBS | Prometheus + Grafana + alertes PagerDuty |
| DNS externe | Géré manuellement | Infra DNS redondante (2 résolveurs séparés) |
| Firewall | pfSense standalone | pfSense HA en CARP (2 instances actif/passif) |
| Téléphonie | FreePBX sans redondance | FreePBX + trunk SIP redondant + QoS réseau |
| Sauvegarde | PBS + NAS + S3 | Idem + tests de restauration trimestriels formalisés |
| Gestion des identités | AD On-Premise | Azure AD Hybrid ou Entra ID pour les nomades |
| Documentation | Manuel (Word + Markdown) | Wiki.js + GitOps — doc as code |
| IPAM | Implicite dans les docs | NetBox ou phpIPAM intégré |

---

## 5. Continuité de Service — Scénarios d'Incident

### 5.1 Stratégie globale

| Pilier | Mécanisme | Objectif |
|---|---|---|
| Prévention | Segmentation VLAN — pare-feux — ACL strictes | Réduire la surface d'attaque et les erreurs de configuration |
| Détection | Alertes PBS — logs pfSense — rapports backup email | Identifier rapidement toute anomalie ou échec de backup |
| Restauration | Snapshots Proxmox — PBS — NAS — S3 | Revenir à un état stable en moins de 4 heures (RTO cible) |

#### Indicateurs cibles

| Indicateur | Valeur | Justification |
|---|---|---|
| RPO (Recovery Point Objective) | 24 heures | Backups quotidiens à 02h00 — perte max = 1 journée de données |
| RTO (Recovery Time Objective) | 4 heures | Restauration VM depuis PBS estimée à 30-60 min par VM |
| Fréquence backups | Quotidien | Rotation GFS : 30j / 4 semaines / 6 mois / 1 an |
| Copies de sauvegarde | 3 (règle 3-2-1) | PBS OVH + NAS Pessac + Cloud S3 Gravelines |

---

### 5.2 Scénario 1 — Suppression accidentelle de fichiers

> 📋 **Contexte** : Un comptable supprime accidentellement le dossier partagé « Dossiers_Clients_2025 » sur Nextcloud (NXT01-OVH). Prise de conscience 2 heures après.

| Étape | Action | Acteur | Durée |
|---|---|---|---|
| 1 — Détection | L'utilisateur constate la disparition du dossier dans Nextcloud | Utilisateur | ~2h après la suppression |
| 2 — Alerte | Appel au référent IT client | Utilisateur → IT Client | 5 min |
| 3 — Évaluation | IT Client contacte INFAUX Gérance N6 — ticket d'incident ouvert | IT Client → TECH | 10 min |
| 4 — Analyse | TECH vérifie si la suppression est dans la corbeille Nextcloud (rétention 30 jours) | TECH | 5 min |
| 5a — Restauration simple | Si oui : restauration depuis la corbeille Nextcloud (admin panel) | TECH | 5 min |
| 5b — Restauration PBS | Si non : connexion à PBS01-OVH, sélection backup J-1 de NXT01-OVH, restauration VM ou fichiers | TECH | 30-60 min |
| 6 — Validation | IT Client valide la restauration des données avec l'utilisateur | IT Client + Utilisateur | 15 min |
| 7 — Post-incident | Rédaction fiche incident, rappel procédure corbeille | TECH + CDP | 30 min |

**Mesures préventives associées :**
- Sensibilisation utilisateurs : ne jamais supprimer directement — utiliser la corbeille
- ACL Nextcloud renforcées : seuls les admins peuvent vider définitivement la corbeille
- Rétention corbeille configurée à 30 jours sur NXT01-OVH
- Snapshot Proxmox de NXT01-OVH conservé 7 jours supplémentaires

---

### 5.3 Scénario 2 — Panne complète du serveur OVH

> 🔴 **Contexte** : Le serveur physique OVH tombe en panne matérielle à 09h00 un lundi matin. Tous les services sont interrompus : Sage, Nextcloud, VPN, AD. Les 38 collaborateurs ne peuvent plus travailler.

| Étape | Action | Acteur | Durée |
|---|---|---|---|
| 1 — Détection | Monitoring PBS + alertes pfSense — détection automatique dans les 5 min | Système + IT Client | 5 min |
| 2 — Diagnostic | Connexion au manager OVH — vérification état serveur — ouverture ticket support si hardware | TECH | 15 min |
| 3 — Décision | Si réparation OVH > 4h : activation plan de reprise — commande nouveau VPS OVH temporaire | CDP + DIR | 30 min |
| 4 — Nouveau serveur | Déploiement Proxmox VE sur le nouveau VPS OVH | TECH | 45 min |
| 5 — Priorité 1 | Restauration DC01-OVH depuis PBS (ou NAS01-PES si PBS inaccessible) — AD opérationnel | TECH | 45 min |
| 6 — Priorité 1 | Restauration PFS01-OVH — remise en route VPN IPsec et OpenVPN | TECH | 30 min |
| 7 — Priorité 2 | Restauration SAGE01-OVH — accès Sage rétabli | TECH | 45 min |
| 8 — Priorité 2 | Restauration NXT01-OVH — accès fichiers rétabli | TECH | 60 min |
| 9 — Validation | Tests fonctionnels complets — validation avec IT Client | TECH + IT Client | 30 min |
| 10 — Communication | Information collaborateurs — estimation retour à la normale | CDP + DIR | 15 min |

> ✅ **Durée totale estimée : 4h30** — conforme à l'objectif RTO de 4 heures. Les sauvegardes NAS01-PES permettent la restauration même si PBS01-OVH est inaccessible.

**Point de défaillance unique (SPOF) identifié :**

Le serveur OVH constitue un SPOF dans notre architecture. C'est le compromis principal de notre solution « budget PME ». La mitigation repose sur la rapidité de restauration plutôt que sur la redondance matérielle. En contexte professionnel réel, un cluster Proxmox HA avec migration live des VMs éliminerait ce risque.

---

### 5.4 Scénario 3 — Compromission par ransomware

> ⚠️ **Contexte** : Un collaborateur de Pessac clique sur une pièce jointe malveillante. Son poste Windows 10 BYOD est infecté par un ransomware qui tente de chiffrer les partages réseau.

**Mécanismes de défense en place :**
- **VLAN Corp 10 isolé** : le ransomware ne peut pas atteindre les VLANs Invités ou Management
- **ACL Samba strictes** : chaque utilisateur n'accède qu'à ses propres dossiers — dommages limités à la victime
- **Snapshots Proxmox** de NXT01-OVH : restauration de l'état propre de la veille
- **Règles pfSense VLAN 10** : les connexions vers les plages RFC1918 non autorisées sont bloquées

| Étape | Action | Acteur |
|---|---|---|
| 1 — Isolation immédiate | IT Client déconnecte physiquement le poste infecté du switch PoE | IT Client |
| 2 — Évaluation périmètre | TECH analyse les logs pfSense — quels partages ont été accédés ? | TECH |
| 3 — Snapshot préventif | Snapshot immédiat de NXT01-OVH avant toute restauration | TECH |
| 4 — Restauration | Si fichiers Nextcloud chiffrés : restauration depuis snapshot Proxmox J-1 | TECH |
| 5 — Nettoyage poste | Formatage complet du poste infecté — réinstallation propre | IT Client + Utilisateur |
| 6 — Audit | Analyse des logs — identification du vecteur d'infection — sensibilisation équipe | CDP + TECH |

---

### 5.5 Politique d'usage des Snapshots

> ⚠️ **Important** : les snapshots Proxmox ne remplacent **pas** les backups PBS. Un snapshot est local au serveur — si le serveur physique tombe en panne, les snapshots sont perdus. Les backups PBS sont la vraie protection contre la perte de données.

| Moment | Action | Justification |
|---|---|---|
| Avant toute mise à jour OS/applicative | Snapshot manuel nommé (ex : `avant-update-sage-v7.2`) | Pouvoir revenir à l'état stable si la MAJ casse quelque chose |
| Avant toute modification de configuration critique | Snapshot manuel (pfSense, AD, Nextcloud) | Restauration rapide en cas d'erreur de configuration |
| Quotidien automatique (02h00) | Backup PBS complet + vérification intégrité | Protection contre la perte de données — conservation 30 jours |
| Avant intervention maintenance planifiée | Snapshot + notification équipe | Traçabilité des interventions + filet de sécurité |

---

## 6. Conclusion — Retour Critique

### 6.1 Ce que ce projet nous a appris

- La virtualisation n'est pas une fin en soi — c'est un outil au service d'un besoin métier. Chaque choix technique (Proxmox, pfSense, VLAN, OpenVPN) découle directement d'un problème exprimé par le client.
- La sécurité s'architecte en couches : pare-feu périmétrique + segmentation VLAN + ACL applicatives + authentification centralisée. Aucune couche seule n'est suffisante.
- La continuité de service est une discipline à part entière — elle se planifie, se documente et se teste. **Un backup non testé est un backup qui n'existe pas.**
- La dette technique est réelle : les compromis acceptés (pas de HA, pas de MFA, SIP non finalisé) sont documentés et constituent une roadmap d'amélioration pour la version 2.0.

### 6.2 Perspectives d'amélioration

| Horizon | Action |
|---|---|
| Court terme | Finalisation FreePBX + trunk SIP OVH VoIP |
| Moyen terme | Ajout MFA sur VPN et RDP + monitoring Prometheus/Grafana |
| Long terme | Migration vers cluster Proxmox HA (2 nœuds) pour éliminer le SPOF |
| Évolutivité | L'architecture est prête pour un 3ème site : nouvelle instance pfSense + VLANs + tunnel IPsec en moins d'une journée |

---

*Document de recherche — Phase 1 — INFAUX Gérance N6 — Version 1.0 — Avril 2026*
