# Project Zomboid Docker

Run a [Project Zomboid](https://projectzomboid.com/) dedicated server with Docker Compose.

The game server files are installed and updated with [SteamCMD](https://developer.valvesoftware.com/wiki/SteamCMD)
using the [`cm2network/steamcmd`](https://hub.docker.com/r/cm2network/steamcmd) image. Both the server files
(`./server` by default) and the game data (`./data` by default: worlds, configs, mods and logs) live on the host, so
they persist between container restarts and updates.

## Requirements

- Linux host with [Docker](https://docs.docker.com/engine/install/) and the
  [Docker Compose plugin](https://docs.docker.com/compose/install/) (`docker compose`, v2.23+ for inline `configs`)
- `bash` and `sudo` (the `update` script may need to fix ownership of the server and data folders)
- Enough disk space for the server files (a few GB)

## Quickstart

```sh
# 1. Clone the repository
git clone https://github.com/GDWR/project-zomboid-docker.git
cd project-zomboid-docker

# 2. Set an admin password (required) - uncomment and edit in .env
#    ADMIN_PASSWORD=change-me

# 3. Install the server files via SteamCMD
./update

# 4. Start the server
docker compose up -d

# 5. Follow the logs (first start generates the world and can take a while)
docker compose logs -f
```

Then connect from the game via **Join → Add server** using your host's IP and port `16261`.

## Configuration

All settings are in [`.env`](.env):

| Variable         | Default    | Description                                                                                       |
| ---------------- | ---------- | ------------------------------------------------------------------------------------------------- |
| `ADMIN_PASSWORD` | _(none)_   | **Required.** Password for the in-game admin account.                                             |
| `ADMIN_USERNAME` | `admin`    | Username for the in-game admin account.                                                           |
| `SERVER_NAME`    | `MyWorld`  | Name of the server/save. Determines which config and save files are used.                         |
| `SERVER_PORT`    | `16261`    | Host UDP port mapped to the game server.                                                          |
| `SERVER_FILES`   | `./server` | Host folder the `update` script installs server files into.                                       |
| `DATA_FILES`     | `./data`   | Host folder for game data (saves, server configs, mods, logs), mounted at `/home/steam/Zomboid`.  |
| `JVM_ARGS`       | _(none)_   | Extra JVM arguments, e.g. `-Xmx8g` to raise memory. See [JVM arguments][jvm].                     |

See the wiki for more [server startup parameters][params].

[params]: https://pzwiki.net/wiki/Startup_parameters#Server
[jvm]: https://pzwiki.net/wiki/Startup_parameters#JVM_arguments

### Server settings and mods

After the first start, Project Zomboid writes its config files to the data folder, in `./data/Server/` (e.g.
`<SERVER_NAME>.ini` and `<SERVER_NAME>_SandboxVars.lua`). Saves are under `./data/Saves/`. Edit configs while the
server is stopped, then start it again. Details on available settings are on the
[PZ wiki](https://pzwiki.net/wiki/Server_settings).

## Usage

| Task                 | Command                     |
| -------------------- | --------------------------- |
| Start (background)   | `docker compose up -d`      |
| View logs            | `docker compose logs -f`    |
| Stop                 | `docker compose down`       |
| Update server files  | `docker compose down && ./update` |

### Updating

The `update` script re-runs SteamCMD to download the latest server build (and validate existing files). It refuses to
run while the server is up, so stop it first:

```sh
docker compose down
./update
docker compose up -d
```

## How it works

- [`compose.yml`](compose.yml) defines a single `project-zomboid` service. It mounts the server folder at `/opt/server`
  and the data folder at `/home/steam/Zomboid` (where the game stores its saves and configs), and launches the game's
  own `start-server.sh` with the name and admin credentials from `.env`.
- An inline Compose config, `steamcmd-script`, holds the SteamCMD script that anonymously installs app `380870`
  (Project Zomboid Dedicated Server) into `/opt/server`.
- [`update`](update) creates the server and data folders, makes sure they're owned by UID/GID `1000` (the `steam`
  user in the container), then runs SteamCMD with that script in a one-off container.

## Troubleshooting

- **`Admin password required, edit the .env file`**: set `ADMIN_PASSWORD` in `.env`.
- **`start-server.sh: No such file or directory`**: the server files aren't installed yet, run `./update`.
- **Permission errors in the server or data folder**: both must be owned by `1000:1000`. Running `./update` fixes
  this, or run `sudo chown -R 1000:1000 ./server ./data`.
- **Players can't connect**: make sure UDP port `16261` (or your `SERVER_PORT`) is open in your firewall and
  forwarded on your router.
