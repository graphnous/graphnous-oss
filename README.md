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

Configure the required values:

```
GRAPHNOUS_VERSION=latest
```

### PostgreSQL
```
GRAPHNOUS_POSTGRES_URL=jdbc:postgresql://postgres:5432/graphnous
GRAPHNOUS_POSTGRES_USERNAME=graphnous
GRAPHNOUS_POSTGRES_PASSWORD=change-me
```
### Neo4j
```
GRAPHNOUS_NEO4J_URL=bolt://neo4j:7687
GRAPHNOUS_NEO4J_USERNAME=neo4j
GRAPHNOUS_NEO4J_PASSWORD=change-me
```
### Docker
```
GRAPHNOUS_DOCKER_SOCKET=/var/run/docker.sock
```
### Onboarding
```
GRAPHNOUS_ONBOARDING_ENABLED=true
```
### Optional: AI features
```
GRAPHNOUS_OPENAI_API_KEY=
```
### Optional: private Git repositories
```
GRAPHNOUS_GIT_SSH_PRIVATE_KEY=
```
3. Start Graphnous

```
docker compose up -d
```

Check the running services:

```
docker compose ps
```

Graphnous is now running locally.

Open the Graphnous web application in your browser:

```
http://localhost
```

How Graphnous uses Docker

Docker is an integral part of the self-hosted Graphnous architecture.

Graphnous uses separate containers and volumes for repository processing so that application services do not need direct access to the host filesystem.

```mermaidjs
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

```mermaidjs
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

```mermaidjs
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

```mermaidjs
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
  - ${GRAPHNOUS_DOCKER_SOCKET}:/var/run/docker.sock
```

Access to the Docker socket gives the Graphnous server significant control over the Docker host.

Only run Graphnous with Docker socket access on infrastructure you trust.

## Git repositories

Graphnous can analyze public and private Git repositories.

### Public repositories

Public repositories can be checked out without additional Git credentials.

### Private repositories

For private repositories, Graphnous can use an SSH private key to authenticate with Git.

Configure the key using:
```
GRAPHNOUS_GIT_SSH_PRIVATE_KEY=
```

The corresponding public key must have access to the Git repository.

Graphnous uses the key for the checkout operation and does not require the private key to be committed to the repository.

## OpenAI

Graphnous can optionally use OpenAI-powered features.

Configure your API key:

```
GRAPHNOUS_OPENAI_API_KEY=...
```

AI functionality is optional. Graphnous can run without an OpenAI API key.

## Configuration

The main configuration options are:

Variable| Required| Description
"GRAPHNOUS_VERSION"| No| Graphnous image version
"GRAPHNOUS_POSTGRES_URL"| Yes| PostgreSQL JDBC URL
"GRAPHNOUS_POSTGRES_USERNAME"| Yes| PostgreSQL username
"GRAPHNOUS_POSTGRES_PASSWORD"| Yes| PostgreSQL password
"GRAPHNOUS_NEO4J_URL"| Yes| Neo4j Bolt URL
"GRAPHNOUS_NEO4J_USERNAME"| Yes| Neo4j username
"GRAPHNOUS_NEO4J_PASSWORD"| Yes| Neo4j password
"GRAPHNOUS_DOCKER_SOCKET"| Yes| Docker socket used for checkouts and scans
"GRAPHNOUS_ONBOARDING_ENABLED"| No| Enables the initial onboarding experience
"GRAPHNOUS_OPENAI_API_KEY"| No| OpenAI API key for AI features
"GRAPHNOUS_GIT_SSH_PRIVATE_KEY"| No| SSH private key for private Git repositories

See ".env.example" for the complete configuration.

## Data persistence

Graphnous uses Docker volumes for persistent data.

The deployment contains persistent storage for:

- PostgreSQL data
- Neo4j data
- Repository checkouts

Repository checkout volumes are used to share source code between the checkout and scanner containers.

Checkout data should be treated as sensitive because it can contain the complete source code of private repositories.

## Architecture

Graphnous separates relational application data from the software graph.

```mermaidjs
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
