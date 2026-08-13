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

Binaire compilé seul : 28 Mo.

## Reproduire

```bash
git clone --depth 1 --branch 2.2.12 https://github.com/miniflux/v2.git upstream
docker build -f docker/Dockerfile.v1_naif -t miniflux:v1 upstream/
docker images miniflux:v1
```