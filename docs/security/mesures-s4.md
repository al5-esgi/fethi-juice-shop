# TP4 — mesures de la chaîne de dépendances

Relevé local du **5 octobre 2026**. Node 24.15.0, npm 11.12.1,
OSV Scanner 2.5.1 et Trivy 0.74.0. Les bases de vulnérabilités et les tags
d'images évoluent : les nombres du cours, relevés le 8 septembre, ne sont pas
des résultats à recopier.

## Lock files et comparaison des scanners

Les deux fichiers `package-lock.json` et `frontend/package-lock.json` sont suivis
par Git et non ignorés. Aucun n'a été modifié pour ce TP. Le lock serveur contient
**1 457 entrées**, dont **654 marquées dev**, pour **66 dépendances directes** et
**51 dépendances de développement** déclarées dans le manifeste.

Les deux scanners du job `deps-scan` lisent **le même lock serveur** sans installer
de paquet. Les rapports JSON sont publiés dans l'artefact `dependency-reports`.

| Scanner | Résultat local | Décision |
|---|---|---|
| npm audit | 58 paquets affectés : 7 critical, 28 high, 20 moderate, 3 low | code 1 avec `--audit-level=high` |
| OSV Scanner | 1 271 paquets analysés ; 101 associations paquet/avis sur 40 paquets, 90 avis distincts | code 1 |

npm compte aussi les dépendances indirectement affectées (`effects`), tandis
qu'OSV liste des avis associés à une version. Un paquet peut avoir plusieurs avis,
et un avis toucher plusieurs paquets. Ces nombres ne sont donc pas comparables
comme deux totaux de « failles ». Les bases et les règles de propagation diffèrent.

Les deux étapes sont en `continue-on-error` pour aller au bout. Le seuil explicite
refuse HIGH/CRITICAL pour npm et **toute vulnérabilité pour OSV**, comme dans le
gabarit ; il bloque également une erreur de scanner. Le texte du seuil le précise.

