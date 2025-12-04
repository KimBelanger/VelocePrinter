# Plan d'Implémentation - Capture Hexdump ESC/POS

## Contexte
Intégration du système POS **Véloce** avec VelocePrinter pour capturer, analyser et stocker les données de factures ESC/POS.

**Note :** Ce fork se concentre uniquement sur **ESC/POS** (pas ZPL). Port d'écoute : **9100**.

## Phase 1 : Capture des Hexdumps (Priorité Immédiate)

### Objectifs
1. ✅ Capturer toutes les données brutes ESC/POS reçues du POS Véloce
2. ✅ Sauvegarder en format texte hexdump lisible pour analyse
3. ✅ Maintenir le fonctionnement actuel (affichage UI inchangé)
4. ✅ Répondre correctement aux commandes de status (pas de timeout Véloce)

### Fichiers à Modifier

#### 1. `/ZplEscPrinter/js/main.js` ⭐ CRITIQUE

##### A. Ajouter les imports (début du fichier, après les imports existants)
```javascript
const fs = require('fs');
const path = require('path');
```

##### B. Créer la fonction `captureHexdump()` (avant la fonction `escpos()`, vers ligne 180)
```javascript
/**
 * Capture raw ESC/POS data as hexdump for analysis
 * @param {Buffer} data - Raw ESC/POS data received from TCP
 * @param {Object} socketInfo - Client connection info {peerAddress, peerPort}
 * @returns {string} Path to saved hexdump file
 */
async function captureHexdump(data, socketInfo) {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const filename = `escpos_${timestamp}_${socketInfo.peerAddress.replace(/:/g, '-')}_${socketInfo.peerPort}.txt`;
    const logsDir = path.join(__dirname, '../../logs/hexdumps');

    // Ensure logs directory exists
    if (!fs.existsSync(logsDir)) {
        fs.mkdirSync(logsDir, { recursive: true });
    }

    // Generate hexdump content
    let hexdump = '';
    hexdump += '='.repeat(80) + '\n';
    hexdump += `CAPTURE TIMESTAMP: ${new Date().toISOString()}\n`;
    hexdump += `CLIENT: ${socketInfo.peerAddress}:${socketInfo.peerPort}\n`;
    hexdump += `DATA SIZE: ${data.length} bytes\n`;
    hexdump += '='.repeat(80) + '\n\n';

    // Hexdump format: offset | hex bytes | ASCII
    for (let i = 0; i < data.length; i += 16) {
        const chunk = data.slice(i, i + 16);
        const offset = i.toString(16).padStart(8, '0');

        // Hex representation
        const hex = Array.from(chunk)
            .map(b => b.toString(16).padStart(2, '0'))
            .join(' ');
        const hexPadded = hex.padEnd(47, ' '); // 16 bytes * 3 chars - 1

        // ASCII representation (printable chars only)
        const ascii = Array.from(chunk)
            .map(b => (b >= 32 && b <= 126) ? String.fromCharCode(b) : '.')
            .join('');

        hexdump += `${offset}  ${hexPadded}  |${ascii}|\n`;
    }

    hexdump += '\n' + '='.repeat(80) + '\n';
    hexdump += 'RAW BASE64 (for re-processing):\n';
    hexdump += data.toString('base64') + '\n';
    hexdump += '='.repeat(80) + '\n';

    // Write to file
    const filepath = path.join(logsDir, filename);
    fs.writeFileSync(filepath, hexdump, 'utf8');

    console.log(`[HEXDUMP] Captured ${data.length} bytes to: ${filename}`);
    notify(`Hexdump capturé: <b>${filename}</b> (${data.length} bytes)`, 'floppy-disk', 'success', 3000);

    return filepath;
}
```

##### C. Modifier la fonction `processData()` - **LIGNE 369** (point d'injection)
**AVANT :**
```javascript
if ($('#isZpl').is(':checked'))
    response = await zpl(code);
else
    response = await escpos(code);
```

**APRÈS :**
```javascript
if ($('#isZpl').is(':checked'))
    response = await zpl(code);
else {
    // NOUVEAU : Capture hexdump pour analyse Véloce
    await captureHexdump(code, clientSocketInfo);
    // Traitement normal inchangé
    response = await escpos(code);
}
```

#### 2. `.gitignore` - Ajouter l'exclusion des logs
**Ajouter à la fin du fichier :**
```gitignore
# Hexdump logs (not tracked in git)
logs/
```

