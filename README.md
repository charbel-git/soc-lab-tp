# TP Cybersécurité — Labo SOC Miniature
### Wazuh + Suricata + Sysmon + VirusTotal + pfSense — Surveillance & Simulation d'attaques

**Contexte** : TP conseillé dans le cadre du programme FORCE-N (Cohorte 6, UN-CHK / Mastercard Foundation, Cisco Networking Academy).
**Environnement** : VMware Workstation, hôte 16 Go RAM.

## État d'avancement

| Brique | Statut |
|---|---|
| pfSense (routeur/firewall) | ✅ Opérationnel |
| Suricata (IDS sur pfSense) | ✅ Installé et actif |
| Wazuh Manager + Dashboard | ✅ Opérationnel — `https://192.168.10.50` |
| Agent Wazuh — Client Linux | ✅ Actif (`client-linux-01`) |
| Sysmon + Agent Wazuh — Client Windows 10 | ⏳ À faire |
| Intégration VirusTotal | ⏳ À faire |
| Simulation d'attaques (Nmap, Hydra) | ⏳ À faire |
| Webinaire réglementation | ⏳ À faire |

---

## 1. Architecture du labo

```
        Internet (simulé via NAT VMware / VMnet8)
             |
        [pfSense] --- Firewall/routeur + Suricata (package IDS intégré)
             |   LAN : VMnet3, Host-only, 192.168.10.0/24
             |
    ---------------------------------------------
    |            |              |                |
 [Wazuh]    [Client Win10]  [Client Linux]    [Kali - attaquant]
 Manager     + Sysmon        + auditd           (allumé seulement
 + Dashboard  + Agent Wazuh   + Agent Wazuh       pendant les tests)
```

| VM | RAM | vCPU | Réseau |
|---|---|---|---|
| pfSense | 1 Go | 1 | em0 = NAT (WAN), em1 = VMnet3 (LAN) |
| Wazuh Manager (all-in-one) | 4 Go | 2 | VMnet3 |
| Client Windows 10 | 3 Go | 2 | VMnet3 |
| Client Linux (Ubuntu Server) | 1 Go | 1 | VMnet3 |
| Kali (attaquant) | 2 Go | 2 | VMnet3 |

---

## 2. Détails par brique

📄 **[pfsense.md](./pfsense.md)** — Installation pfSense, troubleshooting réseau complet, Suricata IDS. ✅ Terminé.

---

## 3. Journal de troubleshooting — Wazuh Manager

### Problème 1 : Session live Ubuntu au lieu d'une vraie installation
- Symptôme : après la config réseau/SSH initiale, tout redevenait vide au redémarrage. `df -h` révélait un montage `/cow` (copy-on-write) et une partition racine de seulement 1,9 Go — signature d'une **session live temporaire** ("Try Ubuntu"), pas d'une installation réelle sur le disque de 40 Go créé.
- Cause : au menu de boot de l'ISO, l'option "Try or Install Ubuntu Server" n'avait pas été confirmée sur "Install" à l'écran suivant.
- **Solution** : suppression de la VM, recréation, et réinstallation complète en s'assurant de bien sélectionner **"Install Ubuntu Server"** (pas "Try") à l'écran de choix, puis en **attendant explicitement le message "Installation complete!"** avant tout reboot ou retrait d'ISO.
- **Leçon retenue** : toujours vérifier `df -h` juste après le premier login — une partition racine anormalement petite (quelques Go) avec un point de montage `/cow` est un signal fiable de session live plutôt que d'installation réelle.

### Problème 2 : Boot en PXE (« Operating System not found »)
- Symptôme : après la (fausse) installation, la VM tentait de booter par le réseau (PXE) et affichait une erreur, faute de trouver un OS sur le disque.
- Cause réelle : conséquence directe du Problème 1 (disque jamais réellement écrit, taille du `.vmdk` = 512 Ko) — pas un souci d'ordre de boot en tant que tel.
- Tentatives de contournement testées (utiles à connaître même si non suffisantes seules) : forcer l'ordre de boot dans le fichier `.vmx` avec `bios.bootOrder = "hdd,cdrom"` et `bios.bootDelay = "3000"`.
- **Solution définitive** : réinstallation complète en corrigeant le Problème 1 à la racine.

