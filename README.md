# JavaMifi Local Development

This repository helps you run JavaMifi on your computer with Docker. It can
run the main frontend, CMS, and backend, or a separate stack for a worktree.

If this is your first time, follow **First run**. You can ignore the rest until
you need it.

## Before you start

Install these tools:

- Docker with Docker Compose v2
- Node.js 20 and npm
- Git
- `curl` and `gzip`

Keep this repository next to the three application repositories:

```text
javamifi/
|-- javamifi-local/
|-- javamifi.com-be/
|-- javamifi.com-fe/
`-- cms.javamifi.com/
```

The folder names matter. The local scripts use this layout to find the main
checkouts and their worktrees.

## First run

### 1. Install the local commands

From this repository:

```bash
./install
```

This installs `javamifi-db` and `javamifi-localhost` in `~/.local/bin`. If the
installer says that directory is not on your `PATH`, add the export command it
prints, then open a new terminal or reload your shell.

You can also run the scripts directly from this folder, for example
`./javamifi-localhost`.

### 2. Prepare the local database

Ask the team for an approved gzip-compressed SQL snapshot. Do not commit or
share database snapshots; they may contain sensitive data.

From the backend repository, restore the snapshot once:

```bash
cd ../javamifi.com-be
javamifi-db baseline restore /path/to/snapshot.gz javamifi_staging_YYYYMMDD
```

The second argument is just a name for the snapshot. Use a date, such as
`javamifi_staging_20260820`.

### 3. Start JavaMifi

From `javamifi-local`:

```bash
cd ../javamifi-local
javamifi-localhost up --all --name main
```

The command starts Docker containers, prepares the backend database, and
prints the frontend and CMS URLs. Open the URL you need in your browser.

The first start may take a while because npm dependencies and Docker images
may need to be downloaded.

## Everyday commands

Run these from `javamifi-local` unless noted otherwise:

```bash
# Show application stacks and live health
javamifi-localhost status

# Show details for one stack
javamifi-localhost status <stack-id>

# View logs for a stack, or for one service
javamifi-localhost logs <stack-id>
javamifi-localhost logs <stack-id> backend

# Stop one stack
javamifi-localhost down <stack-id>

# Stop all application stacks
javamifi-localhost down -all

# Stop and remove every stack using one worktree
javamifi-localhost cleanup --path ../javamifi.com-fe-worktrees/feature-example
```

Status checks the current Compose containers and application URLs. `UP` means
all configured application endpoints respond, `DEGRADED` means a stack is
partially running or unreachable, `DOWN` means its application containers are
stopped, and `UNKNOWN` means Docker Compose could not be queried. Runtime
metadata and old tunnel URLs remain visible after `down`, but they are not
treated as proof that the stack is available.

Stopping an application stack removes its containers, network, and
project-scoped volumes but does not delete its database or generated runtime
state. Starting the same stack again reuses its local ports, but creates new
temporary Quick Tunnel URLs. Always use the URLs printed by the latest `up`
command. Use `cleanup --path` after merging a worktree to remove its runtime
state as well.

## Run a worktree

You can run a frontend and backend worktree together:

```bash
javamifi-localhost up \
  --fe ../javamifi.com-fe-worktrees/feature-example \
  --be ../javamifi.com-be-worktrees/feature-example \
  --name feature-example
```

You can also run only a CMS worktree with the main backend:

```bash
javamifi-localhost up \
  --cms ../cms.javamifi.com-worktrees/feature-example \
  --be ../javamifi.com-be \
  --name feature-example
```

When you run `javamifi-localhost up` inside a JavaMifi checkout, it usually
selects that checkout automatically and finds a matching backend or frontend
worktree when one exists.

## Database commands

Run database commands from a backend checkout:

```bash
# Prepare the worktree database and test database
javamifi-db setup

# Show database names and status
javamifi-db status

# Run tests against the isolated test database
javamifi-db test

# Recreate both local databases
javamifi-db reset

# Remove both local databases before deleting a backend worktree
javamifi-db drop
```

Use `javamifi-db test` instead of running `npm test` directly. It checks that
tests are connected to this worktree's test database, not another database.

After merging a feature, remove its application runtime before deleting the
worktree:

```bash
javamifi-localhost cleanup --path /path/to/worktree
(cd /path/to/worktree && javamifi-db drop)
```

The first command stops every matching application stack, removes its
containers, network, project volumes, and generated runtime state. The second
removes only that backend worktree's development and test databases. Neither
command removes the shared PostgreSQL container, volume, or immutable baseline.

## Important notes

- Quick Tunnel URLs are public, temporary, and for development only.
- Do not use production Turnstile credentials with local Quick Tunnels. The
  local stack supplies Cloudflare's test credentials automatically.
- Do not connect application code to `javamifi_worktree_baseline`. It is the
  read-only source used to create worktree databases.
- Do not run `docker compose down` against the shared database project while
  another worktree may be active.
- `javamifi-localhost down` stops a stack but preserves its runtime state and
  databases; use `cleanup --path` for merged worktree cleanup.
- Schema-changing work is currently blocked until the staging snapshot and
  Prisma migration history are reconciled.

## Troubleshooting

### The command cannot find a repository

Check that the repositories are siblings with the names shown above. Run
`javamifi-localhost up --all --name main` from `javamifi-local`, or pass explicit
paths with `--fe`, `--be`, and `--cms`.

### The database baseline is missing

Restore an approved snapshot from a backend checkout:

```bash
javamifi-db baseline restore /path/to/snapshot.gz javamifi_staging_YYYYMMDD
```

### A Quick Tunnel URL does not work

Run `javamifi-localhost up` again and use the new URL. Quick Tunnels have no
uptime guarantee.

### The backend is unhealthy

Find the stack ID with `javamifi-localhost status`, then inspect the backend
logs:

```bash
javamifi-localhost logs <stack-id> backend
```

### A worktree reports Prisma drift

This is expected with the current staging snapshot. Do not force-reset the
development database or create a migration until the staging schema and
migration history have been reconciled.
