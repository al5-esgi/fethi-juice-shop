# Politique de mise à jour des dépendances

Périmètre : dépendances serveur et frontend, images de base et outils de CI du fork
pédagogique Juice Shop. Fethi est le mainteneur du fork ; les fonctions de sécurité
et de validation décrites ci-dessous sont des responsabilités, pas des approbations déjà obtenues.

## Cadence

- correctifs (patch) : revue hebdomadaire, intégration sous 7 jours après tests ;
  HIGH exploitable : sous 72 heures ;
- versions mineures : revue mensuelle, intégration sous 30 jours après vérification
  des changements et tests de non-régression ;
- versions majeures : décision mensuelle, estimation du coût et migration sur une
  branche dédiée ; sous 7 jours si elle corrige une CRITICAL exploitable ;
- CVE critique activement exploitée : qualification sous 4 heures, mesure de
  confinement sous 24 heures et correctif validé sous 48 heures ; une migration
  majeure ne dispense pas du confinement ;
- images de base : reconstruction et scan hebdomadaires, même sans changement
  applicatif ; conserver le digest et le SBOM de chaque image livrée ;
- scanners : à chaque push/PR. Un rapport en erreur ne vaut jamais absence de
  vulnérabilité. Toute exception précise son périmètre, son propriétaire, sa
  justification et sa date d'expiration.

## Qui décide

| Type de montée | Décide | Valide | Trace |
|---|---|---|---|
| Patch sans rupture | Mainteneur du fork | Build, lint, tests pertinents, nouveau scan | PR, lock files et rapports avant/après |
| Mineure | Mainteneur après lecture des notes de version | Tests serveur/API/frontend et parcours concernés | PR et note de compatibilité |
| Majeure | Mainteneur, après estimation de la migration et du risque | CI, tests des fonctions concernées ; revue du formateur pour le périmètre pédagogique | PR dédiée, décision et possibilité de retour arrière |
| CRITICAL exploitée | Mainteneur en charge de la sécurité | Vérification du confinement puis du correctif | Ticket prioritaire, échéance, preuve du nouveau scan |
| Exception pédagogique | Mainteneur propose, formateur valide | Périmètre local de formation et absence de données réelles | Registre de risque avec expiration |

## Lien MCO-MCS

Le MCO couvre disponibilité, compatibilité, sauvegarde et retour arrière. Le MCS
ajoute veille CVE, qualification des alertes, délais de correction et rescans des
images. Budget proposé pour l'exercice : une heure de maintenance par semaine ;
une migration majeure ou une urgence est estimée et priorisée séparément.

## Cas traité aujourd'hui

Relevé du **5 octobre 2026**, sur le `package-lock.json` commité, sans `audit fix` :

```text
juice-shop@20.1.1 -> pdfkit@0.11.0 (directe) -> crypto-js@3.3.0 (transitive)
```

- Identifiant : [GHSA-xwcq-pm8m-c4vf / CVE-2023-46233](https://github.com/advisories/GHSA-xwcq-pm8m-c4vf).
- Sévérité : **CRITICAL**, CVSS 9.1. Le défaut concerne les paramètres par défaut
  de PBKDF2 ; le correctif de `crypto-js` est disponible à partir de **4.2.0**.
- Preuve : `npm-audit.json` indique `isDirect: false`, `effects: ["pdfkit"]`
  et `fixAvailable: {"name":"pdfkit","version":"0.20.2","isSemVerMajor":true}`.
  OSV retrouve le même identifiant et alias sur `crypto-js@3.3.0`.
- Coût : migrer la dépendance directe `pdfkit` de 0.11.0 à 0.20.2, avec une
  rupture potentielle signalée par npm. Vérifier génération, contenu et
  téléchargement des factures (`routes/order.ts`) ; régénérer et committer les
  deux lock files seulement si les manifestes correspondants changent.
- Qualification : la version vulnérable est réellement présente. La route de
  facture n'appelle pas PBKDF2 et la présence du paquet seule ne démontre pas
  l'exploitation de cette CVE dans Juice Shop. Le nouveau rapport remonte aussi
  [GHSA-rg76-677x-56q9](https://github.com/advisories/GHSA-rg76-677x-56q9) sur le
  générateur aléatoire de `crypto-js` ; `pdfkit` utilise cette fonction, ce qui
  justifie une revue spécifique des chemins de chiffrement des PDF.
- Décision : proposer une PR de migration `pdfkit@0.20.2`, estimée à une journée,
  avec qualification sous 24 heures et validation sous 7 jours pour un usage
  réel. **Dans le TP4**, conserver le lock fourni et les résultats rouges pour
  l'exercice ; limiter l'application à l'environnement de formation, sans
  données réelles. L'exception proposée expire à la fin du module ou sous 30
  jours, au premier terme atteint, et nécessite validation du formateur.

On met à jour la dépendance directe qui introduit le paquet vulnérable ; ajouter
`crypto-js` à la racine ou forcer `npm audit fix --force` ne garantit ni le
remplacement de toutes ses occurrences ni la compatibilité de `pdfkit`.
