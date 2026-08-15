# Miniflux : de l'image au déploiement

Conteneurisation et optimisation de l'image Docker de [Miniflux](https://github.com/miniflux/v2)
(lecteur RSS en Go), publication automatisée via GitHub Actions, déploiement sur AWS.

Le code applicatif n'est pas modifié : le projet porte uniquement sur la chaîne
de build, de publication et de déploiement.

## Contexte

- Application : Miniflux v2, tag `2.2.12` (commit `459b1bf`)
- Toolchain : Go 1.24.0, lu depuis `go.mod`
- La source amont n'est pas versionnée dans ce dépôt (voir *Reproduire* ci-dessous)

## Mesures

| Version | Image de base | Taille | Durée de build |
|---------|---------------|--------|----------------|
| v1 : naïve | `golang:1.24.0` | 1,85 Go | 59 s |
| v2 : multi-stage | `debian:12.6-slim` | 158 Mo | 23 s | -91,5 % |
| v3 — minimale | `scratch` | 43,7 Mo | 19 s | −97,6 % |

Durées mesurées avec `--no-cache`. La v1 inclut le téléchargement initial de
l'image de base ; les durées ne sont donc pas strictement comparables entre elles.

Binaire compilé seul : 28 Mo.

## Choix techniques

### Pourquoi un build multi-stage

L'image v1 embarque tout l'outillage de compilation : toolchain Go, cache de
build, modules téléchargés, code source alors que seul le binaire est nécessaire
à l'exécution.

Supprimer ces fichiers dans une instruction ultérieure ne réduirait rien : les
couches d'une image sont immuables, un fichier effacé reste présent dans la couche
qui le contient. Le multi-stage contourne le problème en repartant d'une base
vierge et en n'y copiant que le binaire.

Un binaire Go compilé avec les réglages par défaut ne peut pas y tourner : CGO
étant actif, le binaire est lié dynamiquement à la bibliothèque C du système
(le paquet `net` appelle `getaddrinfo()` pour la résolution DNS). Le chargeur
dynamique étant absent de `scratch`, le lancement échoue sur un message
trompeur — `no such file or directory` désigne le chargeur, pas le binaire.

`CGO_ENABLED=0` force une implémentation Go pure de ces paquets et produit un
binaire statique, sans aucune dépendance externe.

### Certificats racine

`scratch` ne contient pas les certificats des autorités de certification.
Miniflux ne pourrait valider aucune connexion HTTPS vers les flux RSS. Ils sont
copiés depuis l'étage builder, dont l'image de base les fournit.

### Cache de couches

Les dépendances (`go.mod`, `go.sum`) sont copiées et téléchargées avant le code
source. Une modification du code n'invalide donc que la couche de compilation,
pas celle des dépendances, du plus stable au plus volatil.

### Adresse d'écoute

Par défaut Miniflux écoute sur `127.0.0.1`, c'est-à-dire uniquement depuis
l'intérieur de son propre conteneur : la publication de port reste sans effet et
l'application est injoignable. `LISTEN_ADDR=0.0.0.0:8080` ouvre l'écoute à
toutes les interfaces.


## Reproduire

```bash
git clone --depth 1 --branch 2.2.12 https://github.com/miniflux/v2.git upstream
docker build --no-cache -f docker/Dockerfile.v2_multistage -t miniflux:v2 upstream/
docker run --rm miniflux:v2 miniflux -version
```