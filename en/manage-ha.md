Title: High Availability
TODO:  CDO QA (irc: cgregan/jog) might be testing/using installing HA via Juju
table_of_contents: True

# High Availability

This page describes how to provide high availability (HA) for MAAS at both the
rack-controller level and the region-controller level. See [Concepts and
terms][concepts-controllers] for detailed information on what services are
provided by each of those levels.

## Rack controller HA

Multiple rack controllers are necessary to achieve high availability. Please see
[Rack controller][install-rackd] to learn how to install rack controllers.

### Multiple region endpoints

MAAS will automatically discover and track all reachable region controllers in a
single cluster as well as automatically attempt to connect to them in the event
that the one being used becomes inaccessible.

Administrators can alternatively specify multiple region-controller endpoints
for a single rack controller by adding entries to `/etc/maas/rackd.conf`.

E.g.

```
.
.
.
maas_url:
  - http://<ip 1>:<port>/MAAS/
  - http://<ip 2>:<port>/MAAS/
.
.
.
```

### BMC HA

HA for BMC control (node power cycling) is provided out of the box once a second
rack controller is present. MAAS will automatically identify which rack
controller is responsible for a BMC and set up communication accordingly.

### DHCP HA

DHCP HA affects node management (enlistment, commissioning and deployment). It
enables a primary and a secondary DHCP instance to serve the same VLAN where all
lease information is replicated between rack controllers. MAAS-managed DHCP is a
requirement for DHCP HA.

If you are enabling DHCP for the first time after adding a second rack
controller, please read [Enabling DHCP][enabling-dhcp].

However, if you have already enabled DHCP on your initial rack controller,
you'll need to reconfigure DHCP. Simply access the appropriate VLAN (via the
'Subnets' page) and choose action 'Reconfigure DHCP'. There, you will see the
second rack controller in the 'Secondary controller' field. All you should have
to do is press the 'Reconfigure DHCP' button:

![reconfigure DHCP][img__reconfigure-dhcp]

The setup of rack controller HA is now complete.

!!! Note:
    For HA purposes, DHCP provisioning will take into account multiple DNS
    services when there is more than one region controller on a single region.


## Region controller HA

Implementing region controller HA involves setting up:

- PostgreSQL HA
- Secondary API server(s)

Load balancing is optional.

### PostgreSQL HA

MAAS stores all state information in the PostgreSQL database. It is therefore
recommended to run it in HA mode. Configuring HA for PostgreSQL is external to
MAAS. You will therefore need to study the
[PostgreSQL documentation][upstream-postgresql-docs] and implement the variant
of HA that you feel most comfortable with.

A quick treatment of [PostgreSQL HA: hot standby][postgresql-ha] is provided
here for convenience only. Its purpose is to give an idea of what's involved at
the command line level when implementing one particular form of HA with
PostgreSQL.

!!! Note:
    Each region controller uses up to 40 connections to PostgreSQL in high load
    situations. Running 2 region controllers requires no modifications to the
    `max_connections` in `postgresql.conf`. More than 2 region controllers
    requires that `max_connections` be adjusted to add 40 more connections per
    extra region controller added to the HA configuration.

### Secondary API server(s)

Please see [Region controllers][install-region] and [Multiple region
endpoints][multi-region-endpoints] for more information about how to install
and configure rack controllers for multiple region controllers.

### Load balancing with HAProxy (optional)

Load balancing can be added with [HAProxy][upstream-haproxy] load-balancing
software to support multiple API servers. In this setup, HAProxy provides access
to the MAAS web UI and API.

!!! Note:
    If you happen to have Apache running on the same server where you intend to
    install HAProxy, you will need to stop and disable `apache2`, because
    HAProxy binds to port 80.

#### Install

```bash
sudo apt install haproxy
```

#### Configure

Configure each API server's load balancer by copying the following into
`/etc/haproxy/haproxy.cfg` (see the
[upstream configuration manual][upstream-haproxy-manual]
as a reference). Replace $PRIMARY_API_SERVER_IP and $SECONDARY_API_SERVER_IP
with their respective IP addresses:

```yaml
frontend maas
    bind    *:80
    retries 3
    option  redispatch
    option  http-server-close
    default_backend maas

backend maas
    timeout server 90s
    balance source
    hash-type consistent
    server localhost localhost:5240 check
    server maas-api-1 $PRIMARY_API_SERVER_IP:5240 check
    server maas-api-2 $SECONDARY_API_SERVER_IP:5240 check
```

Where `maas-api-1` and `maas-api-2` are arbitrary server labels.

Now restart the load balancer to have these changes take effect:

```bash
sudo systemctl restart haproxy
```

The configuration of region controller HA is now complete.

**The API server(s) must be now be referenced (e.g. web UI, MAAS CLI) using
port 80 (as opposed to port 5240).**

## Snap

Setting up high-availability using snaps is relatively easy. To use snaps
instead of a package distribution of MAAS:

1. Set up PostgreSQL for high-availability as [explained
   above][postgresql-setup]. PostgreSQL should run outside of the snap.
1. [Install][snap-install] the MAAS snap on each machine you intend to use as a
rack or region controller. You'll need the MAAS shared secret, located here,
`/var/lib/maas/secret`, on the first region controller you set up.
1. [Initialise the snap][snap-config] as a `rack` or `region` controller. Note
that if you intend to use a machine as a region controller, you'll need to tell
MAAS how to access your PostgreSQL database host with the following arguments:
    - `--database-host DATABASE_HOST`
    - `--database-name DATABASE_NAME`
    - `--database-user DATABASE_USER`
    - `--database-pass DATABASE_PASS`


## VIP

Note that in previous versions of MAAS, a virtual IP was required to serve as
the effective IP address of all region API servers in high-availability
environments. Since MAAS 2.5, this is no longer required. See [Virtual
IP][virtualip] for more information.

<!-- LINKS -->

[virtualip]: https://docs.maas.io/2.4/en/manage-ha#virtual-ip
[syslog]: installconfig-syslog.md
[snap-config]: installconfig-snap-install.md#initialisation
[snap-install]: installconfig-snap-install.md#install-from-snap
[concepts-controllers]: intro-concepts.md#controllers
[rackd-communication]: installconfig-rack.md#communication-between-machines-and-rack-controllers
[multi-region-endpoints]: #multiple-region-endpoints
[install-region]: installconfig-region.md
[install-rackd]: installconfig-rack.md#install-a-rack-controller
[enabling-dhcp]: installconfig-network-dhcp.md#enabling-dhcp
[keepalived-man-page]: http://manpages.ubuntu.com/cgi-bin/search.py?q=keepalived.conf
[upstream-keepalived]: http://www.keepalived.org/
[upstream-haproxy-manual]: http://cbonte.github.io/haproxy-dconv/1.6/configuration.html
[upstream-haproxy]: http://www.haproxy.org/
[postgresql-setup]: manage-ha.md#postgresql-ha
[postgresql-ha]: manage-ha-postgresql.md
[upstream-postgresql-docs]: https://www.postgresql.org/docs/9.5/static/high-availability.html

[img__reconfigure-dhcp]: ../media/manage-ha__2.4_reconfigure-dhcp.png
