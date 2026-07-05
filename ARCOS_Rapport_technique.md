# ARCOS — Récapitulatif complet du projet
**Architecture Réseau et Cybersécurité Open Source** - Projet OSSI<br>
Groupe : Tim NGUYEN--MENU, Adrien RIVET, Safiane RAS EL QDIM, Tess POIRAT, Gabriel PRIEUR

---

## Note

Ceci est le rapport de solution technique, pas le rapport. Il explique ce qui a été mené techniquement en précisement dans la réalisation de notre projet.
<br>**Toutes les captures d'écran de réalisation de chaque étape sont jointes à part mais regroupées par étapes.**
<br>L'IA générative a été partiellement utilisée dans la mise en page de ce rapport, évidment les réalisations techniques ont été faites manuellement.

---

## Contexte

Une PME fictive souhaite mettre en place une solution de sécurité réseau basée exclusivement sur des logiciels open source, afin de réduire ses coûts tout en répondant à ses besoins de sécurité et de gestion.

---

## Étape 0 — Scénario d'entreprise

**Entreprise fictive : ARCOS Solutions** — PME de services numériques, ~50 salariés, 2 sites :

### Site Principal (Siège)
- Rôle : administration, héberge le VPN et la PKI
- IP WAN : **fixe** (`203.0.113.1`)
- Poste derrière : **1 poste administrateur**
- pfSense héberge : pare-feu, IPSec, OpenVPN, PKI, IDS/IPS Suricata, ntopng

### Site Isolé (Agence)
- Rôle : utilisateurs, accès Internet filtré
- IP WAN : **dynamique** (`203.0.113.2` en labo)
- Postes derrière : **plusieurs postes Internet**
- pfSense héberge : pare-feu, IPSec, DHCP+NAT, Squid+SquidGuard, portail captif

---

## Étape 1 — Architecture et plan d'adressage

### Choix des outils (justifiés)

| Besoin | Outil retenu | Justification |
|---|---|---|
| Pare-feu / routeur | **pfSense CE** | Open source, tout-en-un, intègre tous les services |
| VPN site-à-site | **IPSec** | Standard, natif dans pfSense |
| VPN accès distant | **OpenVPN** | S'appuie sur la PKI (certificats X.509), livrable du sujet |
| IDS/IPS | **Suricata** | Multi-thread, plus moderne que Snort, bien intégré pfSense |
| Analyse trafic | **ntopng** | Paquet pfSense, visualisation temps réel |
| Tests performance | **iPerf3** | Standard de mesure de débit réseau |
| Filtrage web | **Squid + SquidGuard** | Proxy cache + filtrage d'URL par catégories |
| Portail captif | **pfSense Captive Portal** | Intégré nativement dans pfSense |

Choix d'exclusion :
> WireGuard NON retenu : plus simple mais contourne la PKI → moins intéressant.  
> Snort NON retenu : mono-thread, moins performant que Suricata.

### Topologie — 4 machines virtuelles

Nous avons opté pour 4 VM, 2 postes et 2 VM regroupant pfSense + routeur :

```
                    ┌──────────────────────────────────┐
                    │   Internet — WAN simulé          │
                    │   203.0.113.0/24                 │
                    │   (LAN Segment : wan-link)       │
                    └──────────┬──────────────┬────────┘
                               │              │
                    ┌──────────▼───┐      ┌───▼──────────────┐
                    │ pfSense      │      │ pfSense          │
                    │ Principal    │◄────►│ Isolé            │
                    │ VM1          │IPSec │ VM2              │
                    │ WAN: .113.1  │      │ WAN: .113.2      │
                    │ LAN: 10.10   │      │ LAN: 10.20       │
                    └──────┬───────┘      └────────┬────────┘
                           │                       │
                    ┌──────▼───────┐      ┌────────▼─────────┐
                    │ Poste Admin  │      │ Postes Internet  │
                    │ VM3          │      │ VM4              │
                    │ 10.10.10.x   │      │ 10.20.10.x       │
                    └──────────────┘      └──────────────────┘
```

