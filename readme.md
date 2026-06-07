# 🔓 Lab — Bypass Root Detection avec Frida & Objection

Salma AIT ZIDAN
> **Contexte pédagogique** — Ce lab est réalisé dans un cadre légal et éducatif, sur des appareils et applications dont vous êtes propriétaire ou pour lesquels vous avez une autorisation explicite. Ne pas utiliser ces techniques sur des systèmes tiers sans autorisation.

---

## 📋 Prérequis

| Élément | Détail |
|---|---|
| OS | Windows / macOS / Linux avec droits admin/sudo |
| Python | 3.8+ avec `pip` |
| ADB | [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools) |
| Appareil | Android 8.0+ — Options développeur + Débogage USB activés |
| Frida | Version alignée PC ↔ `frida-server` Android |
| App cible | Ex. : `com.example.rootcheck` (RootBeer Sample, DevAdvance RootChecker…) |

---

## Étape 1 — Préparer l'environnement

### 1.1 Installer Python, Frida et les outils CLI

```bash
# Vérifications préalables
PS C:\Users\HP> python --version
Python 3.11.0
PS C:\Users\HP> pip --version
pip 22.3 from C:\Users\HP\AppData\Local\Programs\Python\Python311\Lib\site-packages\pip (python 3.11)
<img width="749" height="68" alt="image" src="https://github.com/user-attachments/assets/f4867b14-6531-4478-9cc1-67ae32527c07" />



# Vérifier
PS C:\Users\HP> frida --version
16.6.6
PS C:\Users\HP> python -c "import frida; print(frida.__version__)"
16.6.6

<img width="500" height="61" alt="image" src="https://github.com/user-attachments/assets/0b6467e8-8841-4480-a0bd-34d65f0c008b" />

```


### 1.2 Installer ADB (Android Platform Tools)