### Structure des Fichiers Générés
```
/Users/k/_PLAYGROUND_/VelocePrinter/
└── logs/
    └── hexdumps/
        ├── escpos_2025-12-04T14-30-45-123Z_192-168-1-100_9100.txt
        ├── escpos_2025-12-04T14-31-12-456Z_192-168-1-100_9100.txt
        └── ...
```

### Format Hexdump (Exemple)
```
================================================================================
CAPTURE TIMESTAMP: 2025-12-04T14:30:45.123Z
CLIENT: 192.168.1.100:9100
DATA SIZE: 256 bytes
================================================================================

00000000  1b 40 1b 61 01 56 65 6c  6f 63 65 20 50 4f 53 0a  |.@.a.Veloce POS.|
00000010  1b 61 00 46 41 43 54 55  52 45 20 23 31 32 33 34  |.a.FACTURE #1234|
00000020  35 0a 0a 1b 21 00 41 72  74 69 63 6c 65 20 31 20  |5...!.Article 1 |
00000030  20 20 20 20 20 20 20 20  20 31 20 78 20 31 30 2e  |         1 x 10.|
...

================================================================================
RAW BASE64 (for re-processing):
G0AbaQFWZWxvY2UgUE9TChtAERhYQTFURVVSRSAjMTIzNDUKChshAEFydGljbGUgMSAg...
================================================================================
```

### Tests à Effectuer
1. ✅ Démarrer l'application sur port 9100
2. ✅ Envoyer des données de test ESC/POS (via nc ou telnet)
3. ✅ Vérifier que `/logs/hexdumps/` contient les fichiers
4. ✅ Vérifier que l'affichage UI fonctionne toujours
5. ✅ Tester avec Véloce POS réel
6. ✅ Collecter **10-20 factures** pour analyse de structure

### Commandes de Test (sans Véloce)
```bash
# Test simple ESC/POS
echo -ne '\x1b\x40\x1b\x61\x01TEST FACTURE\x0a\x1b\x64\x03' | nc localhost 9100

# Test avec facture simulée
echo -ne '\x1b\x40\x1b\x61\x01VELOCE POS\x0a\x1b\x61\x00FACTURE #12345\x0a\x0a\x1b\x21\x00Article 1         1 x 10.00\x0a\x1b\x64\x03' | nc localhost 9100
```

---

## Phase 2 : Documentation - README.md en Français

### Objectif
Traduire et enrichir le README en français avec les détails de l'intégration Véloce.

### Fichier à Modifier
`/README.md`

### Contenu Cible

```markdown
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
- **npm** ou **yarn**
- **Git**

### Installation
```bash
git clone https://github.com/KimBelanger/VelocePrinter.git
cd VelocePrinter
npm install
```

### Commandes
```bash
npm start       # Mode développement avec logs
npm run package # Package pour OS courant
npm run make    # Générer binaires multi-OS
```

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

- [Guide Véloce POS](https://veloce.ca) (système de point de vente)
- [Protocole ESC/POS](https://en.wikipedia.org/wiki/ESC/P) (Wikipedia)
- [RFC Imprimantes Réseau](https://www.ietf.org/rfc/rfc1179.txt) (LPD/LPR)
```

---

## Résumé des Modifications

### Fichiers à Créer
1. ✅ `_PLAN.md` - Ce document (racine projet, **tracké git**)
2. ✅ `/logs/hexdumps/` - Répertoire auto-créé par code (**non tracké**)

### Fichiers à Modifier
1. ✅ `/ZplEscPrinter/js/main.js` :
   - Ajouter imports `fs`, `path`
   - Ajouter fonction `captureHexdump()`
   - Modifier ligne 369 (appel avant `escpos()`)

2. ✅ `.gitignore` :
   - Ajouter `logs/` (déjà fait : `claude.md` aussi)

3. ✅ `README.md` :
   - Version française complète
   - Focus ESC/POS uniquement
   - Détails intégration Véloce
   - Exemples configuration

### Validation Avant Merge
- [ ] Code compile sans erreur
- [ ] Tests manuels avec `nc localhost 9100`
- [ ] Tests réels avec Véloce POS
- [ ] Hexdumps générés correctement
- [ ] UI inchangée (affichage factures OK)
- [ ] Pas de timeout Véloce

### Prochaines Étapes (Après Phase 1)
1. Collecter 10-20 hexdumps de factures réelles Véloce
2. Analyser la structure des données
3. Identifier les patterns (articles, prix, totaux)
4. Phase 2 : Parser + MariaDB
