---
title: "Private Networking"
layout: docs
sitemap: false
nav: firecracker
redirect_from:
  - /docs/reference/privatenetwork/
  - /docs/reference/private-networking/
---

Fly Apps are connected by a mesh of WireGuard tunnels using IPv6.

Applications within the same organization are assigned special addresses (6PN addresses) tied to the organization. Those applications can talk to each other because of those 6PN addresses, but applications from other organizations can't; the Fly.io platform won't forward between different 6PN networks.

This connectivity is always available to apps; you don't have to do anything special to get it.

You can connect apps running outside of Fly.io to your 6PN network using WireGuard; for that matter, you can connect your dev laptop to your 6PN network. To do that, you'll use flyctl, the Fly.io CLI, to generate a WireGuard configuration that has a 6PN address.

## Discover apps through DNS on a Fly Machine

An app's Fly Machines are configured with their DNS server pointing to `fdaa::3`. The DNS server on this address can resolve arbitrary DNS queries, so you can look up "google.com" with it. But it's also aware of 6PN addresses, and, when queried from a Machine, will let you look up the addresses of other apps in your organization. Those addresses live under the synthetic top-level domain `.internal`.

Since this is the default configuration we set up for Machines on Fly.io, you probably don't need to do anything special to make this work; if your app shares an organization with another app called `random-potato-45`, then you should be able to `ping6 random-potato-45.internal`.

If you want to get fancy, you can install `dig` on the Machine and query the DNS directly. For example:

```cmd
root@f066b83b:/# dig +short aaaa my-app-name.internal @fdaa::3
```
```output
fdaa:0:18:a7b:7d:f066:b83b:2
```

## Discover apps through DNS on a WireGuard connection

The DNS server address is different on WireGuard connections than on Machines. That's because you can run multiple WireGuard connections; your dev laptop could be WireGuard-connected to multiple organizations, but a Machine can't be.

Your DNS server address for a WireGuard connection is part of the WireGuard tunnel configuration that flyctl generates. Your platform WireGuard tools might read and automatically configure DNS from that configuration, or they might not. Here's an example of a WireGuard configuration file generated by the `fly wireguard create` command:

```yaml
[Interface]
PrivateKey = [redacted]
Address = fdaa:0:18:a7b:d6b:0:a:2/120
DNS = fdaa:0:18::3
```

You guessed it; it's the `DNS` line. Your DNS server address will also start with `fdaa:`, but the next two parts are unique to your organization's network. All 6PN addresses are prefixed by the organization's network ID; that's the part of the address that locks it to your organization.

All our WireGuard DNS addresses follow this pattern: take the organization prefix, and tack `::3` onto the end. In this example, the Wireguard peer address is:

```
fdaa:0:18:a7b:d6b:0:a:2
```

The 6PN prefix is the first 3 `:`-separated parts, so the DNS server address for this example is:

```
fdaa:0:18::3
```

To use `dig` to probe DNS on a WireGuard connection, supply the DNS server address to it. For example:

```cmd
root@f066b83b:/# dig +short aaaa my-app-name.internal @fdaa:0:18::3
```
```output
fdaa:0:18:a7b:7d:f066:b83b:2
```
<div class="note icon">
<b>Note:</b>`dig`'s syntax requires a `@` at the beginning of the address; this trips us up all the time.
</div>

## Connect to a running service via its 6PN address

In the `/etc/hosts` of a deployed Fly Machine, we alias the 6PN address of the Machine to `fly-local-6pn`.  

For a service to be accessible via its 6PN address, it needs to bind to/listen on `fly-local-6pn`. For example, if you have a service running on port 8080, then you need to bind it to `fly-local-6pn:8080` for it to be accessible at "[6PN_Address:8080]". 

<div class="note icon">
`fly-local-6pn` is to 6PN addresses  as `localhost` is to 127.0.0.1, so you can also bind directly to the 6PN address itself, that's also fine.
</div>

Learn more about [connecting to app services](/docs/networking/app-services/).

## Fly.io `.internal` addresses

Fly.io `.internal` hostnames resolve to 6PN addresses (internal IPv6 addresses) associated with Fly Machines. You can use `.internal` addresses to connect to apps and Machines in your 6PN network by doing a DNS lookup for specific 6PN addresses and then using those addresses to send requests. You might want to use `.internal` addresses to connect your app to databases, API servers, or other apps in your 6PN network.

<div class="important icon">
**Important:** Queries to Fly.io `.internal` hostnames only return information for started (running) Machines. Any stopped Machines, including those auto stopped by the Fly Proxy, are not included in the response to the DNS query.
</div>

The `.internal` addresses can include qualifiers to return more specific addresses or info. For example, you can add a region name qualifier to return the 6PN addresses of an app's Machines in a specific region: `iad.my-app-name.internal`. Querying this hostname returns the 6PN address (or addresses) of the `my-app-name` Machines in the `iad` region. 

