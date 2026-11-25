# Plan

Draft. Nothing is built yet. Each item below is marked as decided, proposed or open. Proposed items are the starting point for development and can still change.

## Why

The stacks on a few Docker hosts live in one Git repository: one directory per host, one directory per stack (a numbered tier such as `50-apps`), each with its Compose file and the configuration files it mounts. A small wrapper script brings a stack up from the checkout on its host. That works, but a change pushed to Git does nothing until someone logs in and runs the script, and nothing shows which hosts still run an older configuration.

Sizing at the time: three hosts and 14 stacks that would be managed, more than 20 mounted configuration directories (alert rules, dashboards, proxy and log shipping configuration, identity provider blueprints), and on one host 8 of 33 containers without a health check.

### Tools looked at first

| Tool | Why not |
| -- | -- |
| doco-cd | The closest fit: an agent per host, no database, and it notices changes in mounted files. It deploys from its own read-only copy of each commit, so its mount paths differ from the checkout the wrapper uses. Switching between it and the manual way recreates every container that mounts a relative path, both ways |
| Komodo | Needs a core and a database running all the time, and each stack defined again in its own format. Pulls images on every push by default, which would step on hikifune's schedule and rollbacks |
| Portainer CE | Notifications are in the paid edition only |
| Arcane | Git sync results are not sent as notifications, and environment files outside the project are refused |
| Dockhand | Not open source (BSL 1.1) |

What was wanted is the shape of Argo CD (applications, sync status, health, sync windows, a history to roll back to) for Compose hosts, deployed exactly the way a person deploys.

## Requirements

### Decided