### Plan d'adressage IP complet

| VM | Rôle | Interface WAN | Interface LAN |
|---|---|---|---|
| VM1 — pfSense Principal | Routeur + pare-feu + VPN + PKI | `203.0.113.1/24` (fixe) | `10.10.10.1/24` |
| VM2 — pfSense Isolé | Routeur + pare-feu + Squid + captif | `203.0.113.2/24` (dyn) | `10.20.10.1/24` |
| VM3 — Poste Admin | Client site principal | — | `10.10.10.x` (DHCP) |
| VM4 — Postes Internet | Clients site isolé | — | `10.20.10.x` (DHCP) |
| (transverse) | Clients VPN distants | — | `10.30.0.x` (pool OpenVPN) |

### Réseaux virtuels VMware (LAN Segments)

| Nom | Rôle |
|---|---|
| `wan-link` | Internet simulé entre les deux pfSense |
| `lan-principal` | LAN du site principal |
| `lan-isole` | LAN du site isolé |

---

## Étape 2 — Installation et configuration des VM

### VM1 — pfSense Principal

#### Configuration VMware (Settings)

| Paramètre | Valeur |
|---|---|
| RAM | **2.8 Go** (minimum 2 Go) |
| CPU | 2 |
| Disque | 20 Go (SCSI) |
| CD/DVD | ISO pfSense CE 2.7.x+ AMD64 |
| Network Adapter 1 | **NAT** → `em0` = WAN |
| Network Adapter 2 | **LAN Segment `wan-link`** → `em1` = futur OPT1 |
| Network Adapter 3 | **LAN Segment `lan-principal`** → `em2` = LAN |

> Toutes les cartes sont cochées **« Connected »** et **« Connect at power on »**

#### ISO utilisé
- **pfSense CE** (Community Edition) — téléchargée sur `pfsense.org`
- Décompression dy `.iso.gz` → `.iso` avant de la monter

#### Déroulement de l'installation

