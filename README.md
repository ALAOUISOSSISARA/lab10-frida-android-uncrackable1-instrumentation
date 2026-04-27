# LAB 10 — Guide d'installation et d'utilisation de Frida
**Cours : Sécurité des applications mobiles**

---

## Environnement de travail

| Élément | Détail |
|---|---|
| OS | Windows 10 (PowerShell) |
| Python | 3.12.10 |
| pip | 25.0.1 |
| Frida | 17.9.1 |
| ADB | 1.0.41 (version 37.0.0) |
| Émulateur | Android Emulator 5554 (x86_64) |
| Application cible | `owasp.mstg.uncrackable1` |

---

## Étape 1 — Installation du client Frida

### 1.1 Vérification de Python et pip

```powershell
python --version
# Python 3.12.10

pip --version
# pip 25.0.1
```

### 1.2 Installation de frida et frida-tools

```powershell
pip install --upgrade frida frida-tools
```

### 1.3 Vérifications

```powershell
frida --version
# 17.9.1

python -c "import frida; print(frida.__version__)"
# 17.9.1
```




---

## Étape 2 — Installation des outils Android (ADB)

```powershell
adb version
# Android Debug Bridge version 1.0.41

adb devices
# List of devices attached
# emulator-5554   device
```


---

## Étape 3 — Déploiement de frida-server sur l'émulateur Android

### 3.1 Identification de l'architecture CPU

```powershell
adb shell getprop ro.product.cpu.abi
# x86_64
```

### 3.2 Téléchargement de frida-server

Fichier téléchargé : `frida-server-17.9.1-android-x86_64.xz`  
Source : https://github.com/frida/frida/releases

### 3.3 Décompression

Le fichier `.xz` a été extrait avec 7-Zip sous Windows.

### 3.4 Copie vers l'émulateur

```powershell
adb push "F:\frida-server-17.9.1-android-x86_64 (1)" /data/local/tmp/frida-server
# 110787848 bytes in 2.668s — 39.6 MB/s
```

### 3.5 Permissions d'exécution

```powershell
adb shell chmod 755 /data/local/tmp/frida-server
```

### 3.6 Lancement en arrière-plan

```powershell
adb root
adb shell "/data/local/tmp/frida-server &"
```

### 3.7 Vérification du processus

```powershell
adb shell ps | findstr frida
# root  2834  1  frida-server
```

### 3.8 Redirection des ports

