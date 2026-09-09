# Contexte de sécurité - fork Juice Shop

## 1. Contexte métier

Juice Shop simule une boutique de commerce en ligne permettant à des clients de créer un compte, consulter un catalogue, passer des commandes, payer et publier des avis. L'application manipule des données d'identité et de contact, des informations d'authentification, des historiques d'achat et des données liées au paiement. Le commerçant dépend de l'intégrité et de la disponibilité du service pour réaliser ses ventes. Une fuite, une fraude ou une interruption prolongée pourrait entraîner des pertes financières, des obligations au titre du RGPD et une dégradation durable de la confiance des clients.

## 2. Biens essentiels

| # | Bien essentiel | Pourquoi il a de la valeur métier | Biens supports qui le portent |
|---|---|---|---|
| BE1 | Comptes clients et données personnelles | Ils permettent d'identifier et de servir les clients ; leur confidentialité est nécessaire au respect du RGPD et au maintien de la confiance. | Base SQLite via Sequelize, serveur Express, jetons JWT, fichiers `encryptionkeys/` et journaux `logs/` |
| BE2 | Commandes et historique d'achat | Ils matérialisent les ventes, permettent la préparation des commandes et assurent la traçabilité des achats. | Base SQLite via Sequelize, serveur Express, API et routes de commande, jetons JWT |
| BE3 | Transactions et données liées au paiement | Leur intégrité et leur confidentialité conditionnent l'encaissement, la prévention de la fraude et la limitation des litiges. | Base SQLite via Sequelize, serveur Express, routes de paiement, jetons JWT et fichiers de configuration |
| BE4 | Continuité du service de vente en ligne et réputation de la boutique | La disponibilité de la boutique permet de générer du chiffre d'affaires ; son bon fonctionnement entretient la confiance des clients. | Serveur Express, frontend Angular, conteneur Docker, base SQLite et système de fichiers (`ftp/`, `logs/`) |

## 3. Sources de risque

Des cybercriminels, utilisateurs malveillants ou acteurs internes pourraient chercher à voler des données personnelles et de paiement, détourner des comptes, frauder sur les commandes ou perturber le service. Des attaques automatisées peuvent également viser l'indisponibilité de la boutique ou l'exploitation massive d'une vulnérabilité connue.

## 4. Événements redoutés

| # | Événement redouté (fait + impact) | Bien essentiel touché | Gravité (1 à 4) | Justification de la gravité |
|---|---|---|---|---|
| ER1 | Divulgation de l'ensemble des comptes clients et de leurs données personnelles, entraînant un préjudice pour les personnes, une notification à la CNIL et une perte de confiance. | BE1 | 4 | L'incident concernerait potentiellement tous les clients, exposerait l'organisation à des conséquences réglementaires et nuirait durablement à sa réputation. |
| ER2 | Modification non autorisée de commandes ou de montants de paiement, entraînant des fraudes, des pertes financières et des litiges avec les clients. | BE2 et BE3 | 4 | L'atteinte à l'intégrité touche directement les ventes et les paiements et peut produire des pertes importantes ainsi que des contestations multiples. |
| ER3 | Indisponibilité de la boutique pendant 24 heures en période de forte activité, empêchant les achats et provoquant une perte de chiffre d'affaires et d'image. | BE4 | 3 | L'activité commerciale est interrompue et la confiance se dégrade, mais l'impact reste temporaire si le service et les données sont restaurés rapidement. |

## 5. Suivi

- Run `ci` de référence : https://github.com/al5-esgi/fethi-juice-shop/actions/runs/34335824872
