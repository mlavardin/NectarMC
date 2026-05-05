# Manuel utilisateur - NECTAR

Ce document explique comment utiliser NECTAR en version portable.

> **BETA**  
> Le logiciel est encore en version BETA. Certaines fonctions peuvent donc contenir des bugs ou avoir un comportement imparfait. En cas de probleme, voir la section [Contact](#contact).

## Lancer l'application

La version livree est portable : elle ne necessite pas d'installation classique.

1. Decompresser le fichier `.zip` fourni.
2. Garder le contenu du dossier tel quel.
3. Lancer `nectar.exe`.

Le fichier `nectar.exe` doit rester dans le meme dossier que les fichiers fournis dans le zip. Il embarque et lance automatiquement les elements necessaires au fonctionnement de l'application :

- Grafana, pour afficher les tableaux de bord.
- InfluxDB, pour stocker les donnees de telemetrie.
- Le serveur Python, qui gere les routeurs, les decoms et les APIs.
- Le front de l'application, c'est-a-dire l'interface utilisateur.

Au demarrage, l'application peut prendre quelques secondes le temps de lancer les services internes.

## Identifiants par defaut

### Grafana

- URL locale : `http://localhost:3000`
- Identifiant : `admin`
- Mot de passe : `admin`

### InfluxDB

- URL locale : `http://localhost:8086`
- Identifiant : `simpleGCS`
- Mot de passe : `simpleGCS`

Dans une utilisation normale, il n'est pas necessaire d'ouvrir InfluxDB directement. Grafana et l'interface NECTAR utilisent les donnees automatiquement.

## Principe general

NECTAR sert a recevoir, traiter, stocker et visualiser des donnees de telemetrie.

Le fonctionnement se fait en deux grandes etapes :

- Les **Routeurs** receptionnent les donnees brutes depuis une source.
- Les **Decoms** traitent ces donnees pour les transformer en valeurs exploitables.

Un routeur peut recevoir des trames depuis differents types de sources, par exemple un port serie, un WebSocket ou un fichier de replay selon la configuration disponible dans l'interface.

Le decom utilise ensuite un fichier de decommutation pour savoir comment lire les octets de la trame : noms des champs, tailles, types de donnees, unites, conversions, champs GPS, etc.

## Utiliser les Routeurs

Dans la zone **Routers** de l'application :

1. Cliquer sur **New**.
2. Choisir le type de source.
3. Renseigner les informations demandees, par exemple le port serie et le baudrate.
4. Valider la creation du routeur.

Une fois lance, le routeur ecoute la source selectionnee et transmet les trames recues au serveur. L'interface affiche son etat et ses statistiques de reception.

## Utiliser les Decoms

Dans la zone **Telem(s)** de l'application :

1. Cliquer sur **New**.
2. Renseigner le **Mission ID**.
3. Choisir si les paquets bruts doivent etre sauvegardes avec **SAVE RAW PACKETS**.
4. Activer ou non la decommutation avec **ENABLE DECOMUTATION**.
5. Si la decommutation est activee, selectionner un fichier **BDS**.
6. Valider la creation du decom.

Le decom traite les trames correspondant a la mission et ecrit les donnees dans InfluxDB. Les donnees peuvent ensuite etre visualisees dans Grafana depuis l'application.

## Creer un fichier de decommutation

Les fichiers de decommutation sont des fichiers JSON de type **BDS**. Ils decrivent comment transformer une trame binaire en mesures lisibles.

Pour creer un fichier BDS depuis l'interface :

1. Ouvrir la creation d'un decom.
2. Dans la selection du fichier **BDS File**, cliquer sur le bouton **Create BDS**.
3. Le module **Decomb Builder** s'ouvre.
4. Renseigner les informations generales de la trame : identifiant, nom, taille en octets, description si besoin.
5. Ajouter les champs de telemetrie un par un.
6. Pour chaque champ, definir son nom, sa taille en bits, son type de donnee, son unite et eventuellement une fonction de conversion.
7. Si certains champs representent une position GPS, les marquer avec le role GPS correspondant.
8. Generer et telecharger le fichier JSON.
9. Revenir dans la creation du decom, importer ou selectionner ce fichier BDS, puis lancer le decom.

Le Decomb Builder calcule les offsets des champs et valide la structure avant de generer le JSON.

## Format des trames

Les trames attendues par les routeurs sont documentees dans le fichier suivant :

[FRAME_FORMAT.md](./FRAME_FORMAT.md)

Ce document decrit la structure des trames, le header, le SSID, l'APID, la taille du payload et le CRC.

## Visualisation des donnees

Les donnees traitees sont stockees dans InfluxDB puis affichees dans Grafana.

Depuis l'application, la zone Grafana permet de consulter les tableaux de bord associes aux donnees recues. Selon la configuration du fichier BDS, les mesures peuvent apparaitre sous forme de courbes, valeurs, panneaux GPS ou autres visualisations.

## Arreter l'application

Pour quitter NECTAR, fermer la fenetre de l'application. Les services lances avec `nectar.exe` sont arretes automatiquement.

Si un service semble encore occuper un port apres fermeture, relancer l'application peut prendre un peu plus de temps car elle essaie de nettoyer les anciens processus.

## Contact

Pour toute question, bug ou demande d'aide, contacter :

**Mathieu LAVARDIN**  
`mathieu4700@gmail.com`