1. Démarrage de la VM → l'installeur pfSense CE se lance
2. **Active Subscription Validation** → **[ Install CE ]**
3. **WAN Interface** → `em0` → **[ OK ]**
4. **LAN Interface** → `em2` → **[ OK ]**
5. **Installation Options** :
   - **F** → choisir **UFS** (ZFS (par défaut) — cause l'erreur `mountroot`)
   - Partition Scheme : GPT 
   - **>> Continue → [ OK ]**
6. **Disk Selection** → appuyer sur **Espace** sur `da0` → **[ OK ]**
7. **Confirmation** → **[ Yes ]**
8. Installation (progress [182/182])
9. Écran **Complete** → **[ Reboot ]**
10. Avant le reboot : *Settings → CD/DVD* → **on a décoché « Connect at power on »**

#### Problèmes rencontrés et solutions

| Erreur | Cause | Solution |
|---|---|---|
| `mountroot>` au démarrage | ZFS mal supporté sous VMware | Réinstaller en choisissant **UFS** |
| `Child process Killed` | RAM insuffisante | Augmenter à **2 Go minimum** |
| `Failed to fetch repository data` | Mauvaise ISO pfSense | Télécharger la bonne **ISO pfSense CE** |
| `em1` absente dans l'installeur | Normal — seulement WAN+LAN configurés à l'install | On a ajouté em1 en OPT1 depuis l'interface web plus tard |

#### Résultat après installation 

```
pfSense 2.8.1-RELEASE (amd64)

WAN (wan) → em0 → 192.168.244.x/24   (NAT VMware, DHCP, accès Internet)
LAN (lan) → em2 → 10.10.10.1/24      (lan-principal, DHCP 10.10.10.100-199)
```

---

### VM3 — Poste Admin (Ubuntu)

#### Configuration VMware

| Paramètre | Valeur |
|---|---|
| OS | Ubuntu Desktop (dernière LTS) |
| RAM | 2 Go |
| Disque | 20 Go |
| Network Adapter | **LAN Segment `lan-principal`** uniquement |

#### Installation Ubuntu
- Écran "Connect to internet" → **« Do not connect to the internet »**
- Écran "Install recommended proprietary software" → **rien coché**
- Installation normalement

#### Obtenir une IP via DHCP

> `dhclient` n'est pas installé sur Ubuntu récent. On a utilisé `nmcli` à la place.

```bash
# Forcer la demande d'IP DHCP
sudo nmcli device connect ens33

# Vérifier l'IP obtenue
ip a

# → a affiché inet 10.10.10.100/24 sur ens33
```

> **pfSense VM1 doit être allumée** pour que le DHCP fonctionne !

#### Accès à l'interface web pfSense 

1. Sur Firefox → `https://10.10.10.1`
2. Avertissement certificat → **« Advanced »** → **« Accept the Risk and Continue »**
3. Login : `admin` / `pfsense`
4. Assistant de configuration :
   - **Hostname** : `pfsense-principal`
   - **Domain** : `arcos.local`
   - **Timezone** : `Europe/Paris`
   - **Mot de passe admin** : Adrien 22
5. **Reload** puis **Finish**

#### Résultat
```
VM3 IP        : 10.10.10.100/24
Passerelle    : 10.10.10.1 (pfSense)
Interface web : https://10.10.10.1 accessible
```

#### Snapshots 
- VM1 : `VM1-pfSense-Principal-BASE`
- VM3 : `VM3-Poste-Admin-BASE`

> De manière générale, on a fait un snapshot avant **chaque étape importante** (IPSec, Suricata, OpenVPN...), puis on a supprimé les anciens au fur et à mesure (par soucis d'économie d'espace disque)

---

### VM2 — pfSense Isolé 

Même installation que VM1, avec ces différences :

| Paramètre | VM1 (Principal) | VM2 (Isolé) |
|---|---|---|
| Hostname | `pfsense-principal` | `pfsense-isole` |
| Network Adapter 3 | `lan-principal` | `lan-isole` |
| IP LAN | `10.10.10.1/24` | `10.20.10.1/24` |
| DHCP Range | `10.10.10.100-199` | `10.20.10.100-199` |

#### Initialisation interface web VM2
Depuis VM4 → `https://10.20.10.1` :
- Hostname : `pfsense-isole`
- Domain : `arcos.local`
- Timezone : `Europe/Paris`
- Mot de passe admin : Adrien 22

#### Résultat 
```
WAN (wan) → em0 → DHCP NAT VMware
LAN (lan) → em2 → 10.20.10.1/24 (DHCP 10.20.10.100-199)
```

---

### VM4 — Postes Internet

- Clone Full de VM3 (donc toujours Ubuntu)
- Carte réseau → **LAN Segment `lan-isole`**

```bash
sudo nmcli device connect ens33
ip a
# → inet 10.20.10.100/24 
```

---

## Étape 3.1 — Interface OPT1 (WANLINK) 

L'interface `em1` (`wan-link`) ajoutée sur les deux pfSense pour simuler l'Internet entre les deux sites.

### Sur VM1 — pfSense Principal
*Interfaces → Assignments → Add em1 → OPT1* :
- Enable 
- Description : `WANLINK`
- IPv4 : Static → `203.0.113.1/24`
- Gateway : None
- Save → Apply Changes

### Sur VM2 — pfSense Isolé
*Interfaces → Assignments → Add em1 → OPT1* :
- Enable 
- Description : `WANLINK`
- IPv4 : Static → `203.0.113.2/24`
- Gateway : None
- Save → Apply Changes

### Règles pare-feu WANLINK (sur les deux pfSense)
*Firewall → Rules → WANLINK → Add* :
- Action : Pass
- Protocol : ICMP
- ICMP Subtypes : any
- Source : any
- Destination : any
- Description : `Autoriser ICMP inter-sites`
- Save → Apply Changes

### Problème rencontré
> IP saisie `203.0.133.1` au lieu de `203.0.113.1` (faute de frappe) → ping échouait. Corrigé dans *Interfaces → WANLINK*.

### Vérification 
Depuis VM1 → *Diagnostics → Ping* :
- Hostname : `203.0.113.2`
- Source : WANLINK
```
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max = 2.132/2.293/2.488 ms 
```

### Snapshots faits
- `VM1-pfSense-Principal-WANLINK-OK`
- `VM2-pfSense-Isole-WANLINK-OK`

---

## Étape 3.2 — Pare-feu, segmentation, DNS over TLS, NTP

### Choix de ne pas implémenter les VLANs

> Dans une architecture réelle, on segmenterait chaque site en plusieurs VLANs (Postes, Serveurs, DMZ, Admin, Invités). Cependant, dans le cadre de ce projet avec **une seule VM cliente par site**, implémenter des VLANs n'apporte pas de valeur démontrable : on ne peut pas peupler plusieurs VLANs simultanément avec une seule machine.
>
> Nous avons donc retenu une architecture **à un LAN par site**, ce qui est cohérent avec notre système à 4 VM. La segmentation VLAN est documentée dans l'architecture cible (étape 1) et serait la première évolution à apporter dans un déploiement réel.
>
> **Ce que des VLANs apporteraient en production :**
> - VLAN 10 — Postes utilisateurs
> - VLAN 20 — Serveurs internes
> - VLAN 30 — DMZ (services exposés Internet)
> - VLAN 40 — Administration
> - VLAN 50 — Invités Wi-Fi
>
> Chaque VLAN serait isolé par des règles inter-VLAN sur pfSense (principe du moindre privilège).

---

### Règles pare-feu — Deny by default

Par défaut pfSense autorise tout le trafic depuis le LAN (`Default allow LAN to any`). On supprime cette règle et on réouvre uniquement ce qui est nécessaire.

**Sur VM1 et VM2 — Firewall → Rules → LAN :**

1. Supprimer la règle `Default allow LAN to any` → Apply Changes

2. Ajouter les règles suivantes (dans l'ordre) :

| # | Description | Protocol | Source | Destination | Port |
|---|---|---|---|---|---|
| 1 | Autoriser DNS | TCP/UDP | LAN subnets | LAN address | 53 |
| 2 | Autoriser NTP | UDP | LAN subnets | LAN address | 123 |
| 3 | Autoriser interface web pfSense | TCP | LAN subnets | LAN address | 443 |
| 4 | Autoriser ICMP diagnostic | ICMP | LAN subnets | any | any |
| 5a | Autoriser HTTP | TCP | LAN subnets | any | 80 |
| 5b | Autoriser HTTPS | TCP | LAN subnets | any | 443 |

> On n'a pas touché à la règle `Anti-Lockout Rule` ni aux règles IPv6

---

### DNS over TLS (Unbound) 

**Sur VM1 et VM2 — Services → DNS Resolver :**
- Enable DNS Resolver 
- Enable SSL/TLS Service 
- Network Interfaces → **All**
- Outgoing Network Interfaces → **WAN**
- Enable Forwarding Mode 
- Use SSL/TLS for outgoing DNS Queries 
- Save → Apply Changes

**Sur VM1 et VM2 — System → General Setup → DNS Server Settings :**

| DNS Server | Hostname |
|---|---|
| `1.1.1.1` | `cloudflare-dns.com` |
| `9.9.9.9` | `dns.quad9.net` |

- DNS Server Override → **décoché** 
- Save

> Problème rencontré : si Network Interfaces = LAN uniquement → erreur "Localhost or All must be selected". Solution : mettre **All**.

---

### NTP

**Sur VM1 et VM2 — Services → NTP :**
- Enable NTP Server 
- Interface → **LAN**
- Time Servers :
  - `0.fr.pool.ntp.org`
  - `1.fr.pool.ntp.org`
- Save

### Snapshots faits
- `VM1-pfSense-Principal-DNS-NTP-OK`
- `VM2-pfSense-Isole-DNS-NTP-OK`

---

## Étape 4 — PKI (Autorité de certification) 

Créée sur **VM1 uniquement** (site principal héberge la PKI).

### Création de la CA

*System → Cert Manager → Authorities → Add* :

| Champ | Valeur |
|---|---|
| Descriptive name | `ARCOS-CA` |
| Method | `Create an internal Certificate Authority` |
| Key type | `RSA` |
| Key length | `2048` |
| Digest Algorithm | `SHA256` |
| Lifetime | `3650` (10 ans) |
| Common Name | `ARCOS-CA` |
| Country Code | `FR` |
| State | `Ile-de-France` |
| City | `Paris` |
| Organization | `ARCOS Solutions` |

### Certificat serveur (pour OpenVPN)

*System → Cert Manager → Certificates → Add/Sign* :

| Champ | Valeur |
|---|---|
| Method | `Create an internal Certificate` |
| Descriptive name | `pfsense-principal-cert` |
| Certificate Authority | `ARCOS-CA` |
| Key type | `RSA 2048` |
| Digest Algorithm | `SHA256` |
| Lifetime | `3650` |
| Common Name | `pfsense-principal` |
| Certificate Type | `Server Certificate` |
| Country/State/City/Org | `FR / Ile-de-France / Paris / ARCOS Solutions` |

### Certificat client (pour télétravailleur OpenVPN)

*System → Cert Manager → Certificates → Add/Sign* :

| Champ | Valeur |
|---|---|
| Method | `Create an internal Certificate` |
| Descriptive name | `client-vpn-admin` |
| Certificate Authority | `ARCOS-CA` |
| Key type | `RSA 2048` |
| Digest Algorithm | `SHA256` |
| Lifetime | `3650` |
| Common Name | `client-vpn-admin` |
| Certificate Type | `User Certificate` |

### Résultat 

| Certificat | Issuer | Type |
|---|---|---|
| GUI default | self-signed | Server (interface web) |
| pfsense-principal-cert | ARCOS-CA | Server (OpenVPN) |
| client-vpn-admin | ARCOS-CA | User (client VPN) |

### Snapshot fait
- `VM1-pfSense-Principal-PKI-OK`

---

## Étape 5 — Tunnel IPSec site-à-site 

### Sur VM1 — Phase 1

*VPN → IPSec → Add P1* :

| Champ | Valeur |
|---|---|
| Key Exchange version | `IKEv2` |
| Internet Protocol | `IPv4` |
| Interface | `WANLINK` |
| Remote Gateway | `203.0.113.2` |
| Authentication Method | `Mutual PSK` |
| My identifier | `My IP address` |
| Peer identifier | `Peer IP address` |
| Pre-Shared Key | `ARCOSipsec2026!` |
| Encryption Algorithm | `AES 256` |
| Hash Algorithm | `SHA256` |
| DH Group | `14 (2048 bit)` |
| Lifetime | `28800` |

→ Save → puis **Add P2** :

| Champ | Valeur |
|---|---|
| Mode | `Tunnel IPv4` |
| Local Network | `LAN subnet` |
| Remote Network | `10.20.10.0/24` |
| Protocol | `ESP` |
| Encryption | `AES 256` |
| Hash | `SHA256` |
| PFS Group | `14` |
| Lifetime | `3600` |

→ Save → Apply Changes

> Problème rencontré : Phase 2 créée avec AES 128 par défaut → corrigé en AES 256 via le crayon d'édition

### Sur VM2 — même config, inverser local/remote

| Champ | Valeur |
|---|---|
| Interface | `WANLINK` |
| Remote Gateway | `203.0.113.1` |
| Pre-Shared Key | `ARCOSipsec2026!` (identique !) |
| Local Network | `LAN subnet` |
| Remote Network | `10.10.10.0/24` |

### Règle pare-feu IPSec (sur les deux pfSense)
*Firewall → Rules → IPsec → Add* :
- Action : Pass · Protocol : Any · Source : any · Destination : any
- Description : `Autoriser trafic IPSec`

### Résultat 
```
Status → IPSec :
  Local  : 203.0.113.1
  Remote : 203.0.113.2
  Status : Established 
  Algo   : AES_CBC(256), HMAC_SHA2_256, MODP_2048
  Child SA : 1 Connected 

Test ping depuis VM3 :
  ping 10.20.10.1
  5 packets transmitted, 5 received, 0% packet loss 
  rtt avg = 8.163 ms 
```

### Snapshots faits
- `VM1-pfSense-Principal-IPSEC-OK`
- `VM2-pfSense-Isole-IPSEC-OK`

---

## Étape 6 — OpenVPN accès distant / test en cours

### Serveur OpenVPN sur VM1

*VPN → OpenVPN → Servers → Add* :

| Champ | Valeur |
|---|---|
| Server Mode | `Remote Access (SSL/TLS)` |
| Protocol | `UDP on IPv4 only` |
| Interface | `WANLINK` |
| Port | `1194` |
| Description | `ARCOS-VPN` |
| TLS Key | `Use a TLS Key` |
| TLS Key Usage Mode | `TLS Authentication` |
| Peer Certificate Authority | `ARCOS-CA` |
| Server Certificate | `pfsense-principal-cert` |
| DH Parameter Length | `2048` |
| Encryption Algorithm | `AES-256-CBC` |
| Auth Digest Algorithm | `SHA256` |
| IPv4 Tunnel Network | `10.30.0.0/24` |
| IPv4 Local Network | `10.10.10.0/24` |
| DNS Default Domain | `arcos.local` |
| DNS Server 1 | `10.10.10.1` |

### Règles pare-feu OpenVPN

**Firewall → Rules → WANLINK → Add** :
- Pass · UDP · Source any · Destination WANLINK address · Port 1194
- Description : `Autoriser OpenVPN entrant`

**Firewall → Rules → OpenVPN → Add** :
- Pass · Any · Source any · Destination any
- Description : `Autoriser trafic tunnel OpenVPN`

### Utilisateur VPN créé

*System → User Manager → Add* :
- Username : `client-vpn-admin`
- Password : `ARCOSvpn2026!` 
- Certificate créé directement via User Manager → ARCOS-CA, RSA 2048, SHA256

### Export du profil client

*System → Package Manager* → installer **openvpn-client-export**

*VPN → OpenVPN → Client Export* :
- Host Name Resolution : `Other`
- Hostname : `203.0.113.1`
- Télécharger : **Inline Configurations → Most Clients** pour `client-vpn-admin`
- Fichier `.ovpn` téléchargé sur VM3

### Test OpenVPN depuis VM3

> ℹTest depuis VM3 (LAN principal) plutôt que VM4 pour éviter de modifier sa config réseau. Dans un déploiement réel, le test se ferait depuis une machine sur le WAN.

```bash
# Sur VM3
sudo apt update && sudo apt install openvpn -y
sudo openvpn --config ~/Downloads/*.ovpn
# → Initialization Sequence Completed 
# → ip a show tun0 → inet 10.30.0.x 
```

### Limitation OpenVPN — contrainte labo 4 VM

Le serveur OpenVPN est fonctionnel :
- Écoute sur `203.0.113.1:1194` 
- Certificats PKI valides 
- Pool `10.30.0.0/24` actif sur interface `ovpns1` 
- Règles pare-feu en place 

Le test client complet n'a pas pu être réalisé car VM3 est sur le même pfSense que le serveur OpenVPN. Une machine externe sur le segment `wan-link` serait nécessaire pour un test complet. Cette limitation est inhérente à notre architecture à 4 VM et est documentée dans le rapport.

---

## Étape 7 — IDS/IPS Suricata

**Installation :**
- *System → Package Manager → Available Packages* → installer **Suricata**

**Configuration :**
- *Services → Suricata → Global Settings* → cocher **ETOpen Emerging Threats rules** 
- *Services → Suricata → Updates* → **Update** → `Result: success` 
- *Services → Suricata → Interfaces → Add* :
  - Interface : `WAN`
  - Block Offenders :
  - Blocking Mode : **Inline Mode (IPS)**
  - Description : `Suricata WAN`

> Warning Hardware Offloading : normal sous VMware, faux positif — ne bloque pas le fonctionnement

**Catégories de règles activées :**
- `emerging-scan.rules` 
- `emerging-exploit.rules` 
- `emerging-dos.rules` 
- `emerging-malware.rules` 

**Test IDS — nmap depuis VM3 :**
```bash
sudo nmap -sS -A -Pn 10.10.10.1
# Ports détectés : 53/tcp (Unbound), 80/tcp (nginx), 443/tcp (nginx)
```

**Résultat dans Suricata → Alerts :**
```
SURICATA ICMPv4 unknown code    → alertes générées 
Generic Protocol Command Decode → trafic scan détecté 
Source : 192.168.244.143 → 192.168.244.141
```

**Snapshot fait :**
- `VM1-pfSense-Principal-SURICATA-OK`

---

## Étape 9 — ntopng + iPerf3 

**Installation ntopng :**
- *System → Package Manager* → installer **ntopng**
- *Services → ntopng Settings* :
  - Monitored Interfaces : `LAN`
  - DNS Mode : `Decode DNS responses`
  - Network List : `10.10.10.0/24` + `10.20.10.0/24`

> ntopng accessible via `http://10.10.10.1:3000` — nécessite règle pare-feu port 3000
> *Firewall → Rules → LAN → Add* : Pass · TCP · LAN subnets → LAN address · port 3000

**Installation iPerf3 :**
```bash
# Sur VM3 et VM4
sudo apt install iperf3 -y
```

**Test de débit :**
```bash
# Sur VM4 (serveur)
iperf3 -s

# Sur VM3 (client)
iperf3 -c 10.20.10.100 -t 30
```

**Résultat ntopng :**
```
Interface  : em2 (LAN)
Débit ↑    : 26.10 Kbps
Débit ↓    : 21.20 Kbps
Flows      : 33 actifs, DPI activé
Protocol   : TCP:TLS
Hosts      : adrien-VMware ↔ pfsense-principal.arcos.local
```

**Snapshot fait :**
- `VM1-pfSense-Principal-NTOPNG-IPERF-OK`

---

## Étape 8 — Squid + SquidGuard + Portail captif

Sur **VM2** (`https://10.20.10.1`) uniquement.

**Installation Squid :**
- *System → Package Manager* → installer **Squid**
- *Services → Squid Proxy Server → Local Cache* :
  - Hard Disk Cache Size : `100` Mo
  - Memory Cache Size : `64` Mo
  - → Save
- *Services → Squid Proxy Server → General* :
  - Enable 
  - Listen Interface : `LAN`
  - Transparent HTTP Proxy 
  - → Save

**Installation SquidGuard :**
- *System → Package Manager* → installer **SquidGuard**
- *Services → SquidGuard → General Settings* :
  - Enable 
  - → Save (sans Apply)
- *Target Categories → Add* :
  - Name : `sites-bloques`
  - Domain List : `facebook.com youtube.com tiktok.com example.com`
  - Redirect mode : `int error page`
  - Redirect : `Acces refuse par ARCOS Solutions`
  - → Save
- *Common ACL → Target Rules* :
  - `sites-bloques` → **deny**
  - Default → **allow**
  - → Save
- *General Settings* → **Apply** → statut STARTED 

> ℹSquidGuard filtre uniquement HTTP — le filtrage HTTPS nécessiterait une interception SSL (bump) avec injection de certificat CA sur les postes clients, hors scope du projet.

**Test filtrage HTTP :**
```
http://example.com → "Acces refuse par ARCOS Solutions"
http://youtube.com → non filtré (HTTPS) → documenté comme limitation
```

**Portail Captif :**
- *System → User Manager → Add* :
  - Username : `user-isole`
  - Password : `ARCOSuser2026!` 
- *Services → Captive Portal → Add* :
  - Zone name : `ARCOScaptif`
  - Interface : `LAN`
  - Authentication : `Local Database`
  - → Save

**Test portail captif :**
```
VM4 → http://example.com → redirection page login 
Login : user-isole / ARCOSuser2026! → accès accordé 
```

**Snapshot fait :**
- `VM2-pfSense-Isole-SQUID-CAPTIF-OK`

---

## Étape 10 — Traffic Shaping
- *Firewall → Traffic Shaper*
- Prioriser trafic VPN/admin sur trafic web

---

## Étape 11 — Plan de tests

| Fonction | Test | Résultat attendu |
|---|---|---|
| Ping inter-sites | Ping VM1 → VM2 via WANLINK | ✅ 0% perte |
| IPSec | Ping VM3 → 10.20.10.1 tunnel | ✅ 0% perte, 8ms |
| OpenVPN | Serveur opérationnel, port 1194, pool 10.30.0.0/24 | ✅ Serveur OK — ⚠️ Test client impossible (contrainte 4 VM) |
| Suricata IDS | nmap -sS -A -Pn 10.10.10.1 depuis VM3 | ✅ Alertes générées |
| Suricata IPS | nmap depuis VM4 | Connexion bloquée |
| Squid | Accès http://example.com depuis VM4 | ✅ Page bloquée |
| Portail captif | Accès sans auth depuis VM4 | ✅ Redirection login |
| ntopng | Trafic iperf3 VM3↔VM4 | ✅ Flux visibles, 26Kbps |
| iPerf3 | Test débit VM3↔VM4 | ✅ Débit mesuré |

---

## Étape 12 — Livrables à rendre
- [ ] Rapport de gestion de projet
- [ ] WBS (Work Breakdown Structure)
- [ ] Diagramme de Gantt
- [ ] Schéma d'architecture réseau
- [ ] Documentation technique (ce document)
- [ ] Fichiers de configuration pfSense (export XML)
- [ ] Plan de tests avec résultats
- [ ] Présentation PowerPoint finale

---

## Référence rapide — Identifiants et IPs

| Élément | Valeur |
|---|---|
| pfSense Principal — IP LAN | `10.10.10.1` |
| pfSense Isolé — IP LAN | `10.20.10.1` |
| pfSense Principal — IP WAN | `203.0.113.1` |
| pfSense Isolé — IP WAN | `203.0.113.2` |
| pfSense — Login | `admin` / `[mot de passe changé]` |
| Pool DHCP Principal | `10.10.10.100` → `10.10.10.199` |
| Pool DHCP Isolé | `10.20.10.100` → `10.20.10.199` |
| Pool VPN clients | `10.30.0.0/24` |
| VM3 Poste Admin | `10.10.10.x` (DHCP) |
| VM4 Postes Internet | `10.20.10.x` (DHCP) |

---

### TERMINÉ
- Étape 0 : Scénario d'entreprise ✅
- Étape 1 : Architecture et plan d'adressage ✅
- Étape 2 : Installation des 4 VM (VM1, VM2, VM3, VM4) ✅
- Étape 2b : Accès interface web pfSense (VM1 et VM2) ✅
- Étape 3a : Interface OPT1 WANLINK + ping inter-sites OK ✅
- Étape 3b : Règles pare-feu deny by default sur VM1 et VM2 ✅
- Étape 3c : DNS over TLS (Unbound + Cloudflare/Quad9) sur VM1 et VM2 ✅
- Étape 3d : NTP (fr.pool.ntp.org) sur VM1 et VM2 ✅
- Étape 4 : PKI — CA ARCOS-CA + certificat serveur + certificat client créés sur VM1 ✅
- Étape 5 : IPSec site-à-site — tunnel établi, ping VM3→VM4 OK (0% perte) ✅
- Étape 6 : OpenVPN — serveur fonctionnel ✅ — test client impossible depuis labo 4 VM ⚠️ (documenté)
- Étape 7 : Suricata IDS — alertes générées sur scan nmap ✅
- Étape 8 : Squid + SquidGuard + Portail captif — filtrage HTTP + auth captif OK ✅
- Étape 9 : ntopng + iPerf3 — flux visualisés, débit mesuré ✅