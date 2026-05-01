# Rapport de Lab — Contournement de la Detection de Root sur Android via Frida

**Module : Securite Mobile**  
**Cible : OWASP UnCrackable Level 1** (`owasp.mstg.uncrackable1`)  


---

## Contexte et objectifs

Ce rapport documente la mise en oeuvre de techniques de bypass de detection de root sur une application Android volontairement vulnerable. L'objectif est de comprendre les mecanismes de protection Java et natifs, puis de les neutraliser a l'aide de l'outil d'instrumentation dynamique Frida.

> Ce travail est realise dans un cadre strictement pedagogique, sur une application de test prevue a cet effet.

---





## Livrable 1 — Verification de l'installation (20 pts)

Les trois commandes suivantes confirment que l'environnement est correctement configure :

```bash
frida --version
python -c "import frida; print(frida.__version__)"
adb devices
```

> **Sortie terminal :**
<img width="934" height="107" alt="image" src="https://github.com/user-attachments/assets/968683dc-92df-4d6b-aaf9-a16e09c51827" />


> ```

---

## Livrable 2 — Demarrage de frida-server et visibilite (30 pts)

### Deploiement sur l'emulateur

```bash
adb push "frida-server-17.9.1-android-x86_64" /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

> **Sortie terminal :**
> ```
<img width="476" height="130" alt="image" src="https://github.com/user-attachments/assets/81abc3e8-5656-43f8-a854-a6a6a6bdb035" />

> ```

### Confirmation de la connexion Frida

```bash
frida-ps -Uai
```

> **Liste des applications detectees  :**
> ```
<img width="476" height="409" alt="image" src="https://github.com/user-attachments/assets/54551b2e-480e-4b2b-bceb-6461433f3a28" />

> ```

---

## Comment l'application detecte-t-elle le root ?

Avant de contourner les protections, il est utile de comprendre ce qu'elles font.

### Protections implementees en Java

L'application utilise plusieurs techniques classiques au niveau de la JVM :

- **Verification de Build.TAGS** : sur un appareil rooté, cette propriete contient `test-keys` au lieu de `release-keys`.
- **Verification de fichiers** : la methode `File.exists()` est appelee sur des chemins comme `/system/xbin/su` ou `/system/bin/busybox`.
- **Execution de commandes shell** : `Runtime.exec()` est utilise pour tenter d'executer `su` ou `which su`.

### Protections au niveau natif (C/C++)

- Appels systeme `open`, `openat`, `access`, `stat`, `lstat` sur des chemins suspects.
- Lecture de `/proc/mounts` pour detecter des partitions montees en ecriture.
- Detection de Frida par scan des ports 27042/27043.

---

## Livrable 3 — Bypass Java (30 pts)

### Etat initial — Detection active

> **Capture d'ecran montrant le popup "Root detected!" :**  
<img width="194" height="344" alt="image" src="https://github.com/user-attachments/assets/a3040f82-c5be-475e-8680-020156eca5c6" />


L'application affiche une alerte et se ferme sans permettre aucune interaction.

---

### Approche A — Scripts de bypass generiques

Cette premiere approche utilise des hooks Java larges pour couvrir les verifications les plus courantes, sans connaissance prealable du code de l'application.

**Fichiers utilises :**



 bypass_root.js : script Frida utilisé pour contourner les mécanismes de détection du root côté Java. Il modifie ou intercepte plusieurs contrôles courants, comme la valeur de Build.TAGS, les vérifications de fichiers sensibles avec File.exists, l’exécution de commandes système via Runtime.exec, ainsi que les appels à AlertDialog et System.exit afin d’empêcher l’application d’afficher une alerte ou de se fermer automatiquement.
bypass_native.js : script destiné à intercepter les appels natifs effectués par l’application au niveau système. Il surveille et manipule des fonctions comme open, openat, access, stat et lstat, souvent utilisées pour vérifier l’existence de fichiers liés au root, à Magisk, à BusyBox ou à d’autres éléments suspects.
anti_frida.js : script permettant de réduire les traces visibles de Frida pendant l’analyse dynamique. Il vise à masquer certains indicateurs utilisés par les applications pour détecter Frida, comme les variables d’environnement, les ports de communication, ou certains noms/processus associés à l’instrumentation.
**Commande d'execution :**

```bash
frida -U -f owasp.mstg.uncrackable1 -l bypass_root.js -l bypass_native.js -l anti_frida.js
```

**Journaux Frida observes :**

```
```
<img width="1491" height="1055" alt="image" src="https://github.com/user-attachments/assets/4f389f4d-0ff9-4a90-857f-0c1c5eb5e918" />

---

### Approche B — Hooks cibles sur les methodes obfusquees

Cette seconde approche est plus precise : elle cible directement les methodes de detection identifiees par reverse engineering.

**Etape 1 — Identification des methodes**

```bash
frida -U -f owasp.mstg.uncrackable1 -l enumerate_methods.js
```

> **Sortie d'enumeration :**
> ```
<img width="527" height="324" alt="image" src="https://github.com/user-attachments/assets/9dd436fa-a099-4674-a32d-8f5852625310" />

> ```

Les trois méthodes retournant des booléens correspondent aux mécanismes de détection implémentés par l’application. Elles ont été modifiées afin de renvoyer systématiquement false, ce qui permet de désactiver ces vérifications.

**Etape 2 — Injection des hooks**

```bash
frida -U -f owasp.mstg.uncrackable1 -l hook_abc.js -l bypass_root.js
```
<img width="436" height="277" alt="image" src="https://github.com/user-attachments/assets/b160b082-747c-4daf-bd0a-646497732a62" />


**resultat:**

```
désactivé la détection root
empêché les crashs
bloqué les alertes
trompé l’application sur l’état du système)
```

### Etat final — Bypass reussi

> **Capture d'ecran de l'application fonctionnelle :**  
<img width="153" height="326" alt="image" src="https://github.com/user-attachments/assets/8d14ba59-6d73-41d2-86f5-46b43939027f" />

Le rapport présente un laboratoire de sécurité mobile sur OWASP UnCrackable Level 1, visant à contourner la détection de root avec Frida. L’environnement est d’abord vérifié avec frida, python-frida et adb, puis frida-server est déployé sur l’émulateur Android et la connexion est confirmée avec frida-ps -Uai.

L’application détecte le root via plusieurs mécanismes : vérification de Build.TAGS, recherche de fichiers comme su ou busybox, exécution de commandes shell, appels natifs comme open, access, stat, et détection possible de Frida via les ports standards.

Deux approches de bypass sont utilisées. La première repose sur des scripts génériques (bypass_root.js, bypass_native.js, anti_frida.js) pour neutraliser les contrôles Java, natifs et les traces de Frida. La seconde approche est plus ciblée : elle identifie les méthodes obfusquées sg.vantagepoint.a.c.a(), b() et c(), puis les force à retourner false.

Au final, la détection root est désactivée, les alertes sont bloquées, les fermetures forcées sont empêchées, et l’application devient fonctionnelle malgré l’environnement rooté.
---

