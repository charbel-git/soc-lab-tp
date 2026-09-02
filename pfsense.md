# pfSense — Installation, configuration et troubleshooting

Routeur/firewall du labo, avec Suricata (IDS) installé en package. LAN sur `192.168.10.0/24` (VMnet3, Host-only), WAN en NAT VMware.

---

## Téléchargement de l'ISO

Le lien officiel `pfsense.org/download` redirige désormais vers le **Netgate Store**, qui demande la création d'un compte + informations de carte bancaire, même pour un téléchargement à 0 $.

**Solution retenue** : miroir officiel Netgate sans compte requis :
`https://atxfiles.netgate.com/mirror/downloads/`
→ Fichier utilisé : `pfSense-CE-2.7.2-RELEASE-amd64.iso.gz`

Vérification d'intégrité effectuée avec `Get-FileHash -Algorithm SHA256` comparé au fichier `.sha256` fourni à côté. **Conforme.**

---

## Journal de troubleshooting

### Problème 1 : IP du LAN non appliquée après le wizard console
- **Symptôme** : `ifconfig em1` n'affichait aucune ligne `inet` après configuration via le menu "2 Set interface(s) IP address".
- **Cause identifiée** : conflit d'adressage IP côté hôte Windows — l'adaptateur `VMware Network Adapter VMnet3` avait **deux IP simultanées** (`192.168.10.1` ajoutée automatiquement par VMware après modification du subnet, en plus de `192.168.10.5` ajoutée manuellement), créant un conflit direct avec l'IP LAN de pfSense (`192.168.10.1`).
- **Solution** : suppression de l'IP en doublon côté Windows avec `Remove-NetIPAddress`.

### Problème 2 : Interface em1 marquée `(down)` lors du ré-assignement
- **Cause** : le lien réseau virtuel de l'Adapter 2 dans VMware s'était déconnecté suite aux changements successifs de configuration réseau (VMnet, subnet).
- **Solution** : reconnexion manuelle via VMware (VM > Removable Devices > Network Adapter 2 > Connect), puis vérification que les cases **"Connected"** et **"Connect at power on"** étaient cochées dans les Settings.

### Problème 3 : Configuration réseau incohérente entre VMware et pfSense
- Le Virtual Network Editor avait initialement assigné un subnet différent (ex. `192.168.139.0`) à VMnet3, différent du `192.168.10.0/24` prévu pour le LAN pfSense.
- **Solution** : correction directe du Subnet IP de VMnet3 dans le Virtual Network Editor (`Edit > Virtual Network Editor`, sélection VMnet3, changement du Subnet IP vers `192.168.10.0/24`).

### Problème 4 : Corruption de configuration (Fatal PHP Error)
- Après plusieurs reconfigurations successives des interfaces, une erreur fatale PHP est survenue au boot (`Uncaught ValueError: Path cannot be empty in /etc/inc/notices.inc`), signe d'une corruption du fichier de config système.
- **Décision** : réinstallation complète de pfSense plutôt que dépannage — plus rapide et plus fiable qu'un correctif en profondeur sur un labo sans données critiques à préserver.
- **Leçon retenue** : configurer le Virtual Network Editor (VMnet, subnet, host adapter) **avant** de lancer l'installation, pour éviter les reconfigurations réseau après coup qui fragilisent la config pfSense.

### Problème 5 : Avertissement PHP + échec de mise à jour système
- **Message** : `WARNING: Current pkg repository has a new PHP major version` lors d'une tentative d'installation de package (Suricata), suivi d'un échec de `pfSense-upgrade` : `ERROR: It was not possible to determine pfSense-upgrade remote version`.
- **Cause** : bug connu sur pfSense 2.7.x lié au magasin de certificats.
- **Solution** :
  ```
  certctl rehash
  pfSense-upgrade
  ```
  (alternative si insuffisant : `pkg-static upgrade`, puis `pkg-static upgrade -f` en cas d'erreur de version du kernel)

---

## Configuration finale validée
- Partitionnement : Auto (UFS), Entire Disk, GPT
- VLANs au premier boot : **Non**
- WAN = em0 (NAT VMware, DHCP) — `192.168.50.138/24`
- LAN = em1 (VMnet3, Host-only) — `192.168.10.1/24`
- DHCP LAN activé : plage `192.168.10.100` – `192.168.10.200`
- webConfigurator : HTTPS
- Ping hôte ↔ pfSense : **fonctionnel**
- Interface web accessible : `https://192.168.10.1`
- Mot de passe admin par défaut changé lors du Setup Wizard

---

## Suricata (IDS) — installation

1. Interface web pfSense → **System > Package Manager > Available Packages**
2. Rechercher `Suricata`, cliquer **Install**
3. Une fois installé : **Services > Suricata > Interfaces** → **Add**, sélectionner l'interface **LAN**
4. Onglet **WAN/LAN Settings** de l'interface ajoutée : activer les catégories de règles (ET Open est gratuit et suffisant pour un labo) sous l'onglet **Rules**
5. Démarrer Suricata sur l'interface (bouton play vert)

Doc officielle : `https://docs.netgate.com/pfsense/en/latest/packages/suricata/index.html`

**Statut :  Installé et actif**
