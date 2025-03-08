# WireGuard Easy

[WireGuard Easy](https://github.com/wg-easy/wg-easy) is the easiest way to run [WireGuard](https://www.wireguard.com/) VPN + Web-based Admin UI.

Another more powerful alternative for a self-hosted WireGuard VPN server is [Firezone](firezone.md). WireGuard Easy is easier, lighter and more compatible with various ARM devices.


## Dependencies

This service requires the following other services:

- a [Traefik](traefik.md) reverse-proxy server
- a modern Linux kernel which supports WireGuard
- `devture_systemd_docker_base_ipv6_enabled: true` if you'd like IPv6 support


## Configuration

To enable this service, add the following configuration to your `vars.yml` file and re-run the [installation](../installing.md) process:

```yaml
########################################################################
#                                                                      #
# wg-easy                                                              #
#                                                                      #
########################################################################

wg_easy_enabled: true

wg_easy_hostname: wg-easy.example.com

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

In the example configuration above, we configure the service to be hosted at `https://wg-easy.example.com/`.

Previously (prior to wg-easy v15), a `wg_easy_path_prefix` variable could allow you to host wg-easy at a subpath (e.g. `wg_easy_path_prefix: /wg-easy`), to this is [no longer possible](https://github.com/wg-easy/wg-easy/issues/1704#issuecomment-2705873936) and such a feature [may re-appear later](https://github.com/wg-easy/wg-easy/issues/1704#issuecomment-2706575504).


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

After installation, you can go to the wg-easy [URL](#url).

It will ask you to sign up with a username and a password and configure your VPN hostname and WireGuard port advertised to clients.
The WireGuard port in the container is always 51820/udp and the exposed port is controlled by `wg_easy_container_wireguard_bind_port` (defaulting to `51820` as well).

💡 Before you [create clients](#creating-wireguard-clients), we recommend that you adjust a few global settings first:

- [Adjusting the default DNS](#adjusting-the-default-dns)
- [Adjusting the IPv6 CIDR for full IPv6 connectivity](#adjusting-the-ipv6-cidr-for-full-ipv6-connectivity)

### Adjusting the default DNS

Before [creating WireGuard clients](#creating-wireguard-clients) and downloading their configuration profiles, you may wish to go to the Admin Panel -> Config section (`/admin/config` URL path) and adjust the default DNS settings.

wg-easy defaults the global DNS configuration to `1.1.1.1` & `2606:4700:4700::1111`, but you may wish to use your own.

DNS configuration can also be adjusted later on, on a per-client basis, but these changes need to be made before one downloads the WireGuard configuration profile files, because they do hardcode the DNS configuration (and all lots of other configuration) inside them.

### Adjusting the IPv6 CIDR for full IPv6 connectivity

💡 For IPv6 to work, you need your container networks to have been created with IPv6 support (`devture_systemd_docker_base_ipv6_enabled: true`) as mentioned in the [Prerequisites](#prerequisites). If your wg-easy container network (`mash-wg-easy`) was created before you flipped this setting to `true`, you may need to stop the wg-easy service, delete the container network manually (`docker network rm mash-wg-easy`) and re-run the playbook to have it create the container network anew.

By default, wg-easy uses a [default CIDR value](https://github.com/wg-easy/wg-easy/blob/0597470f4cea239b4a572208ef01d490e2ade2d2/src/server/database/migrations/0001_classy_the_stranger.sql#L6) of `fdcc:ad94:bacf:61a4::cafe:0/112`, so all your clients will receive an IPv6 address from this subnet.

> [!WARNING]
> The default CIDR value provides a [Unique Local Address (ULA)](https://en.wikipedia.org/wiki/Unique_local_address), not a [Global Unicast Address (GUA)](https://www.oreilly.com/library/view/ipv6-fundamentals-a/9780134670584/ch05.html).

Most operating systems, when dealing with a network interface with a ULA address (instead of a GUA address), will prefer IPv4 instead of IPv6 for outgoing connections.

This means that you may get a `10/10` score on the [test-ipv6.com](https://test-ipv6.com/) test, but you'll also see a warning message like this:

> [!WARNING]
> Your browser has real working IPv6 address - but is avoiding using it. We're concerned about this. [more info](https://test-ipv6.com/faq_avoids_ipv6.html)

If this is alright with you, you can leave things like that.

We believe that just like IPv6 is typically preferred over IPv4 (when without using a VPN), the same should happen when using a VPN. If it doesn't, then you don't really have "proper IPv6" connectivity, because your client is avoiding it IPv6 and favoring IPv4. In this day and age, most services on the internet are either IPv4-only or dual-stack (and less often IPv6 only), so a client that prefers IPv4 will pretty much always stick to IPv4 in practice.

📖 This issue is also discussed [here](https://www.reddit.com/r/WireGuard/comments/q8t9bj/wireguard_doesnt_seem_to_work_with_ipv6/) and a workaround is documented [here](https://www.reddit.com/r/ipv6/comments/ngug1e/comment/gyw1ni8/). We'll summarize these findings and propose alternatives below.

To get full IPv6 connectivity and have it be preferred on most operating systems, you'll:

- either need to fiddle with the `/etc/gai.conf` file and make your operating system prefer IPv6 even when dealing with a ULA address. This is difficult, requires manual work and is potentially impossible on certain systems (iOS)

- or you fix this globally by switching from ULA addresses to GUA addresses

To do the latter, **you need to change the IPv6 CIDR used by wg-easy** to a GUA or GUA-like one. The proposed workaround [here](https://www.reddit.com/r/ipv6/comments/ngug1e/comment/gyw1ni8/) suggests using a random real/global/public IPv6 subnet for your WireGuard clients. While these addresses will only be used inside your WireGuard for NAT purposes, using them will still break your connectivity to this subnet.

We propose 2 alternatives for your IPv6 CIDR:

- (recommended) a CIDR derived from a GUA one that you own. For example, if you get `2001:555:5555:5555:/64` from your ISP for your network, you could assign something like `2001:555:5555:5555::cafe:0/112` for your 1st (`0`-th) wg-easy instance. If you run more wg-easy instances in your network, you could use `2001:555:5555:5555::cafe:1/112`, `2001:555:5555:5555::cafe:2/112`, etc. for them, so that they all have unique subnets.

- (not so recommended) [a GUA-like CIDR](https://en.wikipedia.org/wiki/IPv6_address#Special-purpose_addresses). We've had success using the documentation-reserved CIDR (`2001:db8::/32`) as per [RFC 3849](https://datatracker.ietf.org/doc/html/rfc3849). You can pick a random CIDR value from this range, like `2001:db8:100:5000::cafe:0/112` and use it for your 1st (`0`-th) wg-easy instance. If you run more wg-easy instances in your network, you could use `2001:db8:100:5000::cafe:1/112`, `2001:db8:100:5000::cafe:2/112`, etc. for them, so that they all have unique subnets. Using documentation-reserved addresses for non-documentation-purposes is not great, but we've confirmed that it works well in practice. These addresses are only behind your NAT and never used for real addressing outside of it, so there shouldn't be a problem.

Regardless of what you choose, your WireGuard clients will get a network interface which uses a GUA or GUA-like IPv6 address instead of a ULA IPv6 address. With that, IPv6 connectivity will be preferred over IPv4 even without custom changes to `/etc/gai.conf`.


### Creating WireGuard clients

You can then create various Clients and import the configuration for them onto your devices - either by downloading a file or by scanning a QR code.

## Recommended other services

- [AdGuard Home](adguard-home.md) - A network-wide DNS software for blocking ads & tracking
