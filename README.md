# Émulateur d'Imprimante ESC/POS - Fork VelocePrinter

Émulateur d'imprimante pour système de point de vente (POS). Ce fork est optimisé pour l'intégration avec le système **Véloce POS**.

**Note :** Ce fork se concentre exclusivement sur le protocole **ESC/POS** (support ZPL retiré).

---

## 🎯 Objectif du Fork

Intégrer le système POS Véloce pour :
- ✅ **Capturer** les données de factures ESC/POS en temps réel
- 📊 **Analyser** la structure des données via hexdump
- 💾 **Stocker** les factures dans MariaDB (Phase 2)
- 📱 **Exposer** les données aux systèmes d'affichage externes

### Architecture du Système

```
┌──────────────┐    Port 9100    ┌────────────────┐    Hexdump    ┌──────────┐
│  Véloce POS  │ ───────────────> │ VelocePrinter  │ ────────────> │  /logs/  │
└──────────────┘    TCP/ESC-POS   └────────────────┘   Analyse     └──────────┘
                                          │
                                          │ (Phase 2)
                                          v
                                   ┌─────────────┐
                                   │   MariaDB   │
                                   └─────────────┘
```

---

## 🚀 Démarrage Rapide

### 1. Installation

#### Windows
- Télécharger `zpl-escpos-printer-*-setup.exe` depuis [Releases](https://github.com/KimBelanger/VelocePrinter/releases)
- Exécuter l'installateur

#### Linux
```bash
# Debian/Ubuntu
sudo dpkg -i zpl-escpos-printer_*_amd64.deb

# RedHat/CentOS
sudo rpm -i zpl-escpos-printer-*.x86_64.rpm
```

#### macOS
```bash
# Décompresser le zip
unzip Zpl-EscPos.Printer-darwin-*.zip
# Déplacer vers Applications
mv "Zpl-EscPos Printer.app" /Applications/
```

### 2. Configuration

#### Dans VelocePrinter
1. Lancer l'application
2. Cliquer sur l'icône ⚙️ (Paramètres)
3. **Décocher "ZPL Mode"** (on utilise uniquement ESC/POS)
4. Configurer le serveur TCP :
   - **Host** : `0.0.0.0` (écoute toutes les interfaces réseau)
   - **Port** : `9100` (standard imprimantes ESC/POS)
   - **Buffer Size** : `4096` (par défaut, suffit pour factures Véloce)

#### Dans Véloce POS
1. Aller dans **Configuration → Imprimantes**
2. Ajouter une nouvelle imprimante :
   - **Type** : Imprimante réseau ESC/POS
   - **Adresse IP** : `<IP du PC où tourne VelocePrinter>`
   - **Port** : `9100`
   - **Protocole** : TCP/IP
3. Tester avec une facture

### 3. Vérification
1. Imprimer une facture depuis Véloce
2. ✅ La facture s'affiche dans VelocePrinter
3. ✅ Un fichier hexdump est créé dans `/logs/hexdumps/`
4. ✅ Pas de timeout dans Véloce

---

## 📦 Fonctionnalités

### Phase 1 (Actuelle) : Capture Hexdump
- ✅ Capture automatique de toutes les données ESC/POS reçues
- ✅ Stockage en format hexdump lisible (`/logs/hexdumps/`)
- ✅ Métadonnées : timestamp, IP client, taille données
- ✅ Export Base64 pour retraitement
- ✅ Affichage visuel des factures (UI inchangée)

### Phase 2 (À venir) : Parsing & Base de Données
- 🔄 Extraction des données structurées (articles, prix, totaux)
- 🔄 Stockage dans MariaDB
- 🔄 API REST pour systèmes externes
- 🔄 Dashboard d'affichage temps réel

---

## 🔧 Développement

### Prérequis
- **Node.js** 18+ (recommandé : 20 LTS)
- **yarn** (recommandé) ou **npm**
- **Git**

### Installation
```bash
git clone https://github.com/KimBelanger/VelocePrinter.git
cd VelocePrinter
yarn install  # ou: npm install
```

### Commandes
```bash
yarn start       # Mode développement avec logs
yarn package     # Package pour OS courant
yarn make        # Générer binaires multi-OS
```

*Équivalent npm :* `npm start`, `npm run package`, `npm run make`

### Tests Manuels (sans Véloce)
```bash
# Test simple
echo -ne '\x1b\x40TEST\x0a\x1b\x64\x03' | nc localhost 9100

# Facture simulée
cat test_facture.bin | nc localhost 9100
```

### Structure du Projet
```
VelocePrinter/
├── main.js                      # Processus Electron principal
├── package.json                 # Dépendances npm
├── forge.config.js              # Config build Electron Forge
│
├── ZplEscPrinter/
│   ├── main.html               # Interface utilisateur
│   └── js/
│       ├── main.js             # ⭐ Serveur TCP + traitement ESC/POS
│       └── esc_commands.js     # Gestion commandes de status
│
├── logs/
│   └── hexdumps/              # 📁 Captures hexdump (non tracké git)
│
├── _PLAN.md                    # Plan d'implémentation (tracké)
├── claude.md                   # Notes développement (non tracké)
└── README.md                   # Ce fichier
```

---

## 📊 Contexte Opérationnel

### Charge Véloce
- **Factures/jour** : 50-150
- **Factures/heure (max)** : 50
- **Taille moyenne facture** : 256-2048 bytes

### Traitement
- ✅ Asynchrone non-bloquant
- ✅ Buffer accumulatif (gère >64KB)
- ✅ Timeout débounce 100ms (entre chunks)

---

## 📚 Références Techniques

### ESC/POS
- [Documentation ESC/POS](https://escpos.readthedocs.io/en/latest/commands.html)
- [Outils ESC/POS](https://github.com/receipt-print-hq/escpos-tools)
- [Service de rendu](https://github.com/gilbertfl/escpos-netprinter) ([@gilbertfl](https://github.com/gilbertfl))

### Electron
- [Electron Docs](https://www.electronjs.org)
- [Electron Forge](https://www.electronforge.io)

### Projet Original
- [erikn69/ZplEscPrinter](https://github.com/erikn69/ZplEscPrinter) (upstream)

---

## 📝 Notes de Version

### Version 3.1.0-veloce (Fork - Décembre 2025)
- ✅ **Nouveau** : Capture hexdump automatique des données ESC/POS
- ✅ **Nouveau** : Logging structuré (`/logs/hexdumps/`)
- ✅ **Nouveau** : Documentation française complète
- ✅ **Optimisation** : Focus exclusif ESC/POS (ZPL retiré)
- 📋 **Planifié** : Parsing intelligent + MariaDB (Phase 2)

### Version 3.0 (Upstream - erikn69)
- Ajout support ESC/POS
- Refonte architecture
- Support multi-plateforme

---

## 🤝 Contribution

Les contributions sont bienvenues !

1. Fork le projet
2. Créer une branche feature : `git checkout -b feature/ma-feature`
3. Commit : `git commit -m 'Ajout ma feature'`
4. Push : `git push origin feature/ma-feature`
5. Ouvrir une Pull Request

---

## 📄 Licence

**ISC License** - Voir fichier LICENSE

---

## 📸 Captures d'Écran

### Interface Principale
![ESC/POS Demo](./Promo/escpos.png)

### Exemple Hexdump
```
================================================================================
CAPTURE TIMESTAMP: 2025-12-04T14:30:45.123Z
CLIENT: 192.168.1.100:9100
DATA SIZE: 256 bytes
================================================================================

00000000  1b 40 1b 61 01 56 65 6c  6f 63 65 20 50 4f 53 0a  |.@.a.Veloce POS.|
00000010  1b 61 00 46 41 43 54 55  52 45 20 23 31 32 33 34  |.a.FACTURE #1234|
...
```

---

## 🆘 Support

- **Issues GitHub** : [KimBelanger/VelocePrinter/issues](https://github.com/KimBelanger/VelocePrinter/issues)
- **Upstream Original** : [erikn69/ZplEscPrinter](https://github.com/erikn69/ZplEscPrinter)

---

## 🔗 Liens Utiles

- [Guide Véloce POS](https://velocehq.com/) (système de point de vente)
- [Protocole ESC/POS](https://en.wikipedia.org/wiki/ESC/P) (Wikipedia)
- [RFC Imprimantes Réseau](https://www.ietf.org/rfc/rfc1179.txt) (LPD/LPR)
