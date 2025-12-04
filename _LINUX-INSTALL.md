# Guide d'Installation VelocePrinter sur Linux (Ubuntu/Debian)

## Prérequis Système

- **Ubuntu 20.04 LTS** ou **Debian 11+** (ou distributions compatibles)
- **CPU** : x64 ou ARM64
- **RAM** : 512 MB minimum, 1GB recommandé
- **Espace disque** : 500 MB minimum
- **Connexion réseau** : Interface Ethernet ou WiFi configurée
- **Accès sudo** ou root (pour certaines étapes)

---

## Étapes d'Installation et Configuration

### 1. Mettre à jour le système

```bash
sudo apt update
sudo apt upgrade -y
```

Cela télécharge les listes de paquets et installe les dernières mises à jour de sécurité.

---

### 2. Installer les dépendances système requises

```bash
sudo apt install -y curl wget git libssl-dev libxss1 libgconf-2-4
```

**Détail des paquets :**
- `curl` / `wget` : Téléchargement fichiers
- `git` : Clonage du repository
- `libssl-dev` : Support SSL/TLS
- `libxss1` : Dépendance Electron
- `libgconf-2-4` : Dépendance Electron

---

### 3. Installer Node.js et yarn

#### Option A : Utiliser NodeSource (recommandé - Node.js 20 LTS)

```bash
# Ajouter le repository NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Installer Node.js et npm
sudo apt install -y nodejs

# Vérifier l'installation
node --version  # Doit afficher v20.x.x
npm --version   # Doit afficher 10.x.x
```

#### Option B : Utiliser apt (version 18+ minimum)

```bash
sudo apt install -y nodejs npm

# Vérifier la version
node --version
```

#### Installer yarn (gestionnaire de paquets)

```bash
sudo npm install -g yarn

# Vérifier l'installation
yarn --version  # Doit afficher 1.22.x
```

---

### 4. Cloner le repository VelocePrinter

```bash
# Créer un répertoire pour l'application
mkdir -p ~/applications
cd ~/applications

# Cloner le fork VelocePrinter
git clone https://github.com/KimBelanger/VelocePrinter.git
cd VelocePrinter

# Vérifier la branche principale
git branch -a
```

---

### 5. Installer les dépendances du projet

```bash
cd ~/applications/VelocePrinter

# Installer les dépendances Node.js
yarn install

# Cette étape peut prendre 5-10 minutes
# Elle télécharge tous les paquets npm et Electron
```

**Si vous préférez npm :**
```bash
npm install
```

---

### 6. Vérifier l'installation en mode développement

```bash
# Tester l'application (interface graphique + serveur TCP)
yarn start

# Logs :
# - Electron window s'ouvre (X11 forwarding si accès SSH)
# - Serveur TCP écoute sur 0.0.0.0:9100
# - Console affiche les connexions

# Pour arrêter : CTRL+C
```

**Alternative npm :**
```bash
npm start
```

---

### 7. Configurer l'application VelocePrinter

#### Via Interface Graphique (si accès local)

1. Dans la fenêtre Electron qui s'ouvre :
   - Cliquer sur ⚙️ (Paramètres)
   - Décocher "ZPL Mode" (uniquement ESC/POS)
   - Vérifier les paramètres TCP :
     - **Host** : `0.0.0.0` (écoute toutes les interfaces)
     - **Port** : `9100` (standard imprimantes ESC/POS)
     - **Buffer Size** : `4096`
   - Cliquer "Enregistrer" ou "Save"

#### Ou configurer via SSH (headless)

L'application stocke sa configuration dans :
```bash
# Linux : ~/.config/zpl-escpos-printer/config.json
# Ou : ~/.local/share/zpl-escpos-printer/config.json

# Éditer directement (avant 1ère exécution)
cat ~/.config/zpl-escpos-printer/config.json 2>/dev/null || echo "À créer"
```

---

### 8. Installer comme service systemd (démarrage auto)

#### Créer le fichier service

