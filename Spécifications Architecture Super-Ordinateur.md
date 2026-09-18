# **CONTEXTE SYSTÈME : ARCHITECTURE SUPER-ORDINATEUR DISTRIBUÉ**

**À l'attention de :** Agent IA Autonome (Gemini) **Rôle de l'Agent :** Mainteneur, orchestrateur et superviseur de l'infrastructure. **Objectif de l'architecture :** Unifier 100 nœuds physiques (machines locales et distantes) en un cluster unique ("Super-Ordinateur") permettant le partage, le streaming et le montage d'un système de fichiers de médias distribué.

## **1\. TOPOLOGIE ET STACK TECHNIQUE (MILLE-FEUILLE RÉSEAU)**

Le système repose sur 4 couches distinctes, de l'amorçage jusqu'au système de fichiers.

### **Couche 0 : Bootstrap & Déploiement (GitOps Minimaliste)**

> * **Technologie :** GitHub \+ Bash (curl \-sL \<url\> | sh) \+ Systemd Quadlets (Podman).  
> * **Mécanique :** Un script d'installation bash gère le déploiement brut.  
> * **Directives Agent :** L'agent doit s'assurer que tout code généré pour install.sh est **strictement idempotent**. Le script doit déposer des configurations déclaratives (Quadlets .container, .network) pour systemd, plutôt que de manipuler l'état de l'OS de manière impérative.

### **Couche 1 : Maillage Physique / L2 (B.A.T.M.A.N.-adv)**

> * **Technologie :** Module noyau batman-adv, batctl.  
> * **Mécanique :** Agrégation des interfaces Wi-Fi (Ad-Hoc / 802.11s) des nœuds physiquement proches.  
> * **Interface cible :** bat0.  
> * **Directives Agent :** Gérer l'itinérance physique. Ce réseau ne doit pas sortir sur le WAN. Il sert de raccourci radio à ultra-basse latence pour la Couche 2\.

### **Couche 2 : Overlay Global / L3 (Nebula)**

> * **Technologie :** Nebula (par Slack), protocole Noise.  
> * **Mécanique :** Réseau maillé chiffré de bout en bout. Unifie les nœuds locaux (via bat0) et distants (via WAN/Internet).  
> * **Interface cible :** nebula1 (ex: subnet 10.0.0.0/16).  
> * **Routage intelligent :** Le trafic Nebula doit prioriser la découverte locale (local discovery) sur l'interface bat0. Si un nœud sort de la portée radio, Nebula bascule le trafic sur le WAN via les serveurs *Lighthouse* sans perte de paquets.  
> * **Directives Agent :** Gérer les PKI (certificats de nœuds) et maintenir la haute disponibilité des serveurs *Lighthouse*. Les applications de niveau supérieur ne doivent voir QUE les adresses IP 10.0.x.x.

### **Couche 3 : Stockage Unifié / L7 (SeaweedFS)**

> * **Technologie :** SeaweedFS (FUSE, Go).  
> * **Mécanique :** Système de fichiers distribué. Séparation stricte entre les métadonnées (Master) et les données brutes (Volume).  
> * **Topologie :**  
  * 3x Master Nodes (Quorum / Raft) pour l'annuaire.  
  * \~97x Volume Nodes pour le stockage brut.  
> * **Directives Agent :** Gérer le montage global (weed mount) sur les nœuds applicatifs (ex: Jellyfin). Surveiller le quorum des Master nodes pour éviter le *split-brain*.

## **2\. RÈGLES DE DÉCISION POUR L'AGENT IA**

Lors de la génération de code, de la résolution de bugs ou de l'évolution de cette infrastructure, l'agent doit respecter les axiomes suivants :

> 1. **Agnosticisme Matériel :** Les applications (Jellyfin, trackers, etc.) ne communiquent qu'avec l'interface Nebula (10.0.x.x) et écrivent sur le point de montage SeaweedFS. Elles ignorent l'existence de BATMAN ou du WAN.  
> 2. **Tolérance aux pannes (Partitionnement) :** Si la connexion internet tombe, les machines connectées via BATMAN (bat0) doivent continuer de communiquer via Nebula et accéder à la fraction de SeaweedFS disponible localement.  
> 3. **Légèreté absolue :** Privilégier les binaires uniques (Go/Rust) ou les conteneurs *rootless* (Podman). Interdiction d'introduire des usines à gaz de type Kubernetes ou des brokers de messages lourds pour cette stack.