### Problème 3 : Vérification matérielle bloquante au lancement du script Wazuh
- Message : `ERROR: Your system does not meet the recommended minimum hardware requirements of 4Gb of RAM and 2 CPU cores.`
- Cause : la VM affichait `free -h` = 3,3 Gi total (au lieu des 4096 Mo alloués dans VMware) — écart normal dû à la réservation mémoire du firmware/BIOS de la VM.
- **Solution** : contournement du contrôle strict avec le flag `-i` :
  ```bash
  sudo bash ./wazuh-install.sh -a -i
  ```

### Configuration finale validée — Wazuh Manager
- Ubuntu Server 26.04 LTS, 4 Go RAM (3,3 Gi vus par le système), 2 vCPU, disque 40 Go
- IP statique : `192.168.10.50/24` via Netplan, gateway `192.168.10.1`
- Installation Wazuh all-in-one (Indexer + Manager + Dashboard) via script officiel, version `4.14`
- Dashboard accessible : `https://192.168.10.50`

---

## 4. Journal de troubleshooting — Client Linux (agent Wazuh)

### Problème 1 : Confusion "Create bond" pendant la config réseau
- L'installeur Ubuntu propose une option "Create bond" (agrégation de cartes réseau) qui n'est pas nécessaire pour une VM à interface(s) simple(s).
- **Solution** : ignorer cette option, sélectionner directement l'interface physique proposée (`ens33` ou équivalent) et configurer l'IP dessus.

### Problème 2 : Deuxième carte réseau ajoutée sans s'en rendre compte
- La VM s'est retrouvée avec **2 cartes réseau** (`ens33` en NAT, `ens37` sur VMnet3) au lieu d'une seule comme prévu initialement — configuration plus proche de celle de la VM Wazuh que du client "léger" prévu au départ.
- Conséquence : complexifie le routage mais reste gérable (voir Problème 4).

### Problème 3 : Résolution DNS impossible pendant l'installation (mirroir Ubuntu)
- Message : `Erreur temporaire de resolution de <ml.archive.ubuntu.com>` pendant la phase `configuring apt`.
- Cause : à ce stade de l'installation, la VM n'avait pas encore de route réseau fonctionnelle vers Internet (voir Problème 4, cause profonde commune).
- **Décision** : ne pas bloquer l'installation pour ça — les paquets essentiels viennent du CD (`/cdrom`), l'installation aboutit normalement malgré l'avertissement.

### Problème 4 : Route par défaut pointant vers la mauvaise interface (cause racine des deux problèmes réseau précédents)
- Symptôme : `ping 192.168.10.1` fonctionnait, mais `ping 8.8.8.8` échouait avec `Destination Host Unreachable` renvoyé par l'IP WAN de pfSense elle-même — signe que le paquet arrivait jusqu'à pfSense mais que le retour se perdait.
- Diagnostic mené :
  1. Vérification du pare-feu LAN pfSense (règle "Default allow LAN to any" présente) → OK
  2. Vérification du NAT sortant pfSense (`Automatic outbound NAT`, règle couvrant `192.168.10.0/24` → WAN) → OK
  3. Vérification de la gateway WAN pfSense (`Status > Gateways`, WAN_DHCP en `Online`) → OK
  4. Test `ping -S 192.168.10.1 8.8.8.8` depuis la console pfSense (simulation du trafic forwardé) → **réussi**, ce qui innocentait complètement pfSense
  5. `ip route show` côté client Linux → révèle la vraie cause : une route statique `default via 192.168.10.1 dev ens33` alors que `192.168.10.1` n'existe que sur `ens37`, pas sur `ens33` (carte NAT)
