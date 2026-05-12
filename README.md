# go-provision

Provisioning service for GOcontroll Moduline controllers (Debian, ARM64, i.MX8).

Exposes an HTTP endpoint on port **8743** that returns controller hardware
information as JSON and accepts provisioning commands. Not started automatically —
triggered by `go-reprovision` or an external provisioning workflow (e.g. the
GOcontroll database UI via the provision agent).

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/info`        | Returns hardware info as JSON (network, serial, modules, modem, wifi) |
| `POST` | `/sn`          | Writes the controller serial number via `go-sn` and validates by read-back |
| `POST` | `/postinstall` | Starts `/usr/local/sbin/postinstall.sh` in the background (apt update/upgrade, hostname, …) |
| `POST` | `/run-script`  | Saves the request body script to `/tmp/go-provision-run.sh` and executes it in the background |

### `GET /info`

Example response:

```json
{
  "mac": "AA:BB:CC:DD:EE:FF",
  "ip": "192.168.1.42",
  "hostname": "moduline-aabbcc",
  "serial": "XXXX-XXXX-XXXX-XXXX",
  "platform": "Moduline Mini",
  "hardware_version": "1.11",
  "hardware_full": "Moduline Mini V1.11",
  "software_version": "Linux 6.12.x",
  "modules": [
    { "slot": 1, "empty": false, "qr_front": "1234567890123456", "qr_back": "6543210987654321", "module_type": "Output module 8CH", "firmware": "1.2.0", "manufacturer": "GOcontroll" },
    { "slot": 2, "empty": true }
  ],
  "modem": {
    "detected": true,
    "serial": "862636055064800",
    "sim_iccid": "89314404000123456789",
    "imei": "862636055064800"
  },
  "wifi": { "model": "1DX" }
}
```

`modem.detected` is `false` when no modem is present. `wifi` is `null` when no
supported USB adapter is connected. `serial` is `null` when no serial number
has been written yet.

### `POST /sn`

Writes the controller serial number and validates by reading it back.

Request body:

```json
{ "serial": "XXXX-XXXX-XXXX-XXXX" }
```

Success response (HTTP 200):

```json
{ "ok": true, "wrote": "XXXX-XXXX-XXXX-XXXX", "read_back": "XXXX-XXXX-XXXX-XXXX" }
```

Failure response (HTTP 400):

```json
{ "ok": false, "wrote": "…", "error": "read-back mismatch" }
```

The serial format must be four groups of four alphanumeric characters separated
by dashes (e.g. `A4CG-B056-A08D-A001`).

### `POST /postinstall`

Starts `/usr/local/sbin/postinstall.sh` in the background. Returns immediately
(HTTP 202). The script may take several minutes (apt update/upgrade, hostname,
go-modules update, …). Output is appended to `/var/log/go-provision-postinstall.log`.

Response:

```json
{ "started": true, "log": "/var/log/go-provision-postinstall.log" }
```

### `POST /run-script`

Executes an arbitrary bash script on the controller. Used by the provisioning
flow to deploy the tunnel install script (cloudflared + go-deploy + MQTT config).

Request body:

```json
{ "script": "#!/bin/bash\ncurl -fL https://…\n…" }
```

Response (HTTP 202):

```json
{ "started": true, "log": "/var/log/go-provision-run.log", "size": 4321 }
```

The script is written to `/tmp/go-provision-run.sh`, made executable, and
spawned via `/bin/bash` in a new session. Output is appended to
`/var/log/go-provision-run.log`. Use this endpoint with care — it grants
remote shell execution to any caller on the LAN.

## Installed files

| Path | Description |
|------|-------------|
| `/usr/bin/go-provision-server` | Python 3 HTTP server (stdlib only) |
| `/usr/lib/systemd/system/go-provision.service` | Systemd unit (not enabled by default) |
| `/usr/bin/go-reprovision` | Helper: clears provisioned flag and starts the service |

## Usage

```bash
# Start provisioning service manually (stops itself after RuntimeMaxSec)
systemctl start go-provision.service

# Re-trigger provisioning from scratch
go-reprovision

# Manual SN write (without going through /sn endpoint)
go-sn write XXXX-XXXX-XXXX-XXXX

# Tail postinstall / run-script logs
tail -f /var/log/go-provision-postinstall.log
tail -f /var/log/go-provision-run.log
```

## Installation

```bash
apt-get install go-provision
```

Requires the [GOcontroll apt repository](https://github.com/GOcontroll/GOcontroll-apt).

## Dependencies

- `python3` — runtime
- [`go-sn`](https://github.com/GOcontroll/go-sn) — serial number read/write
- `postinstall.sh` at `/usr/local/sbin/postinstall.sh` — provided by the base image
- `mmcli` (ModemManager) and `go-modules` — used by `/info` for modem and module detection
