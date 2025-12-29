# Silverpeas Exploitation - TryHackMe Write-up

## 🎯 Objectif
Ce dépôt documente la résolution d'une room TryHackMe ciblant l'application **Silverpeas**. L'objectif est d'enchainer plusieurs CVEs (Auth Bypass, Information Disclosure) pour obtenir un accès initial, puis d'exploiter une mauvaise configuration des permissions de groupe (`adm`) et des droits `sudo` pour devenir **root**.

**Compétences démontrées :**
* **Reconnaissance Web** (Identification de version & Endpoints).
* **Recherche & Exploitation de CVE** (Auth Bypass & IDOR/SQLi).
* **Manipulation de requêtes HTTP** (Burp Suite).
* **Énumération Système** (Analyse des groupes et des logs).

---

## 🔍 1. Reconnaissance Web

### Scan de ports & Découverte
Le scan Nmap initial révèle les ports **80** et **8080** ouverts.
En visitant le port 80, nous trouvons un site web standard. Cependant, le port 8080 renvoie une erreur par défaut.
Une énumération de répertoires (Gobuster) ou une analyse du site sur le port 80 nous permet d'identifier le nom de l'application : **Silverpeas**.

Nous testons l'accès direct : `http://IP:8080/Silverpeas`, ce qui nous mène à la page de connexion de l'application.

![Page de login Silverpeas](web_login.webp)

---

## 🔓 2. Exploitation : Authentication Bypass & Data Leak

### Recherche de Vulnérabilités (CVE)
Une recherche Google "SilverPeas CVEs" nous dirige vers plusieurs failles critiques.
Nous identifions une vulnérabilité de **contournement d'authentification** (Authentication Bypass).

**Exploitation (Auth Bypass) :**
La faille réside dans le traitement de la connexion. Si le champ `password` est absent de la requête POST, le système valide la connexion.
1.  Nous capturons la requête de login avec **Burp Suite**.
2.  Nous supprimons le paramètre `password` du corps de la requête.
3.  Nous forwardons la requête modifiée.

![Burp Suite Request Modification](burp_bypass.webp)

Une fois la requête envoyée, nous désactivons l'interception et rafraichissons la page : nous sommes connectés en tant qu'administrateur.

### Obtention d'identifiants (Information Disclosure)
L'accès administrateur ne nous donne pas directement un shell. Nous cherchons une autre CVE pour lire des données sensibles.
Nous trouvons une référence : *"Multiple CVEs leading to File Read on Server"*.

En modifiant l'ID dans l'URL d'une fonctionnalité de messagerie (SQL Injection ou IDOR selon la version), nous pouvons lire les messages des autres utilisateurs.
En itérant jusqu'à **ID=6**, nous découvrons un message contenant des identifiants en clair.

![Message contenant les credentials](msg_creds.webp)

**Credentials trouvés :** `tim` / `[MOT_DE_PASSE]`

---

## 👣 3. Mouvement Latéral : Tim vers Tyler
*(Technique : Log Analysis via ADM Group)*

Nous nous connectons en SSH avec les identifiants de **tim**.
Une fois connecté, nous effectuons une énumération basique avant de lancer des outils automatisés.

### Analyse des groupes
La commande `id` révèle une information cruciale :

```bash
uid=1001(tim) gid=1001(tim) groups=1001(tim), 4(adm)
