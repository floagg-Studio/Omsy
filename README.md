# Omsy

**Omsy** est un CMS modulaire écrit en **PHP**, pensé pour créer et administrer des applications web. Il fournit un noyau léger (sessions, base de données, permissions, cache, plugins) et un panneau d'administration séparé, extensible via des plugins et des templates.

> ⚠️ **Statut : en développement actif.** Le projet n'a pas encore fait l'objet d'un audit de sécurité complet. Lisez la section [Sécurité](#-sécurité) avant tout déploiement en production.

---

## 📚 Sommaire

- [Fonctionnalités](#-fonctionnalités)
- [Prérequis](#-prérequis)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [Structure du projet](#-structure-du-projet)
- [Base de données](#-base-de-données)
- [Plugins et templates](#-plugins-et-templates)
- [Sécurité](#-sécurité)
- [Conventions de code](#-conventions-de-code)
- [Contribuer](#-contribuer)
- [Licence](#-licence)
- [Liens et contact](#-liens-et-contact)

---

## ✨ Fonctionnalités

- **Architecture modulaire** : le noyau charge ses composants (`OmsySecurity`, `OmsySql`, `OmsyPermission`, `OmsySession`, `OmsyPlugin`, `OmsyInterface`, `OmsyCache`…) depuis la liste déclarée dans `Core.xml`.
- **Panneau d'administration** : accessible par défaut sur `/panel` (URI et dossier configurables).
- **Comptes et rôles** : gestion des comptes, des rôles et des permissions.
- **Plugins et templates** : le contenu et les fonctionnalités s'étendent via `usr/plugin` et un template actif défini dans la configuration.
- **Réécriture d'URL** : toutes les requêtes passent par `index.php` grâce à `.htaccess`.
- **PWA** : un `manifest.json` est fourni pour une expérience proche d'une application mobile.
- **Cache** : composant `OmsyCache`, avec une configuration Redis prévue.
- **E-mails** : module `OmsyPHPMailer` inclus.

---

## 🧾 Prérequis

| Élément | Version / détail |
|---|---|
| PHP | 7.4 minimum (développé et testé avec PHP 8.2) |
| Extensions PHP | `simplexml`, sessions, `mysqli` ou `pdo_mysql` |
| Serveur web | Apache avec `mod_rewrite` et `AllowOverride All` |
| Base de données | MySQL 5.7+ / MariaDB 10.4+ |
| Redis | Optionnel (cache) |

> Avec Nginx, les règles de `.htaccess` ne s'appliquent pas : la réécriture et le blocage de `Core.xml` doivent être recréés dans la configuration du serveur.

---

## ⚙️ Installation

1. **Cloner le dépôt**

   ```bash
   git clone https://github.com/floagg-Studio/Omsy.git
   cd Omsy
   ```

2. **Créer la configuration**

   ```bash
   cp default.Core.xml Core.xml
   ```

   `index.php` charge `Core.xml` au démarrage : sans ce fichier, l'application s'arrête avec « Erreur de chargement du fichier XML ».

3. **Créer la base de données et importer le schéma**

   ```bash
   mysql -u <utilisateur> -p -e "CREATE DATABASE omsy_sql CHARACTER SET utf8mb4;"
   mysql -u <utilisateur> -p omsy_sql < omsy_sql.sql
   ```

4. **Renseigner `Core.xml`** (section `<database>`) avec un utilisateur dédié, **pas `root`** (voir [Configuration](#-configuration)).

5. **Configurer Apache** : pointer le `DocumentRoot` vers le dossier du projet, activer `mod_rewrite` et autoriser les `.htaccess`.

6. **Ouvrir le site** dans votre navigateur, puis l'administration sur `/panel`.

---

## 🛠️ Configuration

Toute la configuration passe par `Core.xml` (modèle : `default.Core.xml`).

| Section | Rôle |
|---|---|
| `<omsy>` | Environnement (`local`, …), version, liste des composants à charger, options d'initialisation PHP, accès à l'administration, dossier de logs |
| `<database>` | Connexion MySQL / MariaDB |
| `<redis>` | Connexion Redis (cache) |
| `<template>` | Template actif, titre, plugins à activer |
| `<security>` | Autorisation des fonctions sensibles : `eval`, `exec`, `system`, `passthru` |
| `<server>` | Clés privée et de sel, URL du serveur de mises à jour |

### Options importantes

| Option | Valeur par défaut | Recommandation en production |
|---|---|---|
| `display_errors` | `1` | `0` : ne jamais afficher les erreurs aux visiteurs |
| `session.cookie_httponly` | `True` | Conserver |
| `session.cookie_secure` | `1` | Conserver (impose HTTPS) |
| `memory_limit` | `800M` | Réduire selon vos besoins réels |
| `<security><exec>` | `true` | Passer à `false` sauf nécessité |
| `<database><username>` | `root` | Utiliser un compte dédié aux droits limités |
| `<server><privatekey>` / `<saltkey>` | vides | Renseigner des valeurs longues et aléatoires |

Le fuseau horaire par défaut est `Europe/Paris` ; il se modifie dans `index.php` :

```php
date_default_timezone_set('Votre/Fuseau');
```

---

## 🗂️ Structure du projet

```
Omsy/
├── index.php          # Point d'entrée : charge Core.xml puis instancie OmsyMain
├── default.Core.xml   # Modèle de configuration (à copier en Core.xml)
├── .htaccess          # Réécriture d'URL et protection de Core.xml (Apache)
├── manifest.json      # Manifeste PWA
├── omsy_sql.sql       # Schéma de la base de données
├── sql_install.json   # Données d'installation SQL
├── panel/             # Panneau d'administration
├── sys/               # Noyau (sys/class/OmsyMain.php et composants)
├── usr/plugin/        # Plugins utilisateur
└── share/             # Ressources partagées (logs : share/_)
```

---

## 🗄️ Base de données

Le schéma `omsy_sql.sql` crée les tables suivantes :

| Table | Contenu |
|---|---|
| `omsy_account` | Comptes (identifiant, mot de passe, rôle, date d'inscription) |
| `omsy_account_login` | Jetons de connexion |
| `omsy_account_obj` | Objets rattachés à un compte |
| `omsy_object` | Objets génériques (type, nom, contenu) |
| `omsy_session` | Sessions et dernière activité |
| `omsy_viewpage` | Journal de pages vues (user-agent, IP, date) |
| `omsyreport_report` | Signalements |

> `omsy_viewpage` stocke des adresses IP : ce sont des données personnelles au sens du RGPD. Si votre site s'adresse à des visiteurs dans l'UE, prévoyez une information des utilisateurs et une durée de conservation limitée.

---

## 🔌 Plugins et templates

- Les plugins se placent dans `usr/plugin/` et s'activent via `<template><plugins>` dans `Core.xml` (liste séparée par des virgules).
- Le template actif est défini par `<template><folder>`.
- Le serveur de mises à jour est défini par `<server><update>`.

> Un plugin s'exécute avec les mêmes droits que le CMS. N'installez que des plugins de sources de confiance.

---

## 🔒 Sécurité

Points à traiter avant une mise en production :

1. **Désactiver l'affichage des erreurs** (`display_errors` à `0` dans `Core.xml` et `index.php`) et journaliser à la place.
2. **Protéger `Core.xml`** : il contient les identifiants de base de données. Idéalement, placez-le **en dehors du dossier public** ; à défaut, vérifiez qu'il renvoie bien une erreur 403 depuis le navigateur.
3. **Ne pas utiliser `root`** pour la connexion MySQL, et définir un mot de passe.
4. **Forcer HTTPS** (requis par `session.cookie_secure`).
5. **Limiter `exec`** : laissez `<exec>` à `false` si aucun plugin n'en a besoin.
6. **Renseigner `privatekey` et `saltkey`** avec des valeurs aléatoires.
7. **Restreindre l'accès à `/panel`** (liste d'adresses IP, authentification supplémentaire, limitation des tentatives).
8. **Sauvegarder** régulièrement la base et le dossier `share/`.

### Signaler une vulnérabilité

Merci de **ne pas ouvrir d'issue publique** pour une faille de sécurité : contactez directement l'auteur (voir [Contact](#-liens-et-contact)).

---

## 🧭 Conventions de code

| Élément | Style | Exemple |
|---|---|---|
| Nom de fichier | `snake_case` | `ce_si_est_un_nom` |
| Variable | `camelCase` | `ceSiEstUnNom` |
| Classe (fichier et variable de classe) | `PascalCase` | `CeSiEstUneClass` |
| Constante | `__snake_case__` | `__ce_si_est_une_constance__` |

---

## 🤝 Contribuer

Les contributions sont bienvenues. Pour une modification importante, ouvrez d'abord une **issue** afin d'en discuter.

1. Forkez le dépôt
2. Créez une branche : `git checkout -b feature/ma-fonctionnalite`
3. Commitez : `git commit -m "Ajout de ma fonctionnalité"`
4. Poussez : `git push origin feature/ma-fonctionnalite`
5. Ouvrez une Pull Request

---

## 📜 Licence

Aucune licence n'est définie pour le moment. Sans licence, le code reste protégé par défaut : personne n'a le droit de le réutiliser, le modifier ou le redistribuer. Contactez l'auteur ou ajoutez un fichier `LICENSE` (MIT, GPL, etc.) pour clarifier les droits.

---

## 🔗 Liens et contact

- **Dépôt** : <https://github.com/floagg-Studio/Omsy>
- **Issues** : <https://github.com/floagg-Studio/Omsy/issues>
- **Discord** : <https://discord.gg/mc7Tx89dsH>
- **Auteur** : is-omini ([@is-omini](https://github.com/is-omini)) — Discord : `@is_omini`