| ID | Requirement |
| -- | -- |
| R1 | mizusaki deploys with the same command and from the same checkout path a person uses (the repository's own wrapper script, run in the clone on that host). The result of a sync and of a manual run is the same, so falling back to the manual way never recreates containers |
| R2 | The Git repository that already holds the Compose files is the GitOps repository, as it is. It is not split or renamed. mizusaki's own settings go in one directory in it (for example `.cd/`) |
| R3 | An application is one stack on one host. An application set makes them from the `<host>/<stack>` directories. Stacks deploy in the order of their tier numbers, like sync waves |
| R4 | Desired state: `docker compose config --hash` plus a hash of the contents of every mounted file and directory. Live state: the `com.docker.compose.config-hash` label and the state of the containers. An application is Synced or OutOfSync |
| R5 | Health: a container's health check when it has one. Without one, a container that keeps running for a set time (proposed: 2 minutes) with no restart is Healthy, one that keeps restarting or exits non-zero is Degraded, one with `restart: no` that exited 0 is Completed, and a missing one is Missing. Chosen services can add an HTTP or TCP check with a label |
| R6 | Manual and automatic sync, self-heal, prune, a history of deploys, and rollback. Prune is off by default |
| R7 | One server and one agent per Docker host, as in hikifune. The agent keeps the clone and runs the wrapper, since the sources of bind mounts have to be on that host |
| R8 | The server keeps no state of its own. Applications, projects, options and sync windows live in Git; the deploy history and paused applications live with the agents; status is worked out on each request. A second server can run next to the first |
| R9 | Agents lock their stacks, so two syncs of the same stack never run at once. A sync of the commit already deployed does nothing |
| R10 | Hosts pull with a read-only deploy key and never push |
| R11 | Rollback is a revert commit by default. A rollback from the web UI also turns automatic sync off for that application, so the next sync does not undo it |
| R12 | Setting up goes in this order: agent, repository, project, application set, application. The first two carry secrets and live in the server's configuration and the hosts' files. The other three live in Git |
| R13 | Web UI in Go with HTMX, notifications to Discord. Written in Go |
| R14 | Some stacks stay manual only: the host basics (VPN, the agents themselves) and the Docker socket proxy mizusaki goes through. A change to identity provider blueprints alone does not recreate the provider, which applies blueprints from the files itself (marked with a label) |

### Proposed

| ID | Proposal | Why |
| -- | -- | -- |
| P1 | Image tags stay in Git and hikifune keeps updating the images. The other way, digests pinned in Git and hikifune committing the new digest, can come later | Keeps hikifune as it is. hikifune recreates a container with its labels copied and only the image changed, so the configuration hash mizusaki compares does not move |
| P2 | Secrets stay in each host's `.env` file, outside Git. Encrypted secrets in Git (sops with an age key per host) can come later | Nothing has to change on the hosts to start |
| P3 | Only Compose stacks. DNS and tunnel routes, and identity provider configuration, stay where they are managed now | Each has its own tool and source of truth |
| P4 | A sync window around hikifune's update run (for example 00:50 to 01:30) | mizusaki should not sync a stack while hikifune recreates its containers |
| P5 | Polling first, a webhook later | Polling needs nothing exposed |
| P6 | A second server is reached over the tailnet only, with a local account or the tailnet identity | Sign-in through the identity provider would depend on the host the provider runs on |

### Not doing

| Item | Reason |
| -- | -- |
| Canary and blue-green deployments | Out of scope for a few hosts |
| Deploying from a copy of the repository other than the host's checkout | Breaks R1 |

## With hikifune

| | mizusaki | hikifune |
| -- | -- | -- |
| Keeps in line | The Compose configuration and mounted files in Git | The image behind each tag |
| Recreates a container when | Its configuration hash differs from Git | A newer image digest is out |
| Leaves alone | Image updates (P1) | Configuration: it copies everything but the image |
| Runs | On a push to Git, or by hand | On its daily schedule |

The two meet on the same containers, so P4 keeps them apart in time and P1 keeps them apart in what they change.

## Milestones

Proposed order. Each milestone ends when its "Done when" holds. Integration tests run against Docker-in-Docker containers, never the developer's own Docker.

### M0. Skeleton

- `go.mod`, `cmd/mizusaki`, `cmd/mizusaki-agent`, `internal/`, one Dockerfile with a server target and an agent target, `.env.example`.
- GitHub Actions: `go vet`, `go test` and both image builds on every push. Images go to GHCR on tags only.

Done when: CI passes on `dev` and both images build for amd64 and arm64.

### M1. Desired and live state, read only

- The agent keeps a clone, finds its stacks, and works out each stack's desired hash (R4).
- It reads the live state from the containers' labels and states, and the health (R5).
- `mizusaki-agent status` prints every stack as Synced or OutOfSync and its health. Nothing is changed.

Done when: the status matches what a person finds by hand on one host.

### M2. Server and web UI, read only

- The server dials the agents, builds the applications from the application sets, and shows them with their status, health and the diff between Git and the host.

### M3. Manual sync

- Sync one application from the web UI: the agent pulls, takes the lock and runs the wrapper. History is recorded on the agent.

### M4. Automatic sync

- Polling, automatic sync, self-heal, prune, sync windows (P4), rollback (R11).

### M5. Notifications

- Discord messages for syncs, failures and applications that drift.

### M6. Release candidate

- One low-risk stack on one host first, then the rest of that host, then the other hosts.

## Open questions

| # | Question | Notes |
| -- | -- | -- |
| 1 | Images in Git | P1 proposes tags only |
| 2 | Secrets | P2 proposes host `.env` files first |
| 3 | How long a container without a health check must run before it is Healthy | R5 proposes 2 minutes |
| 4 | Polling interval | P5 |
| 5 | Repository layout | Recommended: the server and the agent in this one repository, as in hikifune, so one tag builds both from the same commit |

## Logo

[docs/logo/mizusaki.svg](logo/mizusaki.svg) is the logo for the README and other large sizes: a container ship coming up a channel between red and green buoys. [docs/logo/favicon.svg](logo/favicon.svg) keeps only the ship and one buoy of each color for 32 pixels and below. Both use the frame and colors of hikifune's logo.
