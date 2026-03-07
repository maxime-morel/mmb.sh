---
title: "De VeraCrypt à Cryptomator : vers un stockage chiffré dans le cloud"
date: 2026-03-07
draft: false
tags: ["cloud", "cryptographie", "linux", "privacy", "cryptomator"]
summary: "Retour d’expérience sur un stockage cloud avec chiffrement côté client."
---

Le mot **« chiffré »** dans le cloud rassure. Pourtant, dans la plupart des cas, vos documents restent techniquement lisibles par l’hébergeur.

Lorsque vous stockez vos fichiers sur Google Drive ou sur un service similaire, la clé de déchiffrement n’est généralement pas entre vos mains, mais entre celles du fournisseur qui stocke vos données.

Résultat : si ce service doit analyser ou accéder à vos fichiers dans le cadre de son fonctionnement, de ses traitements internes ou de ses conditions d’utilisation, il en a la capacité technique. Et en cas d’accès frauduleux à votre compte, un attaquant peut lui aussi accéder directement à l’ensemble de vos documents.

À partir de là, il y a plusieurs solutions :

1. accepter ce fonctionnement et faire confiance au service utilisé
2. choisir une solution cloud spécialisée, souvent payante, comme Proton
3. se risquer à maintenir sa propre solution locale ou à domicile
4. ou alors… allier les avantages du stockage cloud avec la sécurité d’un chiffrement local.

C’est précisément cette quatrième approche que j’ai testée.

## Le besoin de départ

J’utilisais jusqu’ici un volume **VeraCrypt** local pour stocker mes documents sensibles.

Côté sécurité / chiffrement, rien à redire. En revanche, côté usage et stockage, **les limites étaient contraignantes** :

- le conteneur devenait vite volumineux, ce qui rendait la sauvegarde peu pratique et difficile à automatiser
- l’accès était limité aux machines disposant de VeraCrypt
- la consultation depuis mobile était compliquée, surtout avec des volumes de plusieurs Gb à transférer.

Je cherchais donc une alternative capable de conserver un bon niveau de confidentialité, sans perdre la praticité du cloud.

Je souhaitais également éviter les solutions par abonnement ou fortement orientées SaaS, comme Proton, tout en restant sur un outil maintenu, fiable, sécurisé et demandant peu de maintenance.

L’une des problématiques que je cherchais à résoudre était donc de **permettre une sauvegarde dans le cloud**, tout en pouvant **consulter mes données depuis mon smartphone.**

## L’approche retenue

L’idée est simple : **les fichiers sont chiffrés localement avant d’être envoyés dans le cloud**. Le fournisseur cloud n’a alors plus accès au contenu lisible des fichiers.

Dans mon cas, j’ai retenu le montage suivant :

