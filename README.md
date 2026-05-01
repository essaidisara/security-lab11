# Lab Frida — Root Detection Bypass
## Rapport de livrables

**Nom :**essaidi 
**Date :** ___________________________  
**Appareil Android :** ___________________________  
**Version Android :** ___________________________

---

## 1. Installation et preuve (20 pts)

### 1.1 Version de Frida (PC)

```bash
 frida --version
```

> **Sortie obtenue :**
> ```
<img width="473" height="55" alt="image" src="https://github.com/user-attachments/assets/5a1546ed-4bbb-404f-8511-c288a447cb58" />

> ```

---

### 1.2 Version du module Python Frida

```bash
python -c "import frida; print(frida.__version__)"
```

> **Sortie obtenue :**
> ```
<img width="466" height="45" alt="image" src="https://github.com/user-attachments/assets/2359328c-eebb-4f94-bbe7-cb1bddfab973" />

> ```

---

### 1.3 Appareils ADB détectés

```bash
 adb devices
```

> **Sortie obtenue :**
> ```
 <img width="812" height="95" alt="image" src="https://github.com/user-attachments/assets/f18a14b6-3b19-4faa-b662-4c5438b751b7" />

> ```

`

---

## 2. Déploiement et visibilité 

### 2.1 Lancement de frida-server sur l'appareil

Commandes exécutées :

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

> **Capture d'écran ou sortie terminal :**
>
<img width="944" height="188" alt="image" src="https://github.com/user-attachments/assets/9f97cc02-5df0-4152-a89a-95722a0114b6" />

---

### 2.2 Listing des apps avec frida-ps

```bash
 frida-ps -Uai
```

>
<img width="686" height="306" alt="image" src="https://github.com/user-attachments/assets/54c5b5fa-a9a8-4d97-bac8-6b99a3165e5e" />

> ```

>  frida-server opérationnel

---



## 3. Bypass Java (30 pts)

### 3.1 Application cible

| Champ | Valeur |
|---|---|
| Nom de l'app | UnCrackable Level 1 (OWASP MSTG) |
| Package | `owasp.mstg.uncrackable1` |
| Script utilisé | `bypass_root.js` |

### 3.2 Avant le bypass — Root détecté

> **Capture d'écran de l'app affichant "Root detected" :**
>
> `[INSÉRER CAPTURE AVANT ICI]`

---

### 3.3 Commande de lancement Frida

```bash
frida -U -f com.scottyab.rootbeer.sample -l bypass_root.js --no-pause
```

---

### 3.4 Logs Frida observés dans le terminal

```
[+] Hook Build.TAGS -> release-keys
[+] RootBeer.isRooted -> false
[+] File.exists bypass for /system/xbin/su
[+] File.exists bypass for /system/bin/su
[+] Hooks Runtime.exec installés
[+] Java layer bypass installed
[COLLER LES LOGS RÉELS ICI]
```

---

### 3.5 Après le bypass — Root non détecté

> **Capture d'écran de l'app affichant "Not rooted" :**
>
 <img width="254" height="380" alt="image" src="https://github.com/user-attachments/assets/a0187000-7cb5-4963-82e7-e57b4ef032cb" />



---

## 4. Natif / Trace (20 pts)

### 4.1 Identification des appels natifs avec frida-trace

```bash
frida-trace -U -i open -i access -i stat -i openat -i fopen -i readlink com.scottyab.rootbeer.sample
```

> **Sortie frida-trace (au moins 2 appels natifs liés au root check) :**
> ```
> [COLLER LA SORTIE frida-trace ICI]
> Exemple attendu :
>   open("/system/xbin/su", ...)
>   access("/system/bin/busybox", ...)
> ```

---

### 4.2 Appels natifs identifiés

| # | Fonction | Chemin consulté |
|---|---|---|
| 1 | `open` / `openat` | `/system/xbin/su` |
| 2 | `access` | `/system/bin/busybox` |
| ... | ... | ... |

---

### 4.3 Adaptation de bypass_native.js

Chemins ajoutés à la liste `SUS` si nécessaire :

```javascript
const SUS = [
  '/system/bin/su', '/system/xbin/su', '/sbin/su', '/system/su',
  '/system/bin/busybox', '/system/xbin/busybox',
  // [AJOUTER ICI LES CHEMINS DÉCOUVERTS VIA frida-trace]
];
```

---

### 4.4 Lancement combiné

```bash
frida -U -f com.scottyab.rootbeer.sample -l bypass_root.js -l bypass_native.js --no-pause
```

---

### 4.5 Logs [+] Blocked observés

```
[+] Hooked open
[+] Hooked openat
[+] Hooked access
[+] Hooked stat
[+] Hooked lstat
[+] Blocked open on /system/xbin/su
[+] Blocked access on /system/bin/busybox
[COLLER LES LOGS RÉELS ICI]
```

>  Au moins 2 appels natifs bloqués et loggués avec `[+] Blocked ...`.

---


---
