# Miniflux: from image to deployment

Containerizing and optimizing the Docker image for [Miniflux](https://github.com/miniflux/v2)
(a feed reader written in Go), with automated publishing through GitHub Actions
and deployment to AWS.

The application code is untouched. This project covers the build, publish and
deployment chain only.

## Context

- Application: Miniflux v2, tag `2.3.3`
- Toolchain: Go 1.26
- Upstream source is not vendored in this repository (see *Reproduce* below)

## Running the stack

`docker-compose.yaml` runs Miniflux alongside its PostgreSQL database. It builds
nothing: it pulls the image published by the pipeline.

The ECR repository is private, so this only works with credentials for the AWS
account that hosts it. It documents how the stack is wired, not an open
procedure.

Copy `.env.example` to `.env`, then set `MINIFLUX_IMAGE` to the build you want to
run. Images are tagged with the commit SHA, so there is no "current" tag, pick
one from the ECR repository.

```bash
aws ecr get-login-password --region eu-west-3 \
  | docker login --username AWS --password-stdin 396608811172.dkr.ecr.eu-west-3.amazonaws.com

docker compose up -d
```

Miniflux listens on port 8080. On the EC2 instance, the same file runs the same
image, only the `.env` differs.

## Image size

| Version | Base image | Size | Reduction | Build time |
|---------|------------|------|-----------|------------|
| v1 naive | `golang:1.26` | 1.85 GB | - | 59 s |
| v2 multi-stage | `debian:12.6-slim` | 158 MB | −91.5 % | 23 s |
| v3 minimal | `scratch` | 43.7 MB | −97.6 % | 19 s |

Build times measured with `--no-cache`. The v1 figure includes pulling the base
image, so the three are not strictly comparable.

Compiled binary alone: 28 MB.

## Vulnerabilities

Because the v3 image starts from `scratch`, it ships no OS packages: the scanner
has a single target, the binary. A Debian base would add a second one covering
every package in the distribution.

| Application version | Toolchain | HIGH + CRITICAL CVEs |
|---------------------|-----------|----------------------|
| Miniflux 2.2.12 | Go 1.24.0 | 41 (1 CRITICAL) |
| Miniflux 2.3.3 | Go 1.26 | 0 |

All 41 came from outdated dependencies. The Go standard library compiled into
the binary and the `golang.org/x/*` modules pinned in the upstream `go.mod`.
No suppression was needed, upgrading resolved every one of them.

## Continuous integration

The pipeline (`.github/workflows/docker-image.yml`) runs on every push: fetch the upstream
source, build the image, check that the binary runs, scan for vulnerabilities,
publish to ECR.

Images are tagged with the commit SHA, never with a mutable tag. Any deployed
version stays traceable and can be rolled back.

The scan runs before publishing and fails the pipeline on any fixable HIGH or
CRITICAL finding, so a vulnerable image never reaches the registry.

### Authentication without stored credentials

The pipeline holds no AWS access keys. GitHub issues a signed identity token on
each run; AWS validates it through an OIDC provider and returns temporary
credentials valid for one hour.

The assumed role's trust policy restricts it to the
`akramdaboussi/miniflux-docker-optimization` repository and its permissions
allow pushing to one specific ECR repository and nothing else.

An access key stored in repository secrets would be valid indefinitely and, if
leaked, would grant lasting access to the AWS account.

## Deployment

The pipeline deploys to a single EC2 instance running Amazon Linux, where Docker
Compose runs Miniflux alongside PostgreSQL.

GitHub never connects to the instance. The deploy job sends a command to AWS
Systems Manager; an agent already running on the instance polls for it and
executes it. All traffic is outbound, from the instance to AWS.

The instance therefore needs no inbound port beyond the application itself: port
8080 is the only rule in its security group, and SSH is closed. There is no
private key to distribute, rotate or leak and shell access, when needed goes
through Session Manager over the same outbound channel.

Two identities are involved, each scoped to one task:

- the pipeline assumes a role allowed to send one command document to one
  instance, and nothing else
- the instance carries a role allowing read-only pulls from ECR

The deploy step fetches `docker-compose.yaml` from the repository at the deployed
commit, rewrites the image tag in the instance's `.env`, logs in to ECR and
restarts the stack. Compose only recreates the container whose image changed, so
the database keeps running.

Fetching the Compose file at the commit SHA rather than from `main` keeps the
deployment tied to one exact state: the file that runs on the instance is the
file that was reviewed alongside the image being deployed. The instance holds no
copy to keep in sync only `.env`, which carries the secrets and never enters
version control.

Miniflux waits for PostgreSQL to report healthy before starting, rather than
merely for its container to exist. Without that, the application starts against a
database that is not yet accepting connections, crashes, and is restarted until
it happens to succeed.

## Design decisions

### Why multi-stage

The v1 image ships the entire build toolchain: the Go compiler, its build cache,
downloaded modules and the source tree when only the binary is needed at
runtime.

Deleting those files in a later instruction saves nothing. Image layers are
immutable: a deleted file still occupies the layer that contains it. Multi-stage
sidesteps this by starting from a clean base and copying in only the binary.

A Go binary built with default settings will not run there. CGO is enabled by
default, so the binary links dynamically against the system C library (the `net`
package calls `getaddrinfo()` for DNS resolution). `scratch` has no dynamic
loader, and the failure message is misleading, `no such file or directory`
refers to the loader, not to the binary.

`CGO_ENABLED=0` selects pure Go implementations of those packages and produces a
static binary with no external dependencies.

### Root certificates

`scratch` ships no CA certificates, so Miniflux could not validate any HTTPS
connection to a feed. They are copied from the builder stage, whose base image
provides them.

### Layer caching

`go.mod` and `go.sum` are copied and resolved before the source tree. A code
change then invalidates only the compile layer, not the dependency layer
ordering from the most stable to the most volatile.

### Listen address

Miniflux listens on `127.0.0.1` by default, reachable only from inside its own
container: publishing a port has no effect and the application is unreachable.
`LISTEN_ADDR=0.0.0.0:8080` opens it on all interfaces.

## Known limitations

The instance is created and configured by hand. A destroyed instance would have
to be rebuilt the same way.

Secrets live in a plaintext `.env` on the instance. Reading them from Parameter
Store at deploy time would remove the last stored credential in the chain.

The deploy job reports success once the command has run, not once the
application answers. A request against the running instance would close that
gap.