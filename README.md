# go-provision

Provisioning service for GOcontroll Moduline controllers (Debian, ARM64, i.MX8).

Exposes a temporary HTTP endpoint on port **8743** that returns controller hardware
information as JSON. Not started automatically — triggered by `go-reprovision` or
an external provisioning workflow.

## Endpoint

```
GET http://<controller-ip>:8743/info
```

### Example response

```json
{
  "mac": "AA:BB:CC:DD:EE:FF",
  "ip": "192.168.1.42",
  "hostname": "moduline-aabbcc",
  "serial": "XXXX-XXXX-XXXX-XXXX",
  "hardware_version": "Moduline 3.06-D",
  "modules": [
    { "slot": 1, "name": "Output module 8CH", "qr_front": "1234567890123456", "qr_rear": "6543210987654321" },
    { "slot": 2, "name": null, "qr_front": null, "qr_rear": null }
  ],
  "modem": {
    "serial": "SIMCOM-SN-987654",
    "sim_iccid": "89314404000123456789",
    "imei": "352099001761481"
  },
  "wifi": {
    "model": "1DX"
  }
}
```

Fields `modem` and `wifi` are `null` when the hardware is absent.  
`serial` is `null` when no serial number has been written yet.

## Installed files

| Path | Description |
|------|-------------|
| `/usr/bin/go-provision-server` | Python 3 HTTP server (stdlib only) |
| `/usr/lib/systemd/system/go-provision.service` | Systemd unit (not enabled by default) |
| `/usr/bin/go-reprovision` | Helper: clears provisioned flag and starts the service |

## Usage

```bash
# Start provisioning service manually (stops after 120 s)
systemctl start go-provision.service

# Re-trigger provisioning from scratch
go-reprovision

# Write serial number (requires go-sn)
go-sn write XXXX-XXXX-XXXX-XXXX
```

## Installation

```bash
apt-get install go-provision
```

Requires the [GOcontroll apt repository](https://github.com/GOcontroll/GOcontroll-apt).

## Dependencies

- `python3` — runtime
- [`go-sn`](https://github.com/GOcontroll/go-sn) — serial number read/write