Some `.internal` hostnames return a TXT record with Machine, app, or region information. For example, if you request the TXT records using `regions.my-app-name.internal`, then you'll get back a comma-separated list of regions that `my-app-name` is deployed in. And you can discover all the apps in the organization by requesting the TXT records associated with `_apps.internal`. This will return a comma-separated list of the app names.

The following table describes the information returned by each form of `.internal` address.

| Name | AAAA | TXT |
| -- | --- | -- |
|`<appname>.internal`|6PN addresses of all<br> Machines in any<br> region for the app|none
|`top<number>.nearest.of.<appname>.internal`|6PN addresses of<br> top _number_ closest<br> Machines for the app|none
|`<machine_id>.vm.<appname>.internal`|6PN address of<br> a specific Machine<br> for the app|none
|`vms.<appname>.internal`|none|comma-separated list<br> of Machine ID and region<br>name for the app
|`<region>.<appname>.internal`|6PN addresses of<br> Machines in region<br> for the app|none
|`<process_group>.process.<appname>.internal`|6PN addresses of<br> Machines in process<br> group for the app|none
|`global.<appname>.internal`|6PN addresses of<br> Machines in all regions<br> for the app|none
|`regions.<appname>.internal`|none|comma-separated list<br> of region names where<br>Machines are deployed<br> for app|
|`<value>.<key>.kv._metadata.<appname>.internal`|6PN addresses of<br> Machines with<br> matching [metadata](https://community.fly.io/t/dynamic-machine-metadata/13115)|none|
|`_apps.internal`|none|comma-separated list<br> of the names of all apps<br> in current organization|
|`_peer.internal`|none|comma-separated list<br> of the names of all<br> WireGuard peers in<br> current organization|
|`<peername>._peer.internal`|6PN address of peer|none|
|`_instances.internal`|none|comma-separated list<br> of Machine ID, app name,<br>6PN address, and region for<br> all Machines in current<br> organization|

Examples of retrieving this information are in the [fly-examples/privatenet](https://github.com/fly-apps/privatenet) repository.

## Flycast - Private Fly Proxy services

A Flycast address is an app-wide IPv6 address that the Fly Proxy can route to privately. 6PN traffic doesn't go through the Fly Proxy, so can't use its features, like geographically aware load balancing and autostart/autostop based on traffic. Conversely, Flycast services can't use internal DNS to target specific Machines or regions, and are not reachable on `.internal` addresses.

Use Flycast to do the following within the Fly.io private WireGuard mesh:

* [Autostart and/or autostop](https://fly.io/docs/apps/autostart-stop/) Machines based on network requests.
* Use Fly Proxy's [geographically aware load balancing](/docs/reference/load-balancing/) for private services.
* Connect to a service from another app that can't use DNS.
* Connect from third-party software, like a database, that doesn't support round-robin DNS entries.
* Access specific ports or services in your app from other Fly.io organizations.
* Use advanced proxy features like TLS termination or PROXY protocol support.

The general flow for setting up Flycast is:

1. Allocate a private IPv6 address for your app on one of your Fly.io organization networks.
2. Expose services in your app's `fly.toml` `[services]` or `[http_service]` block; **do not use `force_https` as Flycast is HTTP-only**.
3. Deploy your app.
4. Access the services on the private IP from the target organization network.

<div class="warning icon">
**Warning:** If you have a public IP address assigned to your app, then services in `fly.toml` are exposed to the public internet. Verify your app's IP addresses with `fly ips list`.
</div>

### Assign a Flycast address

By default, the Flycast IP address is allocated on an app's parent organization network.

```cmd
 fly ips allocate-v6 --private
 ```
 ```output
VERSION	IP                	TYPE   	REGION	CREATED AT
v6     	fdaa:0:22b7:0:1::3	private	global	just now
```

If you want to expose services to another Fly.io organization you have access to, then use the `--org` flag.

```cmd
 fly ips allocate-v6 --private --org my-other-org
 ```
 ```output
VERSION	IP                	TYPE   	REGION	CREATED AT
v6     	fdaa:0:22b7:0:1::3	private	global	just now
```

### DNS and Flycast

You can use `appname.flycast` domains. They behave like [`appname.internal`](#fly-io-internal-addresses) domains, except they only return the app's Flycast addresses (if you have any).

<div class="callout">
The original motivation for this is accommodating PostgreSQL clients that don’t like raw IPv6 addresses in the connection string. The eagle-eyed and elephant-memoried of you might remember that we introduced Flycast for PostgreSQL to get away from DNS! Why are we going back? The problem we were trying to get away from with DNS is avoiding the case where there is a lag between a PostgreSQL instance becoming unhealthy or dying and it getting removed from DNS. Flycast IPs don’t change so we don’t have to worry about that issue in this case.
</div>

## Private Network VPN

You can use the [WireGuard](https://wireguard.com/+external) VPN to connect to the 6PN private network. WireGuard is a flexible and secure way to plug into each one of your Fly.io organizations and connect to any app within that organization.

### Set up a private network VPN

To set up your VPN, you'll use flyctl to generate a tunnel configuration file with private keys already embedded. Then you can load that file into your local WireGuard application to create a tunnel. Activate the tunnel and you'll be using the internal Fly.io DNS service which resolves `.internal` addresses - and passes on other requests to Google's DNS for resolution.

#### 1. Install your WireGuard App

Visit the [WireGuard](https://www.wireguard.com/install/+external) site for installation options. Install the software that is appropriate for your system. Windows and macOS have apps available to install. Linux systems have packages, typically named wireguard and wireguard-tools, you should install both.

#### 2. Create your tunnel configuration

To create your tunnel, run:

```cmd
fly wireguard create
```

Select the organization you want the WireGuard tunnel to work with:

```output
? Select organization:  [Use arrows to move, type to filter]
> My Org (personal)
  Test Org (test-org)
```

The `fly wireguard create` command configures the WireGuard service and generates a tunnel configuration file, complete with private keys which cannot be recovered. This configuration file will be used in the next step. First, save the configuration file:

```output
!!!! WARNING: Output includes private key. Private keys cannot be recovered !!!!
!!!! after creating the peer; if you lose the key, you’ll need to remove    !!!!
!!!! and re-add the peering connection.                                     !!!!
? Filename to store WireGuard configuration in, or 'stdout':  mypeer.conf
Wrote WireGuard configuration to 'mypeer.conf'; load in your WireGuard client
```

We suggest you name your saved configuration with the same name as the peer you have created. Add the extension `.conf` to ensure it will be recognized by the various WireGuard apps as a configuration file for a tunnel. Note that the name (excluding the `.conf` extension) shouldn't exceed 15 characters since this is the maximum length for an interface name on Linux.

<div class="important icon">
**Important:** If you want to interact with your peer using its name, such as queries to `_peer.internal` or `<peername>._peer.internal`, then you need to specify a name when you create the tunnel.

Defaults are used when you don't specify a `region` and `name` in the `fly wireguard create` command. Default generated names start with `interactive-*` and we filter `interactive-*` out of DNS (because of the sheer volume of them).

To specify a peer name and region, first look up available regions by running `fly platform regions`. Select a region with a check mark in the Gateway column.

Then run: `fly wireguard create [your-org] [region] [peer-name]`
</div>

#### 3. Import your tunnel

##### Windows

Run the WireGuard app. Click **Import tunnel(s) from file**. Select your configuration file. The WireGuard app will display the details of your tunnel. Click **Activate** to bring the tunnel online.

##### macOS

Run the WireGuard app. Click **Import tunnel(s) from file**. Select your configuration file and click **Import**. You might be prompted by the OS that WireGuard would like to add VPN configurations; click **Allow**. The WireGuard app will display the details of your tunnel. Click **Activate** to bring the tunnel online.

##### Ubuntu Linux

If you don't have `wg-quick` installed, then run the command below. For Ubuntu 18.04 to 22.04, `openresolv` is also required.

```
sudo apt install wireguard-tools openresolv
```

Copy the configuration file to `/etc/wireguard`; you'll need root/sudo permissions:

```
sudo cp basic.conf /etc/wireguard
```

Run `wg-quick` to bring `up` the connection by name (i.e. less the `.conf` extension). For example:

```cmd
wg-quick up mypeer
```
```output
[#] ip link add mypeer type wireguard
[#] wg setconf mypeer /dev/fd/63
[#] ip -6 address add fdaa:0:4:a7b:ab6:0:a:102/120 dev mypeer
[#] ip link set mtu 1420 up dev mypeer
[#] resolvconf -a tun.mypeer -m 0 -x
[#] ip -6 route add fdaa:0:4::/48 dev mypeer
```

### Test the VPN tunnel

If you have the `dig` tool installed, a TXT query to `_apps.internal` will show all the apps in the organization you are connected to:

```cmd
dig +noall +answer _apps.internal txt
```
```output
_apps.internal.		5	IN	TXT	"my-app-name,my-app-name-0,my-app-name-1"
```

You can also query for peer names and addresses:

```cmd
dig +short txt _peer.internal @fdaa:0:18::3
```
```output
"my-peer"
```

```cmd
dig +short aaaa my-peer._peer.internal @fdaa:0:18::3
```
```output
fdaa:0:18:a7b:7d:f066:b83b:102
```

Query for the 6PN addresses of all started Machines in an app:

```
dig +short aaaa my-app-name.internal
```

### Manage WireGuard on Fly.io

#### List the tunnels

To list all the tunnels set up for an organization, run `fly wireguard list`. You can provide an organization on the command line or select an org when prompted.

#### Remove a tunnel

To remove a tunnel, run `fly wireguard remove`. You can specify the organization and tunnel name on the command line or be prompted for both.
