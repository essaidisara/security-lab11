# Rapport de Lab — Contournement de la Detection de Root sur Android via Frida

**Module : Securite Mobile**  
**Cible : OWASP UnCrackable Level 1** (`owasp.mstg.uncrackable1`)  


---

## Contexte et objectifs

Ce rapport documente la mise en oeuvre de techniques de bypass de detection de root sur une application Android volontairement vulnerable. L'objectif est de comprendre les mecanismes de protection Java et natifs, puis de les neutraliser a l'aide de l'outil d'instrumentation dynamique Frida.



---





## Livrable 1 — Verification de l'installation 

Les trois commandes suivantes confirment que l'environnement est correctement configure :

```bash
frida --version
python -c "import frida; print(frida.__version__)"
adb devices
```

> **Sortie terminal :**
<img width="479" height="148" alt="image" src="https://github.com/user-attachments/assets/61a30a9c-2cc2-40db-9a70-1ca646e4e613" />



> ```

---

## Livrable 2 — Demarrage de frida-server et visibilite 

### Deploiement sur l'emulateur

```bash
adb push "frida-server-17.9.1-android-x86_64" /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

> **Sortie terminal :**
> ```
<img width="460" height="140" alt="image" src="https://github.com/user-attachments/assets/52443997-8703-45d6-9ce1-8a3c6ec661ee" />

> ```

### Confirmation de la connexion Frida

```bash
frida-ps -Uai
```

> **Liste des applications detectees  :**
> ```
<img width="477" height="444" alt="image" src="https://github.com/user-attachments/assets/3496dbda-f75c-4ce3-aeac-5eb4d132e19d" />

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

## Livrable 3 — Bypass Java 

### Etat initial — Detection active

> **Capture d'ecran montrant le popup "Root detected!" :**  
<img width="194" height="344" alt="image" src="https://github.com/user-attachments/assets/a3040f82-c5be-475e-8680-020156eca5c6" />


L'application affiche une alerte et se ferme sans permettre aucune interaction.

---

### Approche A — Scripts de bypass generiques

La commande frida -U -f owasp.mstg.uncrackable1 -l hook_abc.js -l bypass_root.js permet de lancer l’application cible sur l’émulateur et d’injecter des scripts Frida afin de contourner les mécanismes de détection de root.

L’exécution montre que l’application est correctement attachée par Frida (Spawned… Resuming main thread), ce qui confirme le bon fonctionnement de l’environnement.

Les journaux indiquent que les méthodes a(), b() et c() de la classe sg.vantagepoint.a.c, identifiées comme responsables de la détection, ont été interceptées. Elles sont modifiées pour retourner systématiquement false, ce qui désactive la détection du root.
**Commande d'execution :**

```bash
frida -U -f owasp.mstg.uncrackable1 -l bypass_root.js -l bypass_native.js -l anti_frida.js
```
<img width="1276" height="490" alt="image" src="https://github.com/user-attachments/assets/09dd707b-bae4-469d-8fe7-323032c3e61c" />
```




---

### Approche B — Hooks cibles sur les methodes obfusquees

Cette seconde approche est plus precise : elle cible directement les methodes de detection identifiees par reverse engineering.

**Etape 1 — Identification des methodes**

```bash
frida -U -f owasp.mstg.uncrackable1 -l enumerate_methods.js
```

> **Sortie d'enumeration :**
> ```
<img width="589" height="215" alt="image" src="https://github.com/user-attachments/assets/35e145a6-13fc-4130-a83b-134218da5482" />


> ```

Les trois méthodes retournant des booléens correspondent aux mécanismes de détection implémentés par l’application. Elles ont été modifiées afin de renvoyer systématiquement false, ce qui permet de désactiver ces vérifications.

**Etape 2 — Injection des hooks**

```bash
frida -U -f owasp.mstg.uncrackable1 -l hook_abc.js -l bypass_root.js
```
<img width="605" height="246" alt="image" src="https://github.com/user-attachments/assets/e76cdf1f-ccc6-4c84-986c-8ce79f9fe388" />



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