```powershell
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

> **[SCREENSHOT 3]** — Sortie de `adb shell ps | findstr frida` confirmant frida-server actif

---

## Étape 4 — Test de connexion depuis le PC

```powershell
frida-ps -U
```

Résultat : liste complète des processus Android de l'émulateur affichée avec succès.

```powershell
frida-ps -Uai
```

Résultat : liste des applications installées avec leurs packages.

> **[SCREENSHOT 4]** — Sortie de `frida-ps -U` et `frida-ps -Uai`

---

## Étape 5 — Injection minimale sur UnCrackable1

### 5.1 Installation de l'APK

```powershell
adb install UnCrackable-Level1.apk
# Success
```

### 5.2 Création de hello.js

```javascript
Java.perform(function () {
  console.log("[+] Frida Java.perform OK");

  var RootDetection = Java.use("sg.vantagepoint.a.c");
  RootDetection.a.implementation = function() { return false; };
  RootDetection.b.implementation = function() { return false; };
  RootDetection.c.implementation = function() { return false; };

  var System = Java.use("java.lang.System");
  System.exit.implementation = function(code) {
    console.log("[+] System.exit bloque !");
  };

  console.log("[+] Bypass root OK");
});
```

### 5.3 Problème rencontré et résolution

**Erreur initiale :**
```
Failed to spawn: java.lang.reflect.InvocationTargetException
```
**Cause :** UnCrackable1 détecte l'environnement root/émulateur et se ferme.  
**Solution :** Ajout du bypass de détection root dans le script.

**Deuxième erreur :**
```
Failed to load script: 'utf-8' codec can't decode byte 0xe9
```
**Cause :** Caractères accentués français dans les commentaires.  
**Solution :** Recréation du fichier avec `-Encoding UTF8` et suppression des accents.

### 5.4 Résultat final

```
Spawned `owasp.mstg.uncrackable1`. Resuming main thread!
[+] Frida Java.perform OK
[+] Bypass root OK
```

> **[SCREENSHOT 5]** — Console Frida affichant le bypass root et Java.perform OK

---

## Étape 6 — Exploration de la console interactive Frida

Session lancée avec :
```powershell
frida -U -f owasp.mstg.uncrackable1 -l F:\hello.js
```

### Commandes exécutées et résultats

| Commande | Résultat |
|---|---|
| `Process.arch` | `"x64"` |
| `Java.available` | `true` |
| `Process.id` | `3053` |
| `Process.platform` | `"linux"` |

### Fonctions natives localisées dans libc.so

| Fonction | Adresse mémoire |
|---|---|
| `connect` | `0x7dc4516e5ef0` |
| `send` | `0x7dc4516f4630` |
| `recv` | `0x7dc4516f3d30` |
| `open` | `0x7dc4516f1b40` |
| `read` | `0x7dc45173aec0` |

Toutes ces fonctions sont interceptables via `Interceptor.attach()`.

> **[SCREENSHOT 6]** — Résultats des commandes Process dans la console Frida

---

## Étape 7 — Observation des bibliothèques de chiffrement et appels réseau

### 7.1 Bibliothèques de chiffrement détectées

```javascript
Process.enumerateModules().filter(m =>
  m.name.indexOf("ssl") !== -1 ||
  m.name.indexOf("crypto") !== -1 ||
  m.name.indexOf("boring") !== -1
)
```

| Bibliothèque | Chemin | Rôle |
|---|---|---|
| `libcrypto.so` | `/system/lib64/` | Cryptographie système |
| `libjavacrypto.so` | `/apex/com.android.conscrypt/lib64/` | Cryptographie Java |
| `libcrypto.so` | `/apex/com.android.conscrypt/lib64/` | Cryptographie Conscrypt |
| `libssl.so` | `/apex/com.android.conscrypt/lib64/` | TLS/SSL |

**Conclusion :** L'application utilise **Conscrypt** (implémentation Android de SSL/TLS) avec cryptographie native active.

> **[SCREENSHOT 7]** — Résultat du filtre sur les modules crypto/ssl

---

## Étape 8 — Hooks Java : SharedPreferences, SQLite, Debug, Runtime, File

### Scripts créés et résultats

#### hook_prefs.js — SharedPreferences
```
[+] Hook SharedPreferences charge
```
**Résultat :** Aucune lecture/écriture de SharedPreferences détectée.  
**Interprétation :** UnCrackable1 ne stocke pas de configuration locale.

#### hook_sqlite.js — SQLite
```
[+] Hook SQLite charge
```
**Résultat :** Aucune requête SQL détectée.  
**Interprétation :** L'application n'utilise pas de base de données SQLite.

#### hook_debug.js — Détection débogueur
```
[+] Hook Debug charge
```
**Résultat :** Aucun appel à `isDebuggerConnected()` ou `waitingForDebugger()`.  
**Interprétation :** La protection anti-debug utilise sa propre détection root native.

#### hook_runtime.js — Commandes système
```
[+] Hook Runtime.exec charge
```
**Résultat :** Aucune commande système exécutée.  
**Interprétation :** Pas d'interaction avec le shell Android.

#### hook_file_java.js — Accès fichiers
```
[+] Hook File charge
[File] nouveau chemin : /system/etc/security/cacerts
```
**Résultat :** Accès au répertoire des certificats CA système.  
**Interprétation :** L'application initialise une vérification des certificats SSL/TLS au démarrage.

> **[SCREENSHOT 8]** — Résultat du hook_file_java.js montrant l'accès à cacerts

---

## Exercice 4 — Simulation d'erreur et diagnostic

### Simulation
```powershell
adb shell "pkill frida-server"
frida-ps -U
# Failed to enumerate processes: unable to connect to remote frida-server
```

### Diagnostic
```powershell
adb shell ps | findstr frida
# (aucun résultat)
```
**Cause identifiée :** frida-server n'est plus actif.

### Correction
```powershell
adb root
adb shell "/data/local/tmp/frida-server &"
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
adb shell ps | findstr frida
# root  frida-server  — actif
```

---

## Conclusion

Ce lab a permis de :

- Installer et configurer **Frida 17.9.1** sur Windows avec Python 3.12
- Déployer **frida-server** sur un émulateur Android x86_64
- Réaliser une **injection JavaScript** dans une application Android avec bypass de détection root
- Explorer la **console interactive Frida** pour inspecter l'architecture, les modules et la mémoire du processus
- Identifier les **bibliothèques de chiffrement** (libssl.so, libcrypto.so, Conscrypt)
- Localiser les **fonctions réseau natives** (connect, send, recv, open, read) dans libc.so
- Créer et exécuter des **scripts de hook Java** ciblant SharedPreferences, SQLite, Debug, Runtime et File
- Diagnostiquer et corriger une **erreur de connexion** à frida-server