- **Google Drive** comme stockage cloud distant, car j’utilisais déjà cet environnement, même si cette approche peut fonctionner avec d’autres fournisseurs, y compris compatibles S3 ;
- **[Cryptomator](https://cryptomator.org/)** pour le chiffrement côté client, compatible avec Windows, macOS, Linux et mobile ;
- **rclone** pour la synchronisation sous Kubuntu ; sur Windows ou macOS, l’application Google Drive peut être utilisée plus simplement ;
- l’application mobile **Cryptomator** pour l’accès ponctuel depuis smartphone.

Ce modèle permet de garder les avantages du cloud : accessibilité, souplesse, multi-appareils, tout en évitant de confier le contenu lisible des documents au service d’hébergement.

## Pourquoi Cryptomator ?

![Interface Cryptomator](/images/cryptomator/cryptomator-interface.png)

Pour commencer, [Cryptomator](https://cryptomator.org/) est open source et propose une solution gratuite sur desktop, sans obligation de passer sur un modèle SaaS. C’était l’une de mes contraintes dans le choix de la solution. L’application mobile, elle, est payante, autour de 30 euros, mais avec un achat unique et non un abonnement.

Il remplace avantageusement le modèle du gros conteneur chiffré dans mon cas d'utilisation. Au lieu d’avoir un seul gros fichier VeraCrypt, on travaille avec un **coffre chiffré fichier par fichier**, bien plus adapté à la synchronisation.

Cryptomator a aussi un autre avantage important : il est disponible sur **Windows, macOS, Linux, Android et iOS** et est nativement compatible avec Google Drive sur mobile (cela ne veut pas dire que l’expérience est identique à l'application Google Drive).

En pratique, Cryptomator apporte plusieurs bénéfices :
- l’accès mobile reste possible
- le stockage distant ne voit que des fichiers déjà chiffrés
- on évite les limites de sauvegarde liées à un conteneur unique

## Description du setup

### Sous Linux

Sur Linux, et en particulier sous mon système Kubuntu, il n’existe pas, à ma connaissance, de client Google Drive officiel équivalent à celui disponible sur Windows ou macOS. Il faut donc ajouter une couche supplémentaire pour la synchronisation.

J’ai choisi **rclone**, car c’est un outil robuste, flexible et bien adapté à une logique de synchronisation maîtrisée.

Le workflow retenu est le suivant :

1. travailler localement dans le coffre Cryptomator
2. synchroniser le répertoire chiffré vers Google Drive avec `rclone`
3. puis automatiser cette synchronisation avec un timer `systemd --user`.

### Sous Windows ou macOS

Sous Windows ou macOS, l’approche peut être un peu plus simple.

Il est possible d’utiliser :

- **Cryptomator** pour gérer le coffre ;
- l’application **Google Drive** officielle pour synchroniser le répertoire du coffre vers le cloud.

Le principe reste cependant le même : le coffre est chiffré **avant** la synchronisation. Le fournisseur cloud ne reçoit donc que des fichiers déjà chiffrés.

### Sur mobile

C’est un point important, parce que c’était justement l’une des limites de VeraCrypt dans mon cas.

J'ai cependant vu le prix de l'application mobile (29.99 euros), et j’ai été réticent de faire cet achat sans pouvoir tester l'application. Mais les avis étaient rassurants et j'avais déjà validé le fonctionnement local.

Avec Cryptomator, il est possible d’ouvrir le coffre directement depuis l’application mobile, connectée au service cloud. Cela permet d’accéder à ses fichiers depuis le téléphone de manière assez simple.

En revanche, il faut rester prudent avec les usages mobiles en écriture. Pour une consultation ponctuelle, le modèle est très pratique. Pour modifier régulièrement les fichiers depuis plusieurs appareils, je pense qu’il faut procéder avec prudence.

Je trouve l'application mobile assez simple et intuitive. Je n'ai qu'à sélectionner mon compte Google Drive (avec le dossier contenant la clé Cryptomator) et j'ai alors directement pu accéder à mes documents :

{{< two-images
    left="/images/cryptomator/cryptomator-stockage-mobile.png"
    altleft="Choix de la source de fichier cryptomator"
    right="/images/cryptomator/cryptomator-mobile.png"
    altright="Interface Cryptomator mobile"
>}}

J’avais également peur de devoir saisir mon mot de passe à chaque ouverture de l’application, mais les paramètres sont complets et ils ont intégré la biométrie, ce qui rend l’usage bien plus confortable au quotidien.

![Interface Cryptomator mobile](/images/cryptomator/cryptomator-settings.png)

## Mise en œuvre concrète sous Linux

Sur Linux, et en particulier sous Kubuntu, j’ai préféré éviter un montage direct du cloud dans l’environnement graphique comme base principale de travail.

J’ai retenu un modèle plus simple et plus prévisible :

1. créer un coffre Cryptomator dans un répertoire local
2. travailler localement sur ce coffre
3. utiliser `rclone` pour synchroniser le répertoire chiffré vers Google Drive
![rclone](/images/cryptomator/cryptomator-rclone.png)
4. automatiser l’opération à intervalles réguliers.

Cette approche me semble plus propre qu’un montage distant permanent, surtout pour un coffre contenant beaucoup de petits fichiers.

Par ailleurs elle a aussi un mérite important : **la source de vérité reste locale**. Le cloud devient un stockage synchronisé (que je vois comme un backup), pas le point de travail principal.

## Automatisation

Point important : `rclone sync` n’est pas une synchronisation temps réel. La commande lance une synchronisation ponctuelle lorsqu’elle est exécutée.

Pour automatiser cela proprement, j’ai mis en place un **timer systemd utilisateur** qui lance la synchronisation à intervalle régulier. Cela évite de déclencher une synchronisation à chaque micro-changement dans le coffre.

En pratique, cette automatisation fonctionne bien tant qu’on garde une règle simple : **éviter de modifier le même coffre en parallèle depuis plusieurs terminaux ou appareils**. Je n’ai pas encore testé ce cas en profondeur.

## Les limites de cette approche

Cette solution est intéressante, mais elle n’est pas parfaite.

**1. Ce n’est pas une synchronisation temps réel native sous Linux**

Le comportement dépend de la manière dont on a construit le workflow. Sous Linux, avec `rclone`, on est sur une logique de synchronisation périodique ou manuelle.

**2. Il faut rester discipliné sur les modifications simultanées**

Je n'ai pas poussé mes tests sur ce point, mais si un coffre est modifié à plusieurs endroits en même temps, par exemple sur le PC principal et sur mobile, on peut introduire des conflits ou des comportements imprévisibles.

Ce n’est pas forcément un problème si l’usage mobile reste ponctuel, mais il faut l’avoir en tête.

**3. La confiance ne disparaît pas totalement**

Le fournisseur cloud ne voit plus le contenu lisible des fichiers, mais il voit encore :
- qu’il existe des fichiers
- leur volume approximatif
- leur fréquence de modification
- certaines métadonnées de stockage

On améliore donc fortement la confidentialité du contenu, mais on ne devient pas invisible pour autant.

**4. La sauvegarde reste un sujet à part entière**

Une synchronisation cloud n’est pas un backup au sens strict. Si un fichier est supprimé ou corrompu localement, cette erreur peut aussi être répliquée.
Il faut donc compléter le dispositif par une vraie stratégie de sauvegarde indépendante. Je ne me suis pas encore penché sérieusement sur ce sujet. Pour le moment, mon ancien volume VeraCrypt local me sert encore de sauvegarde complémentaire.

# Mon avis après test

Ce montage n’est pas magique, mais je le trouve très cohérent pour obtenir un bon compromis autour d’un stockage cloud plus sécurisé.

Il permet :
- de conserver le confort d’un stockage cloud
- d’avoir une vraie couche de chiffrement côté client
- d’éviter les limites pratiques d’un gros volume VeraCrypt
- de rester sur une solution relativement simple et peu coûteuse
- et de retrouver un accès mobile raisonnable aux documents sensibles.

VeraCrypt reste excellent pour de l’archive, du stockage statique ou des volumes hors ligne. En revanche, je n’avais pas trouvé avec VeraCrypt un modèle vraiment pratique pour répondre à mes besoins de sauvegarde cloud et d’accès mobile.

## Ce que j’améliorerai ensuite

La suite logique sera probablement une journalisation plus propre des synchronisations et un verrouillage renforcé pour éviter les exécutions simultanées, ainsi qu'une vraie stratégie de backup indépendante de la simple synchronisation cloud.

Et éventuellement une réflexion plus poussée sur les usages mobiles en écriture.

## Conclusion

Si vous stockez aujourd’hui des documents sensibles dans un cloud classique en vous fiant uniquement au mot **« chiffré »**, il vaut probablement la peine de regarder de plus près qui détient réellement les clés.

Dans mon cas, ajouter une couche de chiffrement côté client m’a permis de conserver la praticité du cloud, sans lui abandonner l’accès direct au contenu de mes fichiers.

Ce n’est pas une solution parfaite. Elle demande un peu de méthode, un peu de discipline, et une bonne compréhension de ce qu’est réellement une synchronisation. Mais pour mon besoin, c’est aujourd’hui l’un des meilleurs compromis entre **confidentialité, simplicité et coût**.
