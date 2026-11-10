# Looking for someone to take over this repo as i'm unable to maintain it (no ble bridge devices)

# MeshCore BLE Bridge for ESPHome

This external ESPHome component keeps the BLE connection on the ESP32, where
`ble_client` can answer the MeshCore node's passkey request. It then exposes the
authenticated MeshCore BLE session as a small TCP server using the same framing
that `meshcore-py` already expects for TCP connections.

Use the stock MeshCore Home Assistant integration as a TCP connection:

- host: the ESPHome device hostname or IP, for example `meshcore-ble-link.local`
- port: `5000`

Do not configure this MeshCore node through Home Assistant Bluetooth Proxy. The
proxy creates its own BLE connection and cannot reuse the authenticated
`ble_client` session.

## ESPHome YAML

Keep the `ble_client` section that already authenticates with the node, then add:

```yaml
external_components:
  - source: github://matthew73210/meshcore-ble-bridge@v0.1.0
    refresh: 1d
    components: [meshcore_ble_bridge]

meshcore_ble_bridge:
  id: meshcore_bridge
  ble_client_id: meshcore_ble
  port: 5000
  force_encryption: true
  wait_for_auth: true
```

For a PIN-protected MeshCore node, keep this pattern:

```yaml
logger:
  level: DEBUG
  logs:
    esp32_ble_tracker: DEBUG
    esp32_ble_client: DEBUG
    ble_client: DEBUG
    meshcore_ble_bridge: DEBUG

esp32_ble:
  io_capability: keyboard_only

ble_client:
  - mac_address: AC:A7:04:07:CE:21
    id: meshcore_ble
    auto_connect: true
    on_passkey_request:
      then:
        - ble_client.passkey_reply:
            id: meshcore_ble
            passkey: 771519
```

The bridge's `force_encryption: true` setting replaces the previous
`on_connect` lambda that called `esp_ble_set_encryption` with MITM encryption.

Avoid `VERY_VERBOSE` for `esp32_ble_tracker` during normal testing. On an ESP32
with many BLE advertisements nearby, the scan log flood can trip the task
watchdog before the MeshCore client even connects.

`bluetooth_proxy` is intentionally not used for this MeshCore node. If you need a
general Home Assistant Bluetooth proxy, use another ESP32 or be prepared for BLE
slot/resource conflicts.

## Expected logs

After flashing, the ESPHome logs should show:

```text
BLE authentication complete
BLE handles ready: RX=... TX=... CCCD=...
MeshCore BLE bridge ready
MeshCore TCP bridge listening on port 5000
```

Then add or reconfigure MeshCore-HA with connection type `TCP`, using the ESPHome
device address and port `5000`.


## BLE reconnection diagnostics (experimental branch)

For diagnosis of repeated `ESP_GATTC_DISCONNECT_EVT reason 0x08` after a successful
PIN pairing, use the branch `diagnostic/ble-reconnect` rather than `main`:

```yaml
external_components:
  - source: github://radugeo74/meshcore-ble-bridge@diagnostic/ble-reconnect
    refresh: 1d
    components: [meshcore_ble_bridge]
```

Leave `force_encryption: true`, `wait_for_auth: true`, and the existing
`esp32_ble.io_capability: keyboard_only` and passkey automation enabled.

The patch does **not** change bonding, authentication, connection timing, or
reconnection behavior. It adds log messages for:

- immediate return status from `esp_ble_set_encryption` (a successful return
  means only that ESP-IDF accepted the request, not that encryption succeeded);
- BLE security request, passkey request, and key exchange events, without
  printing any PIN or key material;
- connection teardown state: whether authentication completed and the bridge
  became ready before the disconnect.

Capture DEBUG logs spanning initial pairing, a normal reconnect, and the first
disconnect. Compare an initial successful session with the subsequent failure.
Do not enable `VERY_VERBOSE` BLE scanning.

This branch has **not yet been compiled or verified on the target ESP32**.
Preserve a copy of the currently working YAML and use OTA rollback if needed.

## Project notes

This component was coded through Codex.

It is released under the MIT licence.

MeshCore Home Assistant integration:
[meshcore-dev/meshcore-ha](https://github.com/meshcore-dev/meshcore-ha)