[Run J1, job deps-scan rouge](https://github.com/al5-esgi/fethi-juice-shop/actions/runs/37286483870).
Le cas `pdfkit -> crypto-js` et la décision sont détaillés dans [la politique](politique-maj.md).

## SBOM

Le scan d'un checkout propre produit **1 464 composants**, au format
**CycloneDX 1.7**, dont deux composants `application` correspondant aux deux lock
files. Le frontend a désormais son propre lock, ce qui explique une partie de
l'écart avec les 747 composants annoncés dans l'annexe du cours.

Exemple observé : `crypto-js@3.3.0`, avec le PURL
`pkg:npm/crypto-js@3.3.0`. Ce PURL identifie l'écosystème, le paquet et sa version
pour les rapprocher d'un avis de sécurité. Le SBOM est un inventaire, pas un
certificat de sécurité. Le job `sbom` publie `sbom.cdx.json` dans l'artefact
`sbom-cyclonedx` et n'applique pas de seuil de vulnérabilités.

## Choix de la base finale

Scans distants avec Trivy 0.74.0, plateforme **linux/amd64**, sans filtre de
sévérité pour le JSON. Le tableau compte uniquement la couche **système Debian**.

| Base | Debian | CRITICAL | HIGH | MEDIUM | LOW | UNKNOWN | Total système |
|---|---|---:|---:|---:|---:|---:|---:|
| `node:24` | 12.15 | 32 | 479 | 2 465 | 1 476 | 28 | 4 480 |
| `node:24-slim` | 12.15 | 4 | 53 | 101 | 76 | 2 | 236 |
| `gcr.io/distroless/nodejs24-debian13` | 13.7 | 0 | 0 | 23 | 8 | 0 | 31 |

Les deux images Node ajoutent chacune **20 findings node-pkg** provenant des
outils embarqués (8 HIGH, 11 MEDIUM, 1 LOW). La base distroless n'a pas cette
couche npm et réduit les findings système HIGH/CRITICAL de **511 à 0**.
Cela ne supprime pas les vulnérabilités des dépendances de Juice Shop.

Digests des images de base observées :

- `node:24` : `sha256:64af3819f9275802414d7cdc38c27e9d82bd564dec4d4da87d008255d36c63b4` ;
- `node:24-slim` : `sha256:0e0ff40c39bc087845bfb27465a0df4ea419520094bc35842ff83dd8cbe6f9b6` ;
- distroless : `sha256:96df910f65fdd8a21d00d14d4cc046adcfcf3ced2d5e96be4b39ebde9f4866c6`.

## Dockerfile dégradé et durci

`Dockerfile.ci` conserve exactement le support dégradé du TP. Son
`JWT_SECRET` est une valeur fictive fournie pour l'exercice : **vrai positif de
secret intégré à l'image**, sans exclusion Gitleaks. Cette image ne sert pas au
déploiement. Gitleaks 8.30.1 vérifié sur les commits du TP4 détecte exactement une
nouvelle occurrence : règle `generic-api-key`, `Dockerfile.ci:6`, commit `39ed66f`.

`Dockerfile.hardened` reprend les principes du Dockerfile upstream :

- `npm ci` installe les versions verrouillées dans l'étage `builder` et compile
  Angular ; `npm run build:server` vérifie explicitement la compilation serveur,
  car le postinstall peut en masquer l'échec ;
- `npm prune --omit=dev --ignore-scripts` retire les outils de développement.
  Seuls les fichiers nécessaires sont réunis dans `/runtime` ; les sources
  des exercices de code sont nécessaires à l'exécution de Juice Shop ;
- la dernière étape part de distroless et copie `/runtime` : pas de npm, shell,
  compilateur, dépendances frontend de build, documentation ou Dockerfile dégradé ;
- `COPY --chown=65532:0` permet au processus d'écrire dans les répertoires de
  l'application ; `USER 65532` sélectionne un utilisateur non-root portable entre
  les images. Le nom `node` n'est pas garanti dans distroless ;
- aucune instruction `ENV` ou `ARG` ne contient de secret. Juice Shop conserve
  ses clés pédagogiques d'origine : durcir l'image ne sécurise pas toute l'application.

Un `RUN rm` après un secret déjà enregistré dans l'image finale ne le retire pas
des layers précédents, consultables avec `docker history`. Ici, les fichiers du
support dégradé ne sont jamais copiés dans l'étage final. Gitleaks parcourt aussi
l'historique Git : ajouter un Dockerfile durci n'efface pas le finding historique.

Le [run J2](https://github.com/al5-esgi/fethi-juice-shop/actions/runs/37286573433)
construit et scanne le support dégradé. Le workflow final construit
`Dockerfile.hardened` et publie son SARIF sous la catégorie **`trivy-image`**
dans **Security > Code scanning**, ainsi qu'en artefact `trivy-image-sarif`.
Le seuil HIGH/CRITICAL continue de bloquer les vulnérabilités applicatives restantes.

Écart corrigé par rapport au gabarit : le SARIF J2 contient **4 830 résultats de
toutes sévérités**. L'[action Trivy 0.36.0](https://github.com/aquasecurity/trivy-action/blob/v0.36.0/entrypoint.sh)
efface le filtre de sévérité pour SARIF si `limit-severities-for-sarif` n'est pas
explicitement activé. Le workflow final fixe cet input à `"true"` pour publier
et bloquer uniquement HIGH/CRITICAL.

Vérification locale du Dockerfile durci : build réussi sur linux/arm64, utilisateur
`65532`, pas de `JWT_SECRET` dans l'environnement de l'image, et réponse HTTP
`{"version":"20.1.1"}` sur `/rest/admin/application-version`. La CI reproduit
un test de démarrage avant le scan ; un échec de compilation ou de lancement
ne peut donc pas être présenté comme un simple finding de sécurité.

Le scan de l'image durcie exportée localement avec Trivy 0.74.0 distingue :

| Couche de l'image locale linux/arm64 | CRITICAL | HIGH | MEDIUM | LOW | Total |
|---|---:|---:|---:|---:|---:|
| Système Debian 13.7 | 0 | 0 | 23 | 8 | 31 |
| Dépendances Node.js de Juice Shop | 9 | 48 | 36 | 5 | 98 |

L'image reste donc bloquée par **57 findings HIGH/CRITICAL applicatifs**. Le scan
J2 de l'image dégradée sur GitHub (linux/amd64) comptait **701 résultats SARIF
de niveau error**. La CI du workflow final vérifie la nouvelle image sur
linux/amd64 ; son artefact SARIF est la preuve à retenir pour cette plateforme.
