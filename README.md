# firefox-sync

Self-host Mozilla Firefox Sync with docker-compose — syncstorage-rs and the
tokenserver stack backed by **MariaDB** instead of Google Spanner.

[![GitHub last commit](https://img.shields.io/github/last-commit/tmiland/firefox-sync/main)](https://github.com/tmiland/firefox-sync/commits/main)
[![Images on GHCR](https://img.shields.io/badge/images-ghcr.io%2Ftmiland%2Ffirefox--sync-2496ED?logo=github)](https://github.com/tmiland/firefox-sync/pkgs/container/firefox-sync/versions)

> **This is a maintained fork of [porelli/firefox-sync](https://github.com/porelli/firefox-sync).**
> It carries two fixes not yet merged upstream and publishes production-tested
> images — everything good here originates upstream; see [Credits](#credits).

## What's different in this fork

- **Fresh deployments work** — upstream's `db_init.sh` hardcoded
  `nodes.service = '1'`, but the migrations guarantee `sync-1.5` lands at
  id **3** on every fresh database, so *no* login could ever resolve a node
  (503 "unable to get a node"). The service id is now resolved dynamically —
  upstream PR [#23](https://github.com/porelli/firefox-sync/pull/23).
- **syncstorage-rs 0.23+ builds correctly** — upstream renamed
  `DATABASE_BACKEND` to `SYNCSTORAGE_DATABASE_BACKEND` /
  `TOKENSERVER_DATABASE_BACKEND` (defaulting to Spanner); the image workflow
  now passes both, so 0.23 images don't silently build for Spanner and panic
  on MySQL URLs — upstream PR
  [#24](https://github.com/porelli/firefox-sync/pull/24).
- **Images published under
  [ghcr.io/tmiland/firefox-sync](https://github.com/tmiland/firefox-sync/pkgs/container/firefox-sync/versions)**
  — weekly rebuilds against the latest upstream tag, running in production.

## Architecture

| Service | Role |
|---------|------|
| `syncstorage` | syncstorage-rs (MySQL backend). Since 0.23 the tokenserver is embedded and both migration sets run at startup |
| `syncstorage_db` | MariaDB for BSO storage (bookmarks, passwords, tabs, history…) |
| `tokenserver_db` | MariaDB for the tokenserver (OAuth token → node mapping, user allowlist) |
| `tokenserver_db_init` | one-shot init: waits for syncstorage to be healthy, then seeds `services`/`nodes` (dynamically resolved ids) and the `MAX_USERS` trigger — `Exited (0)` is success |

`syncstorage` listens internally on port 8000; a reverse proxy is expected in
front (nginx example in [config/nginx](/config/nginx/syncstorage-rs-example.conf)).
Health probe: `GET /__heartbeat__`.

## Disclaimer

- ⚠️ The project is under development
- ⚠️ Expect bugs, breaking changes and headache
- ⚠️ **This is not endorsed or supported by Mozilla in any way or form**
- ⚠️ **Do not solely relay on this project to store your bookmarks, passwords and/or other important items**

## Security considerations

1. ~~syncstorage-rs does NOT support account allowlisting. This means that ANY person that has network access to your server can use it.~~
    - **This has been implemented with a SQL trigger workaround that prevents the token database to insert more rows when a new user tries to use the server. This is tested and prevents a new account from using your server if the number of MAX_USERS defined in your .env file is already reached.**
    - Possible alternative (better) solutions:
        - implement the feature directly in syncstorage-rs
        - add the entire rest of the Mozilla stack so that authentication is performed and validated locally

## Background

Mozilla's server side components are open source and Firefox allows to easily change the official endpoints.
[Some documentation](https://mozilla-services.readthedocs.io/en/latest/index.html) is provided to install each component on your own server but this is neither officially supported or very well maintained. Furthermore, there are no official Docker images that can be used to avoid installing everything manually; all the instructions and artifacts provided are focused on setting up a developer environment rather than a production self hosted service. For example, `syncstorage-rs` has a [docker release on docker-hub](https://hub.docker.com/r/mozilla/syncstorage-rs/) but it is targeted to work exclusively with Google Spanner which is what Mozilla uses to provide the service.

## Images

Both images are rebuilt weekly against the latest tag from Mozilla's official
repositories and published to
[ghcr.io/tmiland/firefox-sync](https://github.com/tmiland/firefox-sync/pkgs/container/firefox-sync/versions)
(workflow: [syncstorage-rs.yml](/.github/workflows/syncstorage-rs.yml)):

- `syncstorage-rs-mysql-latest` — [Mozilla's](https://github.com/mozilla-services/syncstorage-rs/blob/master/Dockerfile) container built with `SYNCSTORAGE_DATABASE_BACKEND=mysql` and `TOKENSERVER_DATABASE_BACKEND=mysql` (both old and new arg names are passed, so pre-0.23 tags keep working). Code changes: none. (arm64: the upstream Dockerfile is patched to use `libmariadb-dev-compat`.)
- `syncstorage-rs-mysql-init-latest` — [MariaDB's](https://github.com/MariaDB/mariadb-docker/blob/master/Dockerfile.template) container with the [db_init.sh](/syncstorage-rs-init/db_init.sh) seeding script (dynamic service-id resolution). Code changes: none.

## Server setup

1. clone this repository
1. run `./prepare_environment.sh` to automatically prepare your `.env` file and conf examples according with your variables (`REPOSITORY` defaults to `ghcr.io/tmiland/firefox-sync`)
1. setup your reverse proxy server
    1. if you use nginx, check the [syncstorage-rs.conf](/config/nginx/syncstorage-rs-example.conf) as example
1. start docker compose: `docker compose up -d`
1. verify: `curl https://your-domain/__heartbeat__` returns `200` and `tokenserver_db_init` shows `Exited (0)`
1. OPTIONAL: Install the systemd service (see [firefox-sync.service](/config/systemd/syncstorage-rs-example.service)) and enable it
    - all the containers are already set to restart automatically; stopping Docker (for example when you shutdown your computer) will automatically stop all the services gracefully and restart them once Docker is starting again

## Firefox setup

**Pre-requisite**: if you already logged into your Firefox account you need to temporarily disconnect it

The below examples assume your server respond to this domain: `firefox-sync.example.com`

### Desktop

1. point a browser tab to `about:config` and search for `identity.sync.tokenserver.uri`
1. change it from the default to `https://firefox-sync.example.com/1.0/sync/1.5`
1. log in to Firefox and start syncing.

#### Debug

1. check logs pointing a browser tab to `about:sync-log`

### Android

1. go to App Menu `⋮` > `Settings` > `About Firefox` and click the logo 5 times. You should see a `debug menu enabled` notification
1. go back to the main setting menu and you will see `Sync Debug` at the top, just under the `Synchronize and save your data` box. Tap on it
1. tap on `Custom Sync server` and set it to `https://firefox-sync.example.com/1.0/sync/1.5`
1. log in to Firefox and start syncing.

### iOS

1. go to App Menu `≡` > `Settings` and tap 5 times on the version number (i.e.: `Firefox 127.1 (42781)`) towards the bottom
1. go back at the top of the main setting menu and you will see `Advanced Sync Settings` at the top, just under the `Sync and Save Data`. Tap on it
1. activate `Use Custom FxA Content Server` and set `Custom Account Content Server URI` to `https://firefox-sync.example.com/1.0/sync/1.5`
1. log in to Firefox and start syncing.

## Troubleshooting

- **503 "unable to get a node" on login** — you're on an init image with the hardcoded service id. Pull the latest `syncstorage-rs-mysql-init-latest`, re-run `docker compose up -d` to recreate the init container, and (once) wipe the tokenserver DB volume so it re-seeds — see upstream PR [#23](https://github.com/porelli/firefox-sync/pull/23).
- **Panic `Invalid database url: mysql://…` on syncstorage-rs 0.23** — the image was built for Spanner (old build-args). Pull the latest image built by this fork's workflow — [#24](https://github.com/porelli/firefox-sync/pull/24).
- **Heartbeat check** — the health endpoint is `GET /__heartbeat__` (200), not `/1.0/sync/1.5/heartbeat`.

## Credits

- [porelli](https://github.com/porelli) for [porelli/firefox-sync](https://github.com/porelli/firefox-sync) — this fork's base; fixes are proposed upstream as [#23](https://github.com/porelli/firefox-sync/pull/23) and [#24](https://github.com/porelli/firefox-sync/pull/24)
- [Mozilla](https://www.mozilla.org/) for [Firefox](https://www.mozilla.org/firefox) and opensourcing all their software, including the backend
- [jeena](https://github.com/jeena) for [fxsync-docker](https://github.com/jeena/fxsync-docker) which is the inspiration for this project

## License

GPL-3.0 — inherited from upstream.
