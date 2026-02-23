# victron-bluetooth-safety

Patch Victron's `vesmart-server` on Venus OS so it only disconnects its
own GATT clients (VictronConnect), instead of disconnecting **all** BLE
devices on **all** adapters every 60 seconds.

This is the surgical alternative to
[disable-victron-bluetooth](https://github.com/TechBlueprints/disable-victron-bluetooth),
which disables `vesmart-server` entirely. This patch keeps VictronConnect
BLE access and VE.Smart Networking working while preventing collateral
damage to third-party BLE connections.

## The problem

When any BLE device connects to the Cerbo (even on a different adapter),
`vesmart-server` starts a 60-second keep-alive timer. When that timer
fires (because the connected device isn't a VictronConnect client and
never sends a keep-alive), it disconnects **every** connected BLE device
it can find — batteries, sensors, everything.

Upstream issue:
[victronenergy/venus#1587](https://github.com/victronenergy/venus/issues/1587)

## What the patch does

The fix is based on the fact that vesmart-server's GATT service already
knows which devices are VictronConnect clients — they're the only ones
that read/write its `306b*` characteristics. Other BLE devices (batteries,
sensors) use their own GATT services; the Cerbo connects outbound to them.

The patch makes four changes:

1. **Tracks GATT clients** — records the device path from the BlueZ
   `options["device"]` dict when `ControlChr.read_value()` is called
   (the VictronConnect handshake)
2. **Defers the keep-alive timer** — only starts the 60s timer when a
   real GATT interaction occurs, not on generic BlueZ connection events
3. **Scopes disconnects** — the timeout handler only disconnects tracked
   GATT clients, not all BlueZ devices
4. **Cleans up on disconnect** — removes devices from the tracked set
   when they disconnect naturally

## Install

Copy the project to the Cerbo and run the installer:

```bash
scp -r patches victron-bluetooth-safety.sh root@cerbo:/data/victron-bluetooth-safety/
ssh root@cerbo 'sh /data/victron-bluetooth-safety/victron-bluetooth-safety.sh install'
```

Or install directly:

```bash
ssh root@cerbo 'mkdir -p /data/victron-bluetooth-safety'
scp patches/*.patch root@cerbo:/data/victron-bluetooth-safety/
scp victron-bluetooth-safety.sh root@cerbo:/data/victron-bluetooth-safety/
ssh root@cerbo 'sh /data/victron-bluetooth-safety/victron-bluetooth-safety.sh install'
```

The installer:

1. Remounts the root filesystem read-write
2. Applies the patches to `/opt/victronenergy/vesmart-server/`
3. Restores root to read-only
4. Adds a hook to `/data/rc.local` to reapply the patch on boot
   (including after firmware updates)
5. Restarts `vesmart-server` to pick up the changes

## Uninstall

```bash
ssh root@cerbo 'sh /data/victron-bluetooth-safety/victron-bluetooth-safety.sh uninstall'
```

## Status

```bash
ssh root@cerbo 'sh /data/victron-bluetooth-safety/victron-bluetooth-safety.sh status'
```

## How it survives firmware updates

Venus OS firmware updates replace the entire root filesystem, which
reverts the patched files. The installer adds a hook to `/data/rc.local`
(which lives on the persistent `/data` partition) that reapplies the
patches on every boot.

See [Venus OS: Root Access](https://www.victronenergy.com/live/ccgx:root_access)
for details on the Venus OS customization model.

## Development

A non-production Cerbo GX (`einstein`) is available for testing at
`root@dev-cerbo`.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
