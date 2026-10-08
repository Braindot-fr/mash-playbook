<!--
SPDX-FileCopyrightText: 2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2025 XHawk87
SPDX-FileCopyrightText: 2025, 2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Minecraft Server on Docker (Java Edition)

The playbook can install and configure [itzg docker-minecraft server](https://docker-minecraft-server.readthedocs.io) for you.

Minecraft is a first-person open-world procedurally-generated voxel-based sandbox game with RPG elements. The Docker image which this playbook installs provides a Minecraft Server which will automatically download the latest stable version at startup.

See the project's [documentation](https://docker-minecraft-server.readthedocs.io/en/latest/) to learn what it does and why it might be useful to you.

> [!WARNING]
> itzg docker-minecraft server is published under the Apache-2.0 license, however Minecraft itself is proprietary software, and by using this role you are agreeing to the [EULA](https://www.minecraft.net/en-us/eula). Know your rights!

The [Ansible role for Minecraft Server](https://github.com/mother-of-all-self-hosting/ansible-role-minecraft) is developed and maintained by the MASH project. For details about configuring Minecraft Server, you can check them via:

- 🌐 [the role's documentation](https://github.com/mother-of-all-self-hosting/ansible-role-minecraft/blob/main/docs/configuring-minecraft.md) online
- 📁 `roles/galaxy/minecraft/docs/configuring-minecraft.md` locally, if you have [fetched the Ansible roles](../installing.md)

## Restoring from a backup

The playbook can import a tarball of the Minecraft Server directory before installing or updating the service:

1. Create the tarball of the `minecraft` base directory (e.g. `/mash/minecraft` by default) on the old server:

   ```sh
   tar -C /mash -cf minecraft.tar minecraft
   ```

2. Put the tarball on the machine you run this playbook from (e.g. into `inventory/`, which is ignored by git) and set in your `vars.yml`:

   ```yaml
   minecraft_restore_tarball_path: "{{ playbook_dir }}/inventory/minecraft.tar"
   ```

3. Install (or update) Minecraft as usual, e.g. `just install-all`.

The import stops the service (if it is running), replaces `minecraft_data_path` with the tarball contents, fixes ownership and then lets the normal installation regenerate `env` and `labels` from your inventory configuration. Any `minecraft_environment_variables_*` settings you change afterwards are applied by subsequent runs as well.

The import runs only once: it records `/mash/minecraft/.restore-completed` and later playbook runs skip it, even if `minecraft_restore_tarball_path` stays set. Delete that marker file if you want to import the tarball again.

> [!NOTE]
> The tarball contents are taken as-is. If the backup was not made from a Vanilla server on the latest version (e.g. Paper 1.21.11), set `minecraft_environment_variables_type` and `minecraft_environment_variables_version` accordingly, or the server will start with different settings than the backup had.

Once Minecraft has started with the imported worlds, restart BlueMap (`sudo systemctl restart mash-bluemap.service`) so that it renders them.
