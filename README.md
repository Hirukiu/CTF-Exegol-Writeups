# CTF Exegol — Write-ups

Write-ups de 6 machines compromises lors du module de sécurité offensive d'Epitech (B-SPE-321), réalisées depuis l'environnement [Exegol](https://exegol.com).

Pour chaque machine : un premier write-up pour l'accès initial (`user.txt`), puis un second (`#2`) pour l'élévation de privilèges jusqu'à root (`root.txt`). Chaque document suit la même structure : contexte, reconnaissance, identification de la vulnérabilité, exploitation, outils et techniques utilisés.

## Machines

| Machine | Accès initial | Élévation de privilèges | Difficulté |
| --- | --- | --- | --- |
| [Batman](batman's_secret.md) / [#2](batmans_secret_2.md) | Code récupéré par FTP anonyme, injection de commandes dans un service réseau, reverse shell | Droit `sudo` sur un script Python, PYTHONPATH hijacking | Easy |
| [Fun with Functional](fun_with_functional.md) | Injection de code Template Haskell, reverse shell | — | Easy |
| [H4ck3rz](h4ck3rz.md) / [#2](h4ck3rz_%232.md) | Identifiants laissés dans le HTML, énumération web (gobuster), RCE via un panneau de commandes, contournement de filtre | Analyse d'un binaire exploitable, passage à root | Medium |
| [Jormungandr](jormungandr.md) / [#2](jormungandr_%232.md) | FTP anonyme, décodage d'un cache, SSH, reverse d'un `.pyc` puis exploitation d'un service | Reverse d'un binaire autorisé en `sudo`, désérialisation pickle | Medium |
| [Year 5739](year_5739.md) / [#2](year_5739_%232.md) | Exécution de code à distance via une application web | Droit `sudo` sur un script Python, PYTHONPATH hijacking | Easy |
| [Yer a Wizzard](yer_a_wizzard.md) / [#2](yer_a_wizzard_%232.md) | Cassage d'un secret | Rebonds entre utilisateurs, élévation jusqu'à root | Easy |

## Compétences mises en œuvre

- **Reconnaissance et énumération** : `nmap -sV -sC -p-`, `gobuster`, inspection manuelle du code source des pages
- **Exploitation web** : fuite d'informations, injection de commandes, injection de code, exécution de commandes à distance
- **Reverse engineering** : décompilation de bytecode Python, analyse de binaires
- **Élévation de privilèges Linux** : abus de droits `sudo`, PYTHONPATH hijacking, désérialisation `pickle`, mouvements latéraux entre comptes
- **Post-exploitation** : reverse shells, recherche de fichiers sensibles

## Avertissement

Ces write-ups portent sur des machines volontairement vulnérables, fournies dans un cadre pédagogique. Les techniques décrites ne doivent être utilisées que sur des systèmes pour lesquels on dispose d'une autorisation explicite.

## Auteurs

Mathieu Jin, avec la contribution de Wystan Marshall-Solot — Epitech, 2026.
