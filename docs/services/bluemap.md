<!--
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# BlueMap

The playbook can install and configure [BlueMap](https://bluemap.bluecolored.de/) for you.

BlueMap reads Minecraft world files and renders them into 3D models of the whole surface, which its built-in web server then serves for viewing in a browser.

See the project's [documentation](https://bluemap.bluecolored.de/) to learn what BlueMap does and why it might be useful to you.

> [!NOTE]
> The role for this service ships with the playbook itself (in `roles/mash/bluemap`), so there is no separate role repository with its own documentation.

## Deployment modes

This page describes running BlueMap as a standalone service, with its own container and web server.

BlueMap can also run as a [Minecraft](minecraft.md) plugin inside the Minecraft container. To do that instead, install it there (for example through `minecraft_environment_variables_additional_variables` / `MODRINTH_PROJECTS`) and expose its web interface with `minecraft_web_interfaces` (see the "Exposing plugin web interfaces" section on the [Minecraft](minecraft.md) page). Keep `bluemap_enabled: false` in that case so no separate container is started.

## Dependencies

This service requires the following other services:

- [Traefik](traefik.md) reverse-proxy server

## Configuration

To enable this service, add the following configuration to your `vars.yml` file and re-run the [installation](../installing.md) process:

```yaml
########################################################################
#                                                                      #
# bluemap                                                              #
#                                                                      #
########################################################################

bluemap_enabled: true

bluemap_hostname: bluemap.example.com

########################################################################
#                                                                      #
# /bluemap                                                             #
#                                                                      #
########################################################################
```

### Configuring the worlds to render

BlueMap renders nothing without a world to read. When the [Minecraft](minecraft.md) service is enabled, its overworld, Nether and End directories are mounted for you automatically. For worlds managed elsewhere, list them yourself:

```yaml
bluemap_worlds_custom:
  - name: world
    path: /srv/minecraft/world
```

Each entry is mounted read-only into the container at `/app/<name>`, which is where BlueMap's default map configuration looks for it. Names other than `world`, `world_nether` and `world_the_end` need a matching map configuration in BlueMap.

Only worlds whose host directory already exists when the service is installed get mounted, because Docker refuses to bind-mount a missing path. BlueMap therefore starts before Minecraft has generated every world; re-run the installation to pick up world folders created later. Do not create empty world folders yourself — Minecraft treats a leftover folder such as `world_nether` as a world to migrate and may refuse to start.

### Exposing the service without Traefik

By default the webserver is only reachable through [Traefik](traefik.md). To publish the port directly on the host instead, set `bluemap_container_http_host_bind_port` (e.g. `127.0.0.1:8100`) and disable the Traefik labels with `bluemap_container_labels_traefik_enabled: false`.

## Usage

After running the command for installation, the BlueMap instance becomes available at the URL specified with `bluemap_hostname`. With the configuration above, the service is hosted at `https://bluemap.example.com`.

### Accept the Mojang resource download

BlueMap needs some resource files from Mojang to texture its models, and refuses to download them until you agree to it. To accept from the playbook, set `bluemap_accept_download: true` in your `vars.yml` and re-run the installation. The role manages the `accept-download` line in `<bluemap_base_path>/config/core.conf` (`/mash/bluemap/config` with the default settings), leaving any other settings in that file alone.

On the first start BlueMap writes its remaining default configuration files (including the map configurations) into the same directory. Review them there if you want to change which worlds are rendered or how. The setting defaults to `false` to match BlueMap's own zero-consent default; you can also flip it by editing `core.conf` directly and restarting the service with `sudo systemctl restart <bluemap_identifier>.service` (by default `mash-bluemap.service`) or by running `just install-service bluemap` again.

The pinned `bluemap_version` should always be a specific release, because BlueMap releases occasionally require manual upgrades — see the [release notes](https://github.com/BlueMap-Minecraft/BlueMap/releases) for anything you need to do when bumping it.

>[!NOTE]
> The `bluemap_path_prefix` variable can be adjusted to host under a subpath (e.g. `bluemap_path_prefix: /bluemap`), but this hasn't been tested yet.

## Removing BlueMap

Set `bluemap_enabled: false` and re-run the installation to remove the service. The systemd unit, the `env`/`labels` support files and the container are removed, but BlueMap's base path and its configuration are kept by default.

To also delete the generated data — rendered maps (`bluemap_data_path`) and web assets (`bluemap_web_path`) — set:

```yaml
bluemap_force_uninstall_cleanup: true
```

The configuration under `bluemap_config_path` is never deleted automatically. Delete the remaining `bluemap_base_path` directory manually to remove it.

## Related services

- [Minecraft](minecraft.md) — whose worlds are rendered by default
