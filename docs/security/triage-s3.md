# Triage SAST et secrets - Séance 3

## Périmètre et preuve

Le triage porte sur les résultats Semgrep et Gitleaks obtenus sur la branche
`tp3-sast`. Les deux jobs sont volontairement bloquants afin qu'une alerte ne
puisse pas être ignorée silencieusement.

- [Run SAST et secrets en échec attendu](https://github.com/al5-esgi/fethi-juice-shop/actions/runs/34346088415)
- [Capture du run](./tp3-run-rouge.png)
- [Modèle de menaces et exigences](./threat-model.md)

## Résultats qualifiés

| Fichier:ligne | Règle | CWE | Sévérité | Vrai ou faux positif | Action décidée |
|---|---|---|---|---|---|
| `routes/login.ts:34` | `express-sequelize-injection` | CWE-89 | Erreur, bloquante | **Vrai positif à corriger.** `req.body.email` est concaténé directement dans une requête SQL. L'appel à `security.hash()` présent sur la même ligne concerne le mot de passe et ne neutralise pas l'injection SQL. | Remplacer la concaténation par une requête paramétrée et ajouter un test de non-régression. **Aucune exigence du modèle actuel n'est associée : c'est un manque à ajouter au prochain atelier de modélisation.** |
| `lib/insecurity.ts:150` | `juiceshop-hardcoded-private-key` | CWE-798 | Erreur, bloquante | **Vrai positif à corriger.** La ligne ne contient pas elle-même la clé, mais Semgrep suit la valeur de `privateKey`, déclarée en dur à la ligne 21, jusqu'à son utilisation par `createHmac()`. | Sortir la clé du code, utiliser un gestionnaire de secrets et prévoir sa rotation. Lié à [STRIDE #1 et EX-01](./threat-model.md#3-analyse-stride). |
| `lib/insecurity.ts:41` | `juiceshop-weak-hash-md5` | CWE-327 | Erreur, bloquante | **Vrai positif à corriger.** La fonction `hash()` applique MD5 sans sel aux mots de passe du modèle utilisateur ; une fuite de la base permettrait un cassage hors ligne rapide. | Migrer vers Argon2id ou bcrypt avec un sel unique et un coût adapté, puis migrer les empreintes existantes. Lié à [STRIDE #5 et EX-05](./threat-model.md#3-analyse-stride). |
| `routes/login.ts:64` | `generic-api-key` (Gitleaks) | CWE-798 | Élevée, bloquante | **Vrai positif avec risque accepté.** La valeur Base64 est un identifiant volontairement codé en dur pour le challenge OAuth de Juice Shop. Elle ne donne accès à aucun service de production, mais son motif ressemble correctement à une clé générique. | Exception ciblée dans `.gitleaksignore` par empreinte exacte, datée du 2026-09-09. L'acceptation est limitée au dépôt pédagogique ; la valeur devra être supprimée avant toute réutilisation en production. **Aucune exigence STRIDE existante ne couvre ce challenge.** |

## Décision globale

La CI reste rouge tant que les vrais positifs non acceptés sont présents.
L'exception Gitleaks n'est ni une règle globale ni un chemin complet ignoré :
elle vise uniquement le finding `generic-api-key` de `routes/login.ts:64` dans
le commit où il a été introduit. Toute nouvelle occurrence continuera donc à
faire échouer le job de détection de secrets.
