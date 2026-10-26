# dokku-stats

A [Dokku](https://dokku.com) plugin that reports container resource usage (CPU, memory, disk) per app, plus Docker disk usage and cleanup.

Dokku has no built-in command for this, and the `dokku` SSH user can't run `docker` directly. This plugin exposes the data as `dokku stats:*` commands, so it is available over the regular Dokku SSH connection. It was built for a web dashboard, so the output is machine-readable rather than pretty.

## Requirements

- Dokku
- A Debian or Ubuntu host (uses GNU `du`, `df` and `free`)

## Installation

```bash
sudo dokku plugin:install https://github.com/IamMattco/dokku-stats.git --name dokku-stats
```

Update:

```bash
sudo dokku plugin:update dokku-stats
```

## Commands

```
stats:report [<app>]                 Resource usage of all containers, or of a single app
stats:df                             Docker disk usage by type
stats:services                       Data size of service containers (postgres, mysql, redis...)
stats:prune <build-cache|images>     Remove Docker build cache or dangling images
```

## Output format

One record per line, fields separated by `|`, the first field is the record type. Sizes from Docker are human readable (`256MiB`, `1.2GB`), everything measured by the plugin itself is in bytes.

### `stats:report [<app>]`

```
host|<cpus>|<load1>|<mem_total>|<mem_used>|<disk_total>|<disk_used>
container|<id>|<name>|<image>|<state>|<status>|<size>|<app>|<process_type>
usage|<id>|<cpu%>|<mem_usage>|<mem%>|<net_io>|<block_io>|<pids>
disk|<app>|<repo_bytes>|<storage_bytes>
```

Example:

```
$ dokku stats:report my-app
host|4|0.52|8126812160|6838951936|77309411328|24696061952
container|3f44450264ad|my-app.web.1|dokku/my-app:latest|running|Up 3 days|2.05kB (virtual 340MB)|my-app|web
usage|3f44450264ad|1.73%|335MiB / 7.57GiB|4.32%|9.64GB / 15.9GB|22.5MB / 12.3kB|21
disk|my-app|52428800|2097152
```

- `host`: memory used is total minus available, disk is the filesystem holding `/var/lib/docker`.
- `container`: without an app, every container on the host is listed, including databases and other services. `<size>` is the writable layer plus the image size (`virtual`).
- `usage`: one line per **running** container, from `docker stats --no-stream`.
- `disk`: `repo_bytes` is the app's git repo and build cache in `$DOKKU_ROOT/<app>`. `storage_bytes` is persistent storage mounted from `/var/lib/dokku/data/storage`. Other bind mounts are skipped on purpose, so mounting `/` into a container can't make the report walk the whole disk.

`docker stats --no-stream` takes about 2 seconds, so cache the result if you call it often.

### `stats:df`

```
df|<type>|<total>|<active>|<size>|<reclaimable>
```

The same data as `docker system df`: Images, Containers, Local Volumes, Build Cache.

### `stats:services`

```
service|<plugin>|<name>|<container_id>|<data_bytes>
```

Covers running containers named `dokku.<plugin>.<name>`, the convention of the official service plugins. Data size is measured with `du` **inside** the container, because the data directories on the host belong to the service user and the `dokku` user can't read them. That means one `docker exec` per service, which can take a few seconds with large databases.

### `stats:prune <build-cache|images>`

- `build-cache`: `docker builder prune -af`
- `images`: `docker image prune -f`, **dangling images only**. Images of stopped apps are kept, so `ps:start` still works without a rebuild.

## License

[MIT](LICENSE)
