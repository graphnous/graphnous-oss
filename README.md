# Graphnous OSS

Open-source software intelligence for your codebase.

Graphnous analyzes your software and builds a graph of your systems, projects, modules, packages, files, classes, methods, dependencies, and more.

Run Graphnous yourself with Docker Compose and explore your codebase through the Graphnous application.

## Quick start

### Requirements

- Docker
- Docker Compose v2
- GitHub account (if analyzing private GitHub repositories)

Graphnous uses Docker to:

- Run the Graphnous application services
- Create isolated containers for Git checkouts
- Store repository checkouts in Docker volumes
- Execute language-specific scanning workloads

The Graphnous application therefore requires access to the Docker socket.

1. Clone the repository

```
git clone https://github.com/graphnous/graphnous-oss.git
cd graphnous-oss
```

2. Configure Graphnous

Copy the example environment file:

```
cp .env.example .env
```

Set the two required values, the database passwords, in `.env`:

```
POSTGRES_PASSWORD=change-me
NEO4J_PASSWORD=change-me-too
```

The Neo4j password needs at least 8 characters. Everything else is optional;
see [Configuration](#configuration).

On Linux, also set the group that owns the Docker socket, so the server can
start the checkout and scanner containers (leave it unset on Docker Desktop):

```
echo "DOCKER_GID=$(stat -c %g /var/run/docker.sock)" >> .env
```

3. Start Graphnous

```
docker compose up -d
```

Check the running services:

```
docker compose ps
```

The first start takes a minute: the web app waits for the server, which
waits for PostgreSQL and Neo4j. Once `docker compose ps` shows them all
healthy, open the Graphnous web application in your browser:

```
http://localhost:3000
```

## Docker Compose

[`docker-compose.yml`](docker-compose.yml) runs four services:

| Service | Image | Port on the host | What it does |
|---|---|---|---|
| `web` | `ghcr.io/graphnous/graphnous-web` | 3000 (`GRAPHNOUS_WEB_PORT`) | The web application. Its server calls the API at `http://server:1337`; the browser never calls the API itself |
| `server` | `ghcr.io/graphnous/graphnous-server` | 1337 (`GRAPHNOUS_SERVER_PORT`) | The API: projects, scans and the graph. Starts the checkout and scanner containers through the Docker socket |
| `postgres` | `postgres:17` | Not published | Application data: systems, projects and configuration |
| `neo4j` | `neo4j:5-community` | Not published | The software graph |

```mermaid
flowchart LR
    Browser[Browser]

    subgraph Compose[docker compose]
        Web[web :3000]
        Server[server :1337]
        Postgres[(postgres)]
        Neo4j[(neo4j)]
    end

    Docker[Docker Engine]

    Browser --> Web
    Web --> Server
    Server --> Postgres
    Server --> Neo4j
    Server -- Docker socket --> Docker
```

The services start in order: PostgreSQL and Neo4j first, then the server
once both are healthy, then the web app once the server is. The server
listens on the host too, for uploading scan results and for the GitHub
App's webhooks.

PostgreSQL and Neo4j are only reachable on the compose network. The compose
file reads every value from `.env`, and refuses to start without the two
passwords.

The images' version is `GRAPHNOUS_VERSION`, `edge` by default: the latest
build of graphnous-app's main branch.

## How Graphnous uses Docker

Docker is an integral part of the self-hosted Graphnous architecture.

Graphnous uses separate containers and volumes for repository processing so that application services do not need direct access to the host filesystem.

```mermaid
flowchart TD
    Web[Graphnous Web]
    Server[Graphnous App Server]

    Docker[Docker Engine]
    Checkout[Git Checkout Container]
    Volume[(Checkout Volume)]

    Java[Java Scanner]
    TypeScript[TypeScript Scanner]
    Python[Python Scanner]

    Postgres[(PostgreSQL)]
    Neo4j[(Neo4j)]

    Web --> Server

    Server --> Docker

    Docker --> Checkout
    Checkout --> Volume

    Volume --> Java
    Volume --> TypeScript
    Volume --> Python

    Server --> Postgres
    Server --> Neo4j
```

## Git checkout

When a scan starts, Graphnous creates a dedicated Docker container for checking out the repository.

The repository is checked out into a Docker volume. Scanner containers then access the same checkout volume.

```mermaid
flowchart LR
    Repository[Git Repository]

    Server[Graphnous App Server]
    Docker[Docker Engine]
    Checkout[Git Checkout Container]
    Volume[(Checkout Volume)]

    Repository --> Checkout
    Server --> Docker
    Docker --> Checkout
    Checkout --> Volume
```

This keeps repository access isolated from the Graphnous application container and avoids requiring the host filesystem to be mounted into the application.

For private repositories, the checkout container can authenticate using the configured SSH private key.

## Scanning

After the repository has been checked out, Graphnous runs the appropriate language scanners against the checkout.

```mermaid
flowchart TD
    Volume[(Checkout Volume)]

    Java[Java Scanner]
    TypeScript[TypeScript Scanner]
    Python[Python Scanner]
    Other[Other Language Scanners]

    Result[Graph Scan]

    Volume --> Java
    Volume --> TypeScript
    Volume --> Python
    Volume --> Other

    Java --> Result
    TypeScript --> Result
    Python --> Result
    Other --> Result
```

The resulting graph scan is processed by the Graphnous application and stored in Neo4j.

## Complete scan pipeline

```mermaid
flowchart LR
    Git[Git Repository]

    Checkout[Checkout Container]
    Volume[(Checkout Volume)]

    Scanner[Graphnous Scanner]
    Result[Graph Scan]

    App[Graphnous App Server]
    Neo4j[(Neo4j)]

    Git --> Checkout
    Checkout --> Volume

    Volume --> Scanner
    Scanner --> Result

    Result --> App
    App --> Neo4j
```

## Docker socket

The Graphnous application server needs access to the Docker socket:

```
GRAPHNOUS_DOCKER_SOCKET=/var/run/docker.sock
```

The Compose deployment mounts it into the application container:

```
volumes:
  - ${GRAPHNOUS_DOCKER_SOCKET:-/var/run/docker.sock}:/var/run/docker.sock
```

The server runs as a non-root user. To use the socket, it joins the group
that owns it: `DOCKER_GID`, which is `0` by default, as on Docker Desktop. On
Linux, set it to the socket's group:

```
DOCKER_GID=$(stat -c %g /var/run/docker.sock)
```

Access to the Docker socket gives the Graphnous server significant control over the Docker host.

Only run Graphnous with Docker socket access on infrastructure you trust.

## Git repositories

Graphnous can analyze public and private Git repositories.

### Public repositories

Public repositories can be checked out without additional Git credentials.

### Private repositories

For private repositories, Graphnous can use an SSH private key to authenticate with Git.

Configure the key, in double quotes because it spans several lines, and the
host keys of your Git servers:

```
GRAPHNOUS_GIT_SSH_PRIVATE_KEY="-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----"
GRAPHNOUS_GIT_SSH_KNOWN_HOSTS="github.com ssh-ed25519 AAAA..."
```

`ssh-keyscan github.com` prints the host keys. Graphnous only fetches from
hosts listed in `GRAPHNOUS_GIT_SSH_KNOWN_HOSTS`.

The corresponding public key must have access to the Git repository.

Graphnous uses the key for the checkout operation and does not require the private key to be committed to the repository.

## OpenAI

Graphnous can optionally use OpenAI-powered features.

Configure your API key:

```
OPENAI_API_KEY=...
```

AI functionality is optional. Graphnous can run without an OpenAI API key.

## Configuration

The main configuration options are:

| Variable | Required | Description |
|---|---|---|
| `POSTGRES_PASSWORD` | Yes | Password of the PostgreSQL instance |
| `NEO4J_PASSWORD` | Yes | Password of the Neo4j instance, at least 8 characters |
| `GRAPHNOUS_VERSION` | No | Graphnous image version: `edge` (default) or a release |
| `POSTGRES_DB` | No | PostgreSQL database, `graphnous` by default |
| `POSTGRES_USER` | No | PostgreSQL user, `graphnous` by default |
| `GRAPHNOUS_WEB_PORT` | No | Port of the web application on the host, `3000` by default |
| `GRAPHNOUS_SERVER_PORT` | No | Port of the server on the host, `1337` by default |
| `GRAPHNOUS_DOCKER_SOCKET` | No | Docker socket used for checkouts and scans, `/var/run/docker.sock` by default |
| `DOCKER_GID` | No | Group that owns the Docker socket, `0` by default; set it on Linux |
| `OPENAI_API_KEY` | No | OpenAI API key for AI features |
| `GRAPHNOUS_GIT_SSH_PRIVATE_KEY` | No | SSH private key for private Git repositories |
| `GRAPHNOUS_GIT_SSH_KNOWN_HOSTS` | No | Host keys of the Git servers, in `known_hosts` format |
| `GRAPHNOUS_GITHUB_ENABLED` | No | Enables the GitHub App, which reports on pull requests; see `.env.example` for its settings |

See [`.env.example`](.env.example) for the complete configuration.

## Data persistence

Graphnous uses Docker volumes for persistent data.

The deployment contains persistent storage for:

- PostgreSQL data
- Neo4j data
- Repository checkouts (the `graphnous-checkouts` volume)

Repository checkout volumes are used to share source code between the checkout and scanner containers.

Checkout data should be treated as sensitive because it can contain the complete source code of private repositories.

## Architecture

Graphnous separates relational application data from the software graph.

```mermaid
flowchart TD
    App[Graphnous App Server]

    Postgres[(PostgreSQL)]
    Neo4j[(Neo4j)]

    Systems[Systems]
    Projects[Projects]
    Configuration[Application Configuration]

    Snapshots[Project Snapshots]
    Scans[Scans]
    Modules[Modules]
    Packages[Packages]
    Files[Files]
    Classes[Classes]
    Methods[Methods]
    Dependencies[Dependencies]

    App --> Postgres
    App --> Neo4j

    Postgres --> Systems
    Postgres --> Projects
    Postgres --> Configuration

    Neo4j --> Snapshots
    Neo4j --> Scans
    Neo4j --> Modules
    Neo4j --> Packages
    Neo4j --> Files
    Neo4j --> Classes
    Neo4j --> Methods
    Neo4j --> Dependencies
```

The scanner produces a structured Graphnous scan which is then processed by the application and merged into the software graph.

## Updating Graphnous

Pull the latest images:

```
docker compose pull
```

Then restart the deployment:

```
docker compose up -d
```

To use a specific version:

```
GRAPHNOUS_VERSION=0.1.0
```

Then:

```
docker compose pull
docker compose up -d
```

## Stopping Graphnous

Stop the services without removing persistent data:

```
docker compose down
```

Start them again with:

```
docker compose up -d
```

To remove the containers and volumes:

```
docker compose down -v
```

Warning: removing volumes deletes your PostgreSQL, Neo4j, and checkout data.

## Open source

Graphnous is open source.

The goal is to make software architecture and code intelligence accessible to developers without requiring a hosted service.

You can:

- Run Graphnous locally
- Run it on your own infrastructure
- Analyze public or private repositories
- Contribute to the project
- Build scanners and integrations
- Integrate Graphnous into your development workflow

## Contributing

Contributions are welcome.

Before opening a pull request:

1. Fork the relevant repository.
2. Create a branch for your change.
3. Make your changes.
4. Run the tests.
5. Open a pull request.

For larger changes, please open an issue first so the approach can be discussed.

## Graphnous Cloud

Graphnous can also be used as a hosted service.

The self-hosted OSS deployment is designed to give you full control over your code and data.

Graphnous Cloud provides additional managed capabilities without requiring you to operate the Graphnous infrastructure yourself.

Learn more at:

https://graphnous.dev

## Sponsorship

Graphnous is developed as an open-source project.

If Graphnous is useful to you, consider supporting its development through GitHub Sponsors.

## License

See "LICENSE" (LICENSE) for the license applicable to this repository.

---

# Graphnous

Understand your software as a graph.
