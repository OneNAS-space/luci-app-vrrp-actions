# luci-app-vrrp-actions

Makes keepalived on OpenWrt do something useful when its state changes, and adds a LuCI page to control it.

keepalived itself only decides who is MASTER and who is BACKUP, and moves the VIP. It does not decide how the network services should follow: who hands out DHCP, who answers DNS, whether a proxy service runs. That is what this package adds: **when the node becomes MASTER, BACKUP or FAULT, or when keepalived stops, the matching services are switched on or off according to your settings**. It can also keep DHCP leases and DHCP config in step between two routers.

## Features

### Role actions

For each state (MASTER / BACKUP / FAULT / SHUTDOWN), choose **On**, **Off** or **Keep as it is** for each of these services:

| Service | What it does |
|---|---|
| dnsmasq | Runs or stops the DNS / DHCP service itself |
| DHCPv4 | On: the gateway and DNS handed out are the LAN VIP. Off: DHCP is no longer handed out |
| DHCPv6 / RA | Switches DHCPv6, prefix delegation and router advertisements on or off |
| DHCP sync | Starts or stops the DHCP sync service (see below) |
| Natmap | Enabled and started, or disabled and stopped |
| Cloudflared | Enabled and started, or disabled and stopped |

Natmap and Cloudflared are listed only when they are installed.

The MASTER state can also take over data from the peer **before** DHCP is enabled:

- **Take over the leases**: fetch the peer's lease file first
- **Take over the DHCP config**: fetch the peer's DHCP config (static leases, tags, ...) first

The SHUTDOWN state can clear the RA flags and NTP servers on the LAN that the MASTER state had set.

### DHCP sync

Keeps the DHCP leases (`dhcp.leases`) and the DHCP config (`/etc/config/dhcp`) of two routers in step. Two directions:

- **Push to the peer**: this node is the authority and sends its leases and config to the peer. For two equal routers: the MASTER keeps pushing to the BACKUP, so the BACKUP has the latest data when it takes over.
- **Pull from the peer**: the peer is the authority and this node regularly takes its leases and config; nothing is ever written to the peer. For a main router plus a bypass router: the main router runs DHCP and the bypass router just follows its static leases and tags.

Each router's own DHCP switches (DHCPv4/DHCPv6 of the LAN, gateway and DNS options) are never overwritten. The `dnsmasq` and `odhcpd` sections also stay as each router has them by default (a router running PassWall2 or AdGuard Home has its own settings there); turn on "Sync the dnsmasq and odhcpd options" only for two routers with exactly the same setup.

### Pages

- **Services → Keepalived → Role Actions**: all the settings, in the tabs General, MASTER, BACKUP, FAULT, SHUTDOWN and DHCP sync. The tab of the current state is coloured (MASTER green, BACKUP amber, FAULT red), and hovering over a tab shows what that state means.
- **Status → VRRP**: next to the existing Instances there is a **Role Actions** tab with the current role and a table of what each service gets in this state.
- **Status → Overview**: a **VRRP Role Actions** card with what is followed, the current role and whether the role actions are enabled. It is not shown when no keepalived instance is configured.

## Installation

Requires `luci-app-keepalived`, `keepalived`, `rsync` and `inotifywait` (installed automatically).

```sh
apk add luci-app-vrrp-actions
```

keepalived has to be set up first: at least one `vrrp_instance`, usually plus a `vrrp_sync_group` that groups the LAN and WAN instances.

## Usage

**After installation nothing is done by default.** Every switch is off, so a running VRRP setup is not touched. Check the actions of each state first, then switch it on.

1. Open **Services → Keepalived → Role Actions**.
2. **General** tab:
   - **Follow**: whose state to follow. Automatic (default): the first sync group; without one, the instance on br-lan; otherwise the only instance.
   - The current role is shown here too.
3. **MASTER / BACKUP / FAULT / SHUTDOWN** tabs: choose On / Off / Keep for each service to fit your network. The text under each option changes with your choice and says what it does.
4. When you are happy with the actions, go back to **General**, tick **Enabled**, then Save & Apply. From then on the actions run on every keepalived state change.
5. To use DHCP sync, open the **DHCP sync** tab, tick **Enabled**, choose the **Mode** (push or pull), then Save & Apply. The DHCP sync action of each state decides in which state the sync runs (default: On in MASTER, Off in BACKUP and FAULT).

### Default actions

| Service | MASTER | BACKUP | FAULT | SHUTDOWN |
|---|---|---|---|---|
| dnsmasq | On | On (DNS only) | Keep | Keep |
| DHCPv4 | On | Off | Keep | Keep |
| DHCPv6 / RA | On | Off | Keep | Keep |
| DHCP sync | On | Off | Off | Keep |
| Natmap | On | Off | Off | Off |
| Cloudflared | Off | On | Off | Off |

By default MASTER also takes over the peer's leases and DHCP config first.

### Before using DHCP sync

The sync runs between the two routers over ssh and rsync, so:

1. **Install this package on both routers.** The router that is only pulled from needs it too (it can keep every switch off): it exports its DHCP config to the other one.
2. **Set up passwordless ssh.** On each router, create a key and add its public key to the other router:

   ```sh
   # create the key here (skip if it exists) and show the public key
   [ -f /root/.ssh/id_dropbear ] || dropbearkey -t ed25519 -f /root/.ssh/id_dropbear
   dropbearkey -y -f /root/.ssh/id_dropbear
   ```

   Append the `ssh-ed25519 ...` line of the output to `/etc/dropbear/authorized_keys` on the **other** router. Do it in both directions.

3. **Peer address**: when left empty, the unicast peer of the vrrp instance on br-lan is used (set `unicast_peer` in keepalived); otherwise fill in **Peer address** in the DHCP sync tab.

Other options of the DHCP sync tab:

- **Sync the DHCP config**: also sync `/etc/config/dhcp`, not just the leases
- **Re-check interval**: seconds between periodic runs. In push mode it checks that the peer still matches, in pull mode it is the time between pulls
- **Retry interval**: seconds before a retry after a failure
- **More options kept as they are**: further options that each router keeps for itself, written as `section.option`, for example `lan.leasetime`

### Two typical setups

**Two equal routers (A is MASTER, B is BACKUP)**
The default actions are fine. On both routers tick Enabled in the DHCP sync tab and choose Push. The MASTER keeps pushing its leases and DHCP config to the other, which takes over DHCP when it becomes MASTER.

**Main router is BACKUP, bypass router is MASTER**
The main router always runs DHCP and DNS; the bypass router only runs the proxy and holds the VIP, and when it goes down the VIP moves back to the main router.
- Bypass router: set DHCPv4 to Off in the MASTER tab (it does not hand out DHCP), tick Enabled in the DHCP sync tab and choose Pull, so it keeps following the static leases and tags of the main router.
- Main router: install the package but keep everything off. It is only the one being pulled from.

## Notes

- **Off** for dnsmasq leaves the router itself without a resolver (`/etc/resolv.conf` points to it), so make sure there is another upstream DNS before using it.
- When installing as an upgrade, the existing settings are kept.
- To see what was done:

  ```sh
  logread -e keepalived-scripts   # role actions
  logread -e sync_dhcp            # DHCP sync
  ```

## License

GPL-3.0-or-later