- **Cause racine identifiée** : conflit entre **deux fichiers Netplan simultanés** :
  - `00-installer-config.yaml` (généré automatiquement par l'installeur Ubuntu, subiquity) — avait inversé les cartes, plaçant l'IP LAN statique et la route par défaut sur `ens33` (NAT) au lieu de `ens37` (LAN)
  - `50-cloud-init.yaml` (édité manuellement) — configuration correcte mais entrant en conflit avec le premier fichier lors de la fusion Netplan
- **Solution** :
  1. Vider `00-installer-config.yaml` pour ne garder qu'un fichier minimal (`network:\n  version: 2`), afin qu'il n'entre plus en conflit
  2. Confirmer que `50-cloud-init.yaml` contient la config correcte sur `ens37` (IP statique + route par défaut) et `ens33` en DHCP simple
  3. `sudo netplan apply`
  4. Résultat validé : route par défaut vers Internet via `ens33` (DHCP/NAT), réseau local `192.168.10.0/24` routé directement via `ens37` (route de sous-réseau, sans besoin de route par défaut dédiée) — configuration saine et fonctionnelle
- **Leçon retenue** : sur Ubuntu Server (subiquity), toujours vérifier `ls /etc/netplan/` après installation — la présence de plusieurs fichiers `.yaml` est une source fréquente de conflits de routage, surtout sur des VM à plusieurs interfaces réseau. Un `cat` de chaque fichier permet de repérer rapidement une inversion de cartes.

### Configuration finale validée — Client Linux
- Ubuntu Server 26.04 LTS, specs légères (RAM/CPU réduits par rapport à Wazuh), disque 20 Go
- `ens33` : DHCP (NAT, accès Internet)
- `ens37` : IP statique `192.168.10.60/24` (LAN, communication avec pfSense/Wazuh)
- Agent Wazuh `4.14.7` installé et enregistré sous le nom `client-linux-01`, statut **Active** confirmé dans le Dashboard

---

## 5. Prochaines étapes — Guide complet (à suivre dans l'ordre)

### ✅ Étape A — Suricata sur pfSense (IDS réseau) — FAIT
1. Interface web pfSense → **System > Package Manager > Available Packages**
2. Rechercher `Suricata`, cliquer **Install**
3. Une fois installé : **Services > Suricata > Interfaces** → **Add**, sélectionner l'interface **LAN**
4. Onglet **WAN/LAN Settings** de l'interface ajoutée : activer les catégories de règles (ET Open est gratuit et suffisant pour un labo) sous l'onglet **Rules**
5. Démarrer Suricata sur l'interface (bouton play vert)
6. Doc officielle : `https://docs.netgate.com/pfsense/en/latest/packages/suricata/index.html`

### ✅ Étape B — Wazuh Manager (VM dédiée, 4 Go RAM) — FAIT (voir journal détaillé section 2bis)
1. VM Ubuntu Server (ou Debian) sur VMnet3, IP statique conseillée dans la plage DHCP réservée ou hors plage (ex. `192.168.10.50`)
2. Installation "all-in-one" (Indexer + Manager + Dashboard) via le script officiel :
   ```bash
   curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && chmod 744 wazuh-install.sh && sudo bash ./wazuh-install.sh -a
   ```
3. Le script génère les identifiants admin du Dashboard à la fin — **les noter immédiatement**
4. Accès Dashboard : `https://<IP_Wazuh_Manager>` (port 443)
5. Doc officielle Quickstart : `https://documentation.wazuh.com/current/quickstart.html`

### Étape C — Agent Wazuh + Sysmon sur le client Windows 10
1. **Installer Sysmon d'abord** :
   - Téléchargement : `https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon`
   - Config recommandée pour débuter (config communautaire éprouvée, réduit le bruit) : `https://github.com/SwiftOnSecurity/sysmon-config`
   - Installation : `sysmon64.exe -accepteula -i sysmonconfig-export.xml`
2. **Installer l'agent Wazuh** :
   - Téléchargement MSI + guide : `https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html`
   - Lors de l'install, renseigner l'IP du Wazuh Manager (`192.168.10.50` ou ton IP choisie)
3. Dans `ossec.conf` de l'agent, ajouter la collecte des logs Sysmon (canal `Microsoft-Windows-Sysmon/Operational`) — instructions détaillées dans la doc ci-dessus
4. Vérifier dans le Dashboard Wazuh que l'agent apparaît **Active**

### ✅ Étape D — Agent Wazuh sur le client Linux — FAIT (voir journal détaillé section 2ter)
1. Doc officielle : `https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html`
2. Installer et activer `auditd` en complément pour la surveillance des appels système et de l'authentification — **⏳ auditd reste à installer**, seul l'agent Wazuh est fait pour l'instant
3. ~~Enregistrer l'agent auprès du Manager (IP + clé), vérifier statut Active dans le Dashboard~~ → Statut **Active** confirmé pour `client-linux-01`

### Étape E — Intégration VirusTotal dans Wazuh
1. Créer un compte gratuit VirusTotal, récupérer une clé API (quota gratuit limité, ~4 requêtes/min)
2. Configurer l'intégration côté Wazuh Manager dans `ossec.conf` :
   ```xml
   <integration>
     <name>virustotal</name>
     <api_key>TA_CLE_API</api_key>
     <alert_format>json</alert_format>
   </integration>
   ```
3. Doc officielle : `https://documentation.wazuh.com/current/user-manual/manager/manual-malware-detection/virustotal-integration.html`
4. Cette intégration enrichit automatiquement les alertes de fichiers suspects avec le verdict VirusTotal (hash lookup)

### Étape F — Simulation d'attaques (depuis Kali, déjà installé)
1. **Scan Nmap** vers les clients Windows/Linux :
   ```bash
   nmap -sV -A 192.168.10.0/24
   ```
2. **Brute force SSH** (client Linux) avec Hydra — usage strictement limité à ton propre labo :
   ```bash
   hydra -l <user_test> -P /usr/share/wordlists/rockyou.txt ssh://192.168.10.x
   ```
3. Observer dans le Dashboard Wazuh + Suricata les alertes générées (scan de ports détecté, tentatives d'authentification échouées répétées)
4. Documenter les règles Wazuh/Suricata qui se déclenchent — c'est le cœur de l'analyse pour ton rapport

### Étape G — Webinaire réglementation
- À intégrer séparément selon le contenu donné par ton formateur (RGPD, ISO 27001, cadre légal malien/régional le cas échéant) — dis-moi quand tu as le support du webinaire, je peux t'aider à faire le lien avec les résultats techniques du labo (ex. obligations de notification d'incident, durée de conservation des logs).

---

## 6. Structure du repo GitHub

```
/soc-lab-tp/
  README.md              ← ce fichier : contexte, architecture, état d'avancement, liens vers le détail
  pfsense.md             ← ✅ fait : install pfSense + troubleshooting + Suricata
  wazuh.md                (à créer quand tu scindes la section 3 ci-dessus)
  client-linux.md          (à créer quand tu scindes la section 4 ci-dessus)
  client-windows.md        (à créer une fois Windows fait)
  virustotal.md             (à créer une fois fait)
  attack-simulation.md       (captures Nmap/Hydra + alertes détectées)
  webinaire-reglementation.md
  screenshots/
```

**Approche recommandée** : commit au fur et à mesure, pas tout d'un coup. Chaque fois qu'une brique est terminée, tu extrais sa section de ce README vers son propre fichier `.md` (comme on vient de le faire pour pfSense), tu ajoutes les captures d'écran correspondantes dans `screenshots/`, et tu commits avec un message clair (ex. `"Add Wazuh Manager troubleshooting"`).

Montrer le troubleshooting réel est un vrai plus dans un portfolio GitHub — ça prouve une capacité de diagnostic, pas juste un suivi de tutoriel.
