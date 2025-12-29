# LFI Exploitation (Path Traversal) -  Write-up

## 🎯 Objectif
Ce dépôt documente l'exploitation d'une vulnérabilité de type **Local File Inclusion (LFI)**. L'objectif est d'identifier un paramètre vulnérable dans l'URL pour lire des fichiers sensibles du serveur (comme `/etc/passwd`) et récupérer le flag final.

---

## 📚 Note Théorique : LFI vs RCE
Avant l'exploitation, il est crucial de distinguer ces deux vulnérabilités souvent confondues :

* **LFI (Local File Inclusion) :**
    * **Définition :** Permet d'inclure et de **lire** des fichiers déjà présents sur le serveur.
    * **Mécanisme :** L'application web charge un fichier local basé sur l'entrée utilisateur sans validation suffisante.
    * **Impact :** Accès à des fichiers sensibles (fichiers de configuration, sources, `/etc/passwd`). Le code n'est pas exécuté, il est affiché (sauf si l'on arrive à inclure un fichier contenant du code PHP par exemple, ce qui mène à une RCE).
    * *En résumé : LFI = Lire le contenu.*

* **RCE (Remote Code Execution) :**
    * **Définition :** Permet à l'attaquant d'exécuter des **commandes système** arbitraires.
    * **Mécanisme :** L'attaquant interagit directement avec le shell du serveur.
    * **Impact :** Compromission totale (création d'utilisateurs, reverse shell, suppression de fichiers).
    * *En résumé : RCE = Exécuter des commandes.*

---

## 🔍 1. Reconnaissance & Analyse

### Scan Nmap & Navigation
Nous commençons par un scan de ports classique qui révèle un serveur web (Port 80).
En naviguant sur l'adresse IP, nous arrivons sur une page web basique.

### Analyse du Code Source
Pour comprendre comment l'application fonctionne, nous inspectons le code source de la page (CTRL+U).
Nous remarquons que le site utilise des requêtes **GET** pour charger le contenu des pages via un paramètre URL (souvent `?page=` ou `?file=`).

Cela indique que le serveur prend notre entrée et va chercher un fichier correspondant. C'est un vecteur potentiel de **LFI**.

---

## 🔓 2. Exploitation : Path Traversal

### Test de la vulnérabilité (/etc/passwd)
Nous tentons une attaque par traversée de répertoires (**Path Traversal**). L'idée est d'utiliser `../` pour remonter dans l'arborescence des dossiers jusqu'à la racine (`/`), puis de redescendre vers un fichier que nous savons présent sur tout système Linux : `/etc/passwd`.

**Payload utilisé :**
```text
[http://10.10.138.45/?page=../../../../../etc/passwd](http://10.10.138.45/?page=../../../../../etc/passwd)
```
**Analyse du Payload :**
* `../` : Permet de remonter d'un cran dans l'arborescence des dossiers. On le répète plusieurs fois pour être certain d'atteindre la racine du système (`/`).
* `/etc/passwd` : Fichier standard sous Linux contenant la liste des utilisateurs du système.

**Résultat :**
Le serveur affiche le contenu du fichier `/etc/passwd`. La vulnérabilité **LFI** est ainsi confirmée.
## 🚩 3. Récupération du Flag

Maintenant que nous pouvons lire des fichiers, nous devons trouver le fichier contenant le flag.

### Méthodologie de recherche
Nous essayons d'abord les emplacements standards pour les challenges CTF :
1.  `/root/root.txt` : `../../../../../root/root.txt` (Échec - Permission refusée ou fichier inexistant).
2.  Dossiers utilisateurs trouvés dans `/etc/passwd` : `../../../../../home/user/user.txt` (Échec).

### Payload Final
Après plusieurs essais (Fuzzing manuel), nous testons simplement la présence d'un fichier `flag.txt` à la racine système.

**Payload victorieux :**
```text
[http://10.10.138.45/?page=../../../../../flag.txt](http://10.10.138.45/?page=../../../../../flag.txt)