```bash
sudo tee /etc/systemd/system/veloceprinter.service > /dev/null <<'EOF'
[Unit]
Description=VelocePrinter - ESC/POS Printer Emulator for Véloce POS
Documentation=https://github.com/KimBelanger/VelocePrinter
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/home/$USER/applications/VelocePrinter
Environment="DISPLAY=:0"
Environment="NODE_ENV=production"
Environment="ELECTRON_ENABLE_LOGGING=1"

# Adapter le chemin si Node.js en location différente
ExecStart=/usr/bin/node /home/$USER/applications/VelocePrinter/main.js

# Restart policy
Restart=on-failure
RestartSec=5s

# Logs
StandardOutput=journal
StandardError=journal
SyslogIdentifier=veloceprinter

[Install]
WantedBy=multi-user.target
EOF
```

**⚠️ Important : Remplacer `$USER` par votre nom d'utilisateur réel :**

```bash
# Récupérer votre username
whoami

# Puis adapter le fichier service
sudo nano /etc/systemd/system/veloceprinter.service
# Remplacer $USER par ex. : ubuntu, debian, root, etc.
```

#### Activer et démarrer le service

```bash
# Recharger la configuration systemd
sudo systemctl daemon-reload

# Activer au démarrage
sudo systemctl enable veloceprinter.service

# Démarrer le service
sudo systemctl start veloceprinter.service

# Vérifier le statut
sudo systemctl status veloceprinter.service

# Afficher les logs
sudo journalctl -u veloceprinter.service -f  # (-f = suivi continu)
```

---

### 9. Configurer le firewall (si activé)

```bash
# Vérifier le firewall
sudo ufw status

# Si actif, autoriser le port 9100
sudo ufw allow 9100/tcp

# Vérifier
sudo ufw status
```

---

### 10. Tester la connectivité réseau

#### A. Vérifier que le service écoute

```bash
# Depuis le PC Linux
sudo netstat -tlnp | grep 9100

# Ou avec ss (plus moderne)
ss -tlnp | grep 9100

# Doit afficher quelque chose comme :
# tcp  LISTEN  0  5  0.0.0.0:9100  0.0.0.0:*
```

#### B. Tester depuis un autre PC du réseau

```bash
# Depuis PC Véloce POS (Windows/macOS/Linux)
nc -zv <IP_LINUX> 9100

# Doit afficher : Connection to <IP_LINUX> 9100 port [tcp/*] succeeded!

# Ou utiliser telnet (si disponible)
telnet <IP_LINUX> 9100
# CTRL+] puis quit
```

#### C. Tester envoi de données

```bash
# Depuis PC Véloce POS
echo -ne '\x1b\x40TEST\x0a\x1b\x64\x03' | nc <IP_LINUX> 9100

# Doit être reçu sur Linux et générer un fichier dans /logs/hexdumps/
```

---

### 11. Configurer Véloce POS pour imprimer sur VelocePrinter

#### Dans l'interface Véloce POS

1. Aller dans : **Configuration → Imprimantes**
2. Ajouter une nouvelle imprimante :
   - **Nom** : VelocePrinter (ou autre)
   - **Type** : Imprimante réseau ESC/POS
   - **Protocole** : TCP/IP
   - **Adresse IP** : `<IP_DU_PC_LINUX>` (ex: 192.168.1.50)
   - **Port** : `9100`
   - **Délai d'attente** : 5 secondes (par défaut)
   - **Largeur papier** : 80mm (dépend du POS)

3. Cliquer "Tester" ou "Test Print"
4. Vérifier :
   - ✅ Pas d'erreur timeout dans Véloce
   - ✅ Une facture test s'affiche dans VelocePrinter
   - ✅ Un fichier hexdump créé dans `/logs/hexdumps/`

---

### 12. Vérifier les captures hexdump

```bash
# Lister les captures générées
ls -lah ~/applications/VelocePrinter/logs/hexdumps/

# Exemple de structure de fichier
cat ~/applications/VelocePrinter/logs/hexdumps/escpos_*.txt | head -30

# Nombre total de factures capturées
ls ~/applications/VelocePrinter/logs/hexdumps/ | wc -l
```

---

### 13. Monitoring et Maintenance

#### Vérifier l'espace disque des logs