1. Téléchargez depuis [developer.android.com/tools/releases/platform-tools](https://developer.android.com/tools/releases/platform-tools).
2. Ouvrez un terminal dans le dossier `platform-tools` (ou ajoutez-le au PATH).
3. Branchez le téléphone en USB, activez le Débogage USB, puis acceptez l'empreinte.

```bash
PS C:\Users\HP> adb version
Android Debug Bridge version 1.0.41
Version 37.0.0-14910828
Installed as C:\Users\HP\AppData\Local\Microsoft\WinGet\Packages\Google.PlatformTools_Microsoft.Winget.Source_8wekyb3d8bbwe\platform-tools\adb.exe
Running on Windows 10.0.26200
PS C:\Users\HP> adb devices
List of devices attached
emulator-5554   device

<img width="865" height="132" alt="image" src="https://github.com/user-attachments/assets/ddb73ea1-8428-46ec-b441-1c3204635630" />

```
---

## Étape 2 — Démarrer frida-server sur l'appareil

### 2.1 Identifier l'architecture CPU

```bash

PS C:\Users\HP> adb shell getprop ro.product.cpu.abi
x86_64
```

### 2.2 Télécharger frida-server

> ⚠️ La version de `frida-server` **doit correspondre exactement** à `frida --version` côté PC.
<img width="195" height="29" alt="image" src="https://github.com/user-attachments/assets/d5c4199c-5b66-4d5d-b98c-adcf75e457c8" />


### 2.3 Pousser et lancer frida-server

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server

# Lancement (premier terminal, laisser tourner)
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"

<img width="490" height="96" alt="image" src="https://github.com/user-attachments/assets/659950f0-815c-4691-a439-510a47ff83b5" />

```

### 2.4 Forward de ports (si nécessaire)

```bash
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

### 2.5 Valider la connexion

```bash
# Doit lister les apps/processus de l'appareil

PS C:\Users\HP> frida-ps -Uai
  PID  Name                  Identifier
-----  --------------------  ---------------------------------------
10841  Chrome                com.android.chrome
10436  Gmail                 com.google.android.gm
 2223  Google                com.google.android.googlequicksearchbox
 2223  Google                com.google.android.googlequicksearchbox
 2693  Messages              com.google.android.apps.messaging

<img width="530" height="212" alt="image" src="https://github.com/user-attachments/assets/9889cdd7-2dd8-45cb-9fd3-d9d3527723c5" />

```

---

## Étape 3 — Frida : bases en 10 minutes

| Option | Signification |
|---|---|
| `-U` | Cible un appareil USB |
| `-f <package>` | **Spawn** — démarre l'app et injecte dès le début |
| `-n <ProcessName>` | **Attach** — s'attache à un process déjà lancé |
| `Java.perform(...)` | Attend que la VM Java soit prête avant d'installer les hooks |

**Test minimal — créez `hello.js` :**

```javascript
Java.perform(function () {
  console.log("[+] Script injecté : Java.perform OK");
});
```

```bash
PS C:\Users\HP> frida -U -f com.example.projetws -l hello.js

<img width="605" height="203" alt="image" src="https://github.com/user-attachments/assets/b911ea0a-8c89-4955-a0b2-dc5763ec2d84" />

```

---

## Étape 4 — Bypass de la détection de root avec Frida

### Vecteurs de détection Java courants

- Lecture de `android.os.Build.TAGS` (cherche `test-keys`)
- `java.io.File.exists()` sur `/system/xbin/su`, `busybox`, etc.
- `Runtime.getRuntime().exec("su")` / `which su`
- Bibliothèques tierces : `RootBeer.isRooted()`

### 4.1 Script Java — `bypass_root_basic.js`

```

**Exécution :**

```bash
PS C:\Users\HP> frida -U -f com.example.projetws -l bypass_root1.js

<img width="629" height="225" alt="image" src="https://github.com/user-attachments/assets/bb3ed995-59d8-4822-9eb1-811f460fe1c0" />

```
### 4.2 Checks natifs C/C++ — `bypass_native.js`

Les apps peuvent chercher `su` via `open`/`openat`/`access`/`stat` en natif. Ce script les bloque :


**Exécution combinée (Java + natif) :**

```bash
PS C:\Users\HP> frida -U -f com.example.projetws -l bypass_native.js

<img width="589" height="284" alt="image" src="https://github.com/user-attachments/assets/8e98006d-868a-4cd4-bccf-d57237d68579" />

```

---

## Étape 5 — Bypass simplifié avec Objection

Objection est une surcouche à Frida avec des commandes prêtes à l'emploi.

### 5.1 Installer Objection

```bash
# Méthode recommandée (isolation via pipx)
pip install --user pipx
pipx ensurepath
pipx install objection

# Ou via pip classique
pip install --upgrade objection

# Vérification
PS C:\Users\HP> objection version
objection: 1.12.5
<img width="281" height="34" alt="image" src="https://github.com/user-attachments/assets/e97d2346-33c3-484b-be55-84d758b9c393" />

```

### 5.2 Lancer le bypass

**Spawn (recommandé — hooks appliqués dès le démarrage) :**

```bash
PS C:\Users\HP> objection -g com.example.projetws explore --startup-command "android root disable"
```



### Permissions Android

Sur appareil non rooté, vous pouvez hooker les apps utilisateur mais pas certains processus système.

---

## ✅ Check-list rapide

- [ ] `python` et `pip` OK — version notée
- [ ] `frida` installé — version notée
- [ ] `adb devices` → état `device`
- [ ] `frida-server` lancé — `frida-ps -Uai` liste les apps
- [ ] `hello.js` s'injecte sans erreur
- [ ] `bypass_root_basic.js` neutralise les checks Java
- [ ] `bypass_native.js` neutralise les checks natifs (vus via `frida-trace`)
- [ ] Objection : `android root disable` fonctionne sur l'app cible
- [ ] (Optionnel) Magisk : Zygisk/DenyList configurés pour masquage système

---


