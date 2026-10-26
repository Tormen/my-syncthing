# my-syncthing

Set up and watch a [Syncthing](https://syncthing.net) cluster from ONE config.

The config says which host is the hub, where it listens, which devices
exist and which folders go where. `my-syncthing --install` makes the host
it runs on match it; `my-syncthing` tells what Syncthing really does. It
syncs nothing itself -- Syncthing does, continuously.

Python 3.9+, standard library only. macOS and Linux.

## The shape of the cluster

- **One hub.** Only the hub listens on the network; every other host (a
  peer) listens on loopback and dials the hub at fixed addresses. Peers
  never sync with each other directly, so nothing syncs while the hub is
  down -- by design.
- **No phone home** (`PHONE_HOME=no`): global and LAN announce, relays, NAT
  traversal, usage reports, the upgrade check and crash reports are all
  switched off. Every address is fixed; no third party is contacted.
- **The config is the trust list.** The hub accepts only the devices the
  config names; a folder exists only on the hosts its `hosts=` names.

## Usage

    my-syncthing                    the short usage, then the status
    my-syncthing status             the status alone (one line per fact, the raw value beside it)
    my-syncthing disable [<FOLDER>] stop syncing a folder on THIS host (none: all folders)
    my-syncthing enable [<FOLDER>]  sync it again
    my-syncthing web                open this host's Syncthing web UI in the default browser
    my-syncthing check              the conditions, told through the alerts
    my-syncthing diag [<BLOCK>]     plan, folders, connections, conflicts, errors,
                                    localchanged, ignores, config
    my-syncthing --install          this host as the config says
    my-syncthing --uninstall        its service and check agent (data and identity stay)

Modifiers: `-S` (show what would change, change nothing), `-Q` (only
conditions and errors), `-V` (more detail), `-D` (each REST call),
`--config <FILE>`. Admin: `--create-config [<FILE>]`, `--test-mail`,
`--version`, `--run-tests [<FILTER>]`. All of it: `my-syncthing --help`.

Exit: 0 healthy, 1 a condition holds (or `--install` needs the config
changed), 2 an error.

## The config

Shell syntax, sourced by `/bin/sh` -- so it may source a site file of its
own. The same file on every host. `my-syncthing --create-config` prints the
defaults with a comment on each key. The essentials:

    HUB=server
    HUB_ADDRESSES="tcp://192.0.2.10:22000 tcp://hub.example.org:22000"
    DEVICE_server=XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX
    DEVICE_laptop=...
    FOLDER_data="/srv/data hosts=server,laptop versioning=staggered:365d ignore=(?d).DS_Store,.git perms=ignore xattrs=off"
    SERVICE=agent                  # agent | daemon | none
    SERVICE_server=daemon          # the hub runs from boot
    ALERT_NOW="notify-send"        # at once:  <command> <subject> <body>
    ALERT_LATER="mail-me"          # once, if the condition still holds ALERT_DELAY later

Hosts are named as `hostname -s` gives them, lower case, with anything a
shell variable name cannot hold as `_` (`My-Laptop` -> `my_laptop`).

Search order, first found wins: `$MY_SYNCTHING_CONFIG`, `--config <FILE>`,
`/LINKS/default/my-syncthing.conf` (or `/LINKS/default/my-syncthing`),
`~/.my-syncthing.conf`, `/etc/my-syncthing.conf`,
`/usr/local/etc/my-syncthing.conf`.

## --install

Each step reads first and changes only what differs, naming what and why;
a second run prints nothing to change.

1. **Syncthing** itself: on macOS through Homebrew (`brew install
   syncthing`); on Linux it asks for the distro package. A running
   Syncthing.app is refused: the service replaces it and uses the same
   config directory, so the device keeps its identity.
2. **Identity**: created on first start, never copied between hosts. A host
   the config does not know yet prints its `DEVICE_<host>=` line and the
   folders it is in no `hosts=` of, and stops -- that is how a new host
   introduces itself. Add the line, run `--install` on it and on the hub.
3. **Service** as `SYNC_USER`: `agent` -- a LaunchAgent / a systemd `--user`
   unit, while the user is logged in; `daemon` -- a LaunchDaemon with
   `UserName` / `syncthing@<user>.service`, from boot. A daemon needs root:
   `my-syncthing --install` run as root installs only that, then again as
   the user. `none` -- something else starts Syncthing.
4. **Settings** by role: listen addresses, the phone-home switches (as
   switches -- an emptied list comes back as `default`), no browser.
5. **Devices**: on the hub every device of the config, on a peer the hub;
   folders are never accepted automatically.
6. **Folders** this host is named in, with their options and `.stignore`. A
   new folder is added paused, gets its ignores, then starts. One whose
   directory already holds data starts **receive-only**: it sends nothing
   until its first scan is done and nothing differs here; the next
   `--install` then makes it send-and-receive, or says how many local
   changes keep it receive-only (`diag localchanged`).
7. **The check agent**, every `CHECK_INTERVAL`.

A NEW server: set `HUB=<new name>` and its addresses; every peer's next
`--install` dials it.

## Folder options

- `versioning=staggered:<N>d` keeps, on each device, what OTHER devices
  replace or delete -- a local change is never archived by Syncthing -- so
  a mistake made on one host survives on the others. Versions go to
  `.stversions` in the folder root.
- `xattrs=off` (the default): extended attributes are neither sent nor
  applied. Where each host writes its own values into the same files
  (indexers, `com.apple.provenance`), syncing them would ping-pong.
  `check` reports it as drift when it is switched on.
- `perms=ignore` (the default): permission bits are not synced -- each
  host's own policy sets them. Ownership is never synced.
- Conflicts: Syncthing renames the OLDER copy
  `<name>.sync-conflict-<date>-<time>-<device>.<ext>`, which then syncs
  like any file; `check` counts them (`-V` names them).
- Symlinks are synced as links, never followed.
- The folder ROOT must be writable by the syncing user: the marker,
  `.stignore`, `.stversions` and the temporary files are written there.
- Never put an application's live database inside a synced folder.

## disable / enable

Syncthing's own per-folder `paused` flag, on this host only: it lasts across
restarts, touches no data, and the other hosts go on among themselves.
`--install` leaves it as it is; the status shows it
(`DISABLED here since ... [paused=true]`); it is never an alert. Without a
folder name, both act on every folder of the host.

## status and check

Per folder: state, completion per device, bytes needed, errors, conflict
files. Per host: the hub connection (address, since), the Syncthing
version, and the settings `--install` would change.

Conditions (`CONDITIONS` picks): `not-running`, `hub-unreachable` (after
`HUB_GRACE`), `folder-error`, `conflicts`, `out-of-sync` (after
`SYNC_GRACE`), `drift` (a setting not as configured). `check` runs
`ALERT_NOW <subject> <body>` when a condition first holds, `ALERT_LATER`
once if it still holds `ALERT_DELAY` later, and `ALERT_NOW` once more when
it clears. `--test-mail` sends one test through `ALERT_LATER`.

## Tests

`my-syncthing --run-tests [<FILTER>]` runs the built-in suite against a stub
Syncthing REST API on loopback -- nothing real is touched.