```bash
# Taille du dossier logs
du -sh ~/applications/VelocePrinter/logs/

# Nettoyer les vieux hexdumps (>30 jours)
find ~/applications/VelocePrinter/logs/hexdumps/ -mtime +30 -delete
```

#### Surveiller en temps réel

```bash
# Suivi des logs du service
sudo journalctl -u veloceprinter.service -f

# Ou : suivi des hexdumps générés
watch -n 1 'ls -t ~/applications/VelocePrinter/logs/hexdumps/ | head -5'
```

#### Redémarrer le service

```bash
# Redémarrage propre
sudo systemctl restart veloceprinter.service

# Arrêter
sudo systemctl stop veloceprinter.service

# Relancer
sudo systemctl start veloceprinter.service
```

---

### 14. Mise à Jour du Code

```bash
cd ~/applications/VelocePrinter

# Récupérer les dernières modifications
git pull origin main

# Réinstaller les dépendances (si changements)
yarn install

# Redémarrer le service
sudo systemctl restart veloceprinter.service

# Vérifier que tout fonctionne
sudo systemctl status veloceprinter.service
```

---

### 15. Dépannage

#### Application n'écoute pas sur 9100

```bash
# Vérifier les erreurs
sudo journalctl -u veloceprinter.service -n 50

# Port en usage ?
sudo lsof -i :9100

# Libérer le port (à éviter en prod)
sudo fuser -k 9100/tcp
```

#### Connexion Véloce timeout

```bash
# Tester la connectivité
nc -zv <IP_LINUX> 9100

# Vérifier le firewall
sudo ufw status
sudo ufw allow 9100/tcp

# Vérifier la réponse TCP
echo test | nc <IP_LINUX> 9100  # Doit fermer la connexion proprement
```

#### Permissions fichiers

```bash
# Vérifier les permissions du dossier logs
ls -ld ~/applications/VelocePrinter/logs/

# Corriger si nécessaire
chmod 755 ~/applications/VelocePrinter/logs/
chmod 755 ~/applications/VelocePrinter/logs/hexdumps/
```

#### Espace disque insuffisant

```bash
# Vérifier l'espace
df -h

# Nettoyer les anciens hexdumps
find ~/applications/VelocePrinter/logs/hexdumps/ -mtime +7 -delete  # >7 jours
```

---

## Résumé de la Configuration Finale

### Checklist de Vérification

- [ ] Ubuntu/Debian installé et mis à jour (`sudo apt update && upgrade`)
- [ ] Node.js 18+ et yarn installés (`node -v`, `yarn -v`)
- [ ] Repository cloné (`cd ~/applications/VelocePrinter`)
- [ ] Dépendances installées (`yarn install`)
- [ ] Service systemd créé et activé (`systemctl status veloceprinter`)
- [ ] Port 9100 accessible (`netstat -tlnp | grep 9100`)
- [ ] Firewall autorise 9100/tcp (`ufw allow 9100/tcp`)
- [ ] Véloce POS configuré avec IP:port corrects
- [ ] Test facture reçue et hexdump généré
- [ ] Service redémarre automatiquement (`systemctl enable`)

### Fichiers de Référence

```
~/applications/VelocePrinter/
├── ZplEscPrinter/js/main.js      # Code TCP server (port 9100)
├── logs/hexdumps/                # Captures ESC/POS (auto-créé)
├── _PLAN.md                       # Plan Phase 1/2
├── README.md                      # Documentation générale
└── _LINUX-INSTALL.md            # Ce guide
```

### Support et Documentation

- **Plan détaillé** : [_PLAN.md](_PLAN.md)
- **Documentation générale** : [README.md](README.md)
- **Issues GitHub** : https://github.com/KimBelanger/VelocePrinter/issues
- **Logs du service** : `sudo journalctl -u veloceprinter.service -f`

---

## Prochaines Étapes (Phase 2)

Une fois que vous avez collecté 10-20 hexdumps de factures réelles Véloce :

1. Analyser la structure ESC/POS
2. Identifier les patterns (articles, prix, totaux)
3. Implémenter le parsing intelligent
4. Intégrer MariaDB pour stockage persistant
5. Exposer API REST pour systèmes externes

Voir [_PLAN.md](_PLAN.md) pour détails complets.
