<div align="center">

<img src="docs/logo/mizusaki.svg" alt="" width="200">

# mizusaki

**Keeps the Docker Compose stacks on each host in line with a Git repository, with one agent per host**

</div>

> [!NOTE]
> This is a draft. Nothing is built yet. [The plan](docs/PLAN.md) has the design decisions so far and the open questions.

## What it will do

- Reads one Git repository that already holds the Compose files, with one directory per host and one per stack in it. Each stack on each host becomes an application, the way Argo CD turns a directory into one.
- Shows whether each application matches Git (Synced or OutOfSync) and how its containers run (Healthy, Progressing, Degraded or Missing).
- Deploys with the same command and from the same checkout path a person uses. Running a stack by hand and letting mizusaki run it give the same containers, so falling back to the manual way never recreates anything.
- Notices changes to mounted configuration files, not only to the Compose file, since `docker compose up` alone does not.
- Syncs by hand or on its own, with self-heal, prune (off by default), sync windows that keep it out of the way of scheduled work, a history of what it deployed, and rollback.
- Judges containers without a health check by how long they keep running, whether they restart, and how they exit.
- Runs a small agent on each Docker host and a server that holds the web UI and the Discord notifications. The server keeps no state of its own: what to deploy lives in Git, and the deploy history lives with the agents, so a second server can run next to the first.

## With hikifune

mizusaki is meant to run next to [hikifune](https://github.com/dor1a/hikifune), which updates container images. They split the work the way a harbor does:

| | mizusaki | hikifune |
| -- | -- | -- |
| Keeps in line | The Compose configuration and mounted files in Git | The image behind each tag |
| Recreates a container when | Its configuration differs from Git | A newer image digest is out |
| Leaves alone | Image updates: the tags stay in Git and hikifune swaps the images | Configuration: it copies everything but the image |

A sync window keeps mizusaki from syncing while hikifune runs its updates.

## Name

Mizusaki (水先) is Japanese for pilotage, and a harbor pilot is a mizusakinin (水先人). The pilot brings a ship through the buoyed channel into port, and a tug (hikifune) pulls it to its berth.

## License

[MIT](LICENSE)
