# WireGuard Easy

[WireGuard Easy](https://github.com/wg-easy/wg-easy) is the easiest way to run [WireGuard](https://www.wireguard.com/) VPN + Web-based Admin UI.

Another more powerful alternative for a self-hosted WireGuard VPN server is [Firezone](firezone.md). WireGuard Easy is easier, lighter and more compatible with various ARM devices.


## Dependencies

This service requires the following other services:

- a [Traefik](traefik.md) reverse-proxy server
- a modern Linux kernel which supports WireGuard


## Configuration

To enable this service, add the following configuration to your `vars.yml` file and re-run the [installation](../installing.md) process:

```yaml
########################################################################
#                                                                      #
# wg-easy                                                              #
#                                                                      #
########################################################################

wg_easy_enabled: true

wg_easy_hostname: mash.example.com

wg_easy_path_prefix: /wg-easy

# The default WireGuard port is always 51820 in the container,
# but the advertised port can be configured via the web UI during the initial setup.
# If you'll be using a port other than 51820, uncomment and change the line below
# to make the container publish the port that you've chosen.
# wg_easy_container_wireguard_bind_port: 51820

########################################################################
#                                                                      #
# /wg-easy                                                             #
#                                                                      #
########################################################################
```

### URL

In the example configuration above, we configure the service to be hosted at `https://mash.example.com/wg-easy`.

You can remove the `wg_easy_path_prefix` variable definition, to make it default to `/`, so that the service is served at `https://mash.example.com/`.


### Networking

**In addition** to ports `80` and `443` exposed by the [Traefik](traefik.md) reverse-proxy, the following ports will be exposed by the WireGuard containers on **all network interfaces**:

- `51820` over **UDP**, controlled by `wg_easy_container_wireguard_bind_port` - used for [Wireguard](https://www.wireguard.com/) connections

Docker automatically opens these ports in the server's firewall, so you **likely don't need to do anything**. If you use another firewall in front of the server, you may need to adjust it.

### Additional configuration

The new wg-easy version (after the v15 release) does not support most of the environment variables that were supported in previous versions.
Most of the configuration happens via the web UI after installation. See [Adjusting the post-installation configuration](#adjusting-the-post-installation-configuration) for more details.

Nevertheless, if you need to inject additional environment variables, you can do so with this additional configuration:

```yaml
wg_easy_environment_variables_additional_variables: |
  INSECURE=true
```

💡 Injecting this `INSECURE` environment variable like this is pointless, since the Ansible role provides a dedicated variable for controlling it (`wg_easy_environment_variables_additional_variable_insecure`).


## Usage

After installation, you can go to the wg-easy URL, as defined in `wg_easy_hostname` and `wg_easy_path_prefix`.

It will ask you to sign up with a username and a password and configure your VPN hostname and WireGuard port advertised to clients.
The WireGuard port in the container is always 51820/udp and the exposed port is controlled by `wg_easy_container_wireguard_bind_port` (defaulting to `51820` as well).

### Adjusting the default DNS

Before [creating WireGuard clients](#creating-wireguard-clients) and downloading their configuration profiles, you may wish to go to the Admin Panel -> Config section (`/admin/config` URL path) and adjust the default DNS settings.

wg-easy defaults the global DNS configuration to `1.1.1.1` & `2606:4700:4700::1111`, but you may wish to use your own.

DNS configuration can also be adjusted later on, on a per-client basis, but these changes need to be made before one downloads the WireGuard configuration profile files, because they do hardcode the DNS configuration (and all lots of other configuration) inside them.

### Creating WireGuard clients

You can then create various Clients and import the configuration for them onto your devices - either by downloading a file or by scanning a QR code.

## Recommended other services

- [AdGuard Home](adguard-home.md) - A network-wide DNS software for blocking ads & tracking
