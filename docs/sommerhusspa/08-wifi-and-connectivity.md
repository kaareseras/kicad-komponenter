# Wi-Fi and Connectivity Guide

## 1. Wi-Fi architecture overview

The Sommerhusspa controller uses an ESP32-S3-WROOM-1U-N8R8 as the main connectivity endpoint for local and remote access. Wi-Fi is the primary communications channel during normal operation. BLE should be enabled only for commissioning and provisioning, not for continuous operation.

Recommended architecture:

- The ESP32 joins the customer’s 2.4 GHz Wi-Fi network as a station.
- The controller maintains an outbound connection to an MQTT broker for commands, status, telemetry, and diagnostics.
- A local HTTP/REST interface is exposed on the LAN for same-network access, setup fallback, and service diagnostics.
- OTA firmware updates are fetched over Wi-Fi from a trusted update server.
- BLE is used only to provision Wi-Fi credentials and basic site settings during installation.

Logical connectivity path:

1. Installer provisions the controller over BLE or captive portal.
2. The controller joins Wi-Fi.
3. The controller establishes MQTT and optional HTTPS connections.
4. The controller publishes state, telemetry, and faults.
5. Remote applications issue commands through MQTT.
6. The controller performs OTA update checks and downloads when authorised.

Recommended network dependencies:

- DHCP for IP address assignment.
- DNS for broker and update server resolution.
- NTP for accurate timestamps, logs, and certificate validation.
- MQTT broker reachable on the local network or through a secured internet endpoint.

## 2. Normal operation connectivity

During normal operation, the controller should remain connected to Wi-Fi continuously. This is not a battery-powered device, so the priority should be responsiveness, OTA reliability, and stable telemetry rather than aggressive power saving.

What should stay connected:

- Wi-Fi station connection.
- MQTT session to the broker.
- Local HTTP server, if enabled.
- Time synchronisation service at periodic intervals.

What should not stay active continuously:

- BLE radio. Disable it after provisioning or after a short commissioning timeout.
- Open access point mode. Enable only during first-time setup or recovery.

Recommended runtime behaviour:

- Connect to Wi-Fi during boot.
- Once Wi-Fi is up, resolve broker hostname and connect to MQTT.
- Subscribe to command and configuration topics before declaring the controller online.
- Publish an online birth message and current retained state after successful connection.
- If Wi-Fi drops, continue safe local control of spa functions autonomously.
- If MQTT drops but Wi-Fi remains up, keep attempting MQTT reconnection without interrupting spa control logic.

Operational principle:

- Connectivity improves usability and diagnostics.
- Safety-critical control must not depend on cloud reachability.
- Heater, circulation, fill, and drain interlocks must continue to run locally according to firmware logic even when the network is unavailable.

## 3. Provisioning flow

### 3.1 BLE provisioning option

BLE is the preferred commissioning method because it is simpler for installers and avoids requiring the controller to expose an unsecured setup network during normal installation.

Recommended BLE provisioning flow:

1. Device boots in unprovisioned mode.
2. BLE advertising is enabled with a clear commissioning name, for example `sommerhusspa-setup-34AF`.
3. An installer app connects over BLE.
4. The app writes:
   - Wi-Fi SSID
   - Wi-Fi passphrase
   - MQTT broker hostname or IP
   - MQTT port
   - Device name or site identifier
   - Optional MQTT username
   - Optional MQTT password or token
5. The controller validates the payload.
6. The controller stores credentials securely in non-volatile storage.
7. The controller attempts Wi-Fi connection.
8. On success, the controller disables BLE and enters normal operation.
9. On failure, the controller reports the error over BLE and keeps provisioning mode available.

BLE should support at minimum:

- Read device serial number and current provisioning state.
- Write provisioning bundle.
- Trigger connectivity test.
- Read test result.
- Factory reset of network settings.

### 3.2 Fallback: open AP and captive portal

If BLE provisioning is not available, or if a field technician needs a recovery path, the controller should support a fallback setup AP mode.

Recommended fallback behaviour:

- Start an open or minimally protected AP only when:
  - the device has no valid Wi-Fi credentials,
  - the installer presses a physical setup button, or
  - repeated Wi-Fi failures exceed a defined threshold.
- SSID example: `Sommerhusspa-Setup-34AF`.
- Present a captive portal or a simple local setup page at a fixed address such as `192.168.4.1`.
- Collect the same provisioning information as the BLE flow.
- Test credentials before committing them if possible.
- Shut down AP mode automatically after successful provisioning.

Practical recommendation:

- If an open AP is used, enforce a short setup timeout, for example 10 to 15 minutes.
- If product requirements allow it, WPA2-protect the setup AP with a per-device password printed on a label. This is better than a permanently open network.

### 3.3 What credentials to store and where

Store only the credentials needed for autonomous reconnect and service recovery.

Recommended stored items:

- Wi-Fi SSID.
- Wi-Fi passphrase.
- MQTT broker hostname.
- MQTT port.
- MQTT client ID.
- MQTT username, if used.
- MQTT password or token, if used.
- Device identifier.
- TLS server CA certificate or certificate fingerprint, if TLS is used.
- OTA update endpoint URL.
- OTA signing public key or firmware verification metadata.

Storage recommendations on ESP32:

- Use ESP-IDF NVS for configuration data.
- Use NVS encryption if enabled in the platform configuration.
- Do not hard-code site-specific credentials in firmware.
- Keep factory defaults separate from runtime configuration.
- Store secrets in a dedicated namespace, for example:
  - `wifi.ssid`
  - `wifi.psk`
  - `mqtt.host`
  - `mqtt.user`
  - `mqtt.pass`
  - `ota.url`

Avoid storing:

- Plaintext installer notes.
- Temporary BLE session keys.
- Historical passwords unless explicitly required.

## 4. MQTT design

### 4.1 Recommended broker setup

MQTT is the right transport for this controller because it is lightweight, reliable, and fits command-and-state messaging well.

Recommended broker options:

- Local broker on the customer site for best resilience and low latency.
- Bridged or cloud-accessible broker for remote support and app access.
- Mosquitto or EMQX are both suitable.

Recommended deployment model:

- Primary broker reachable on the local network.
- Optional secure remote broker or bridge for off-site access.
- One unique client ID per controller.
- Per-device credentials and ACLs restricting topic access to that device namespace.

Example device identity:

- Site ID: `summerhouse-a`
- Device ID: `spa-01`
- Client ID: `sommerhusspa-summerhouse-a-spa-01`

Topic namespace example:

```text
sommerhusspa/summerhouse-a/spa-01/
```

### 4.2 Topics

Recommended topics:

```text
sommerhusspa/{site_id}/{device_id}/cmd
sommerhusspa/{site_id}/{device_id}/cmd/ota
sommerhusspa/{site_id}/{device_id}/ack
sommerhusspa/{site_id}/{device_id}/state
sommerhusspa/{site_id}/{device_id}/telemetry
sommerhusspa/{site_id}/{device_id}/telemetry/energy
sommerhusspa/{site_id}/{device_id}/fault
sommerhusspa/{site_id}/{device_id}/availability
sommerhusspa/{site_id}/{device_id}/info
```

Purpose of each topic:

- `cmd`: operational commands from app or backend.
- `cmd/ota`: OTA trigger or policy commands.
- `ack`: command acknowledgement and execution result.
- `state`: current retained operational state.
- `telemetry`: periodic measurements and runtime metrics.
- `telemetry/energy`: ADS131M08-derived energy and power values.
- `fault`: active faults, warnings, and clears.
- `availability`: online or offline presence.
- `info`: static or infrequently changing metadata such as firmware version and hardware revision.

### 4.3 QoS recommendations

Recommended QoS by topic class:

- `cmd`: QoS 1.
- `cmd/ota`: QoS 1.
- `ack`: QoS 1.
- `state`: QoS 1.
- `telemetry`: QoS 0 for frequent updates, or QoS 1 for slower summary updates.
- `telemetry/energy`: QoS 0 for high-rate values, QoS 1 for periodic summaries.
- `fault`: QoS 1.
- `availability`: QoS 1.
- `info`: QoS 1.

Reasoning:

- Commands must be delivered at least once, so QoS 1 is appropriate.
- The firmware must make command handling idempotent because QoS 1 may redeliver.
- Fast telemetry should use QoS 0 to reduce broker load and latency.
- Faults and retained state should use QoS 1.

### 4.4 Retained messages

Use retained messages selectively for last-known-state and discovery.

Recommended retained topics:

- `state`: yes, retained.
- `availability`: yes, retained with LWT.
- `info`: yes, retained.
- `fault`: optionally retained for active uncleared faults only.
- `telemetry`: generally not retained.
- `telemetry/energy`: not retained for streaming data, but retained summaries are acceptable.

Recommended availability pattern:

- Publish `online` retained when connected.
- Configure MQTT last will to publish `offline` retained if the device disconnects unexpectedly.

Example payload:

```json
{"status":"online","ts":"2026-05-10T01:00:00Z"}
```

## 5. Remote command interface

### 5.1 Supported commands

The remote command interface should expose operational commands that are useful, bounded, and safe.

Recommended commands:

- Heater on or off.
- Jet pump on or off.
- Circulation pump mode or enable state, if user-controlled.
- UV enable or service override, only if operationally appropriate.
- Fill valve open, close, or timed fill.
- Drain valve open, close, or timed drain.
- Light on or off, and optional brightness or scene.
- Temperature setpoint update.
- OTA check or OTA install.
- Ping or state refresh request.

Commands that should be constrained locally:

- Fill and drain should enforce maximum run times.
- Heater enable should remain subject to all local interlocks.
- Jet and heater combinations should obey power and thermal rules.
- Dangerous or service-only operations should require a higher privilege level.

### 5.2 Command format

Use JSON payloads over the `cmd` topic.

Example topic:

```text
sommerhusspa/summerhouse-a/spa-01/cmd
```

Example command message:

```json
{
  "msg_id": "c7df2d70-2f5c-4cf1-90d7-73520f0a7f10",
  "ts": "2026-05-10T01:05:00Z",
  "source": "mobile-app",
  "command": "set_heater",
  "params": {
    "enabled": true
  }
}
```

Example temperature setpoint command:

```json
{
  "msg_id": "ef4aa13d-f2c1-4f08-8dd7-5ecba7a1b183",
  "ts": "2026-05-10T01:05:15Z",
  "source": "mobile-app",
  "command": "set_temperature",
  "params": {
    "setpoint_c": 38.5
  }
}
```

Example timed fill command:

```json
{
  "msg_id": "8b76a4a7-8e26-4e68-ae3b-59f9c8b42a2f",
  "ts": "2026-05-10T01:05:30Z",
  "source": "service-app",
  "command": "fill",
  "params": {
    "action": "start",
    "timeout_s": 120
  }
}
```

Recommended command names:

- `set_heater`
- `set_jet`
- `set_circulation`
- `set_uv`
- `fill`
- `drain`
- `set_light`
- `set_temperature`
- `request_state`
- `ota_check`
- `ota_install`

### 5.3 Acknowledgement behaviour

Every accepted or rejected command should result in an acknowledgement on the `ack` topic.

Acknowledgement stages should be explicit:

- `accepted`: syntax and permissions are valid; execution queued.
- `applied`: command executed successfully.
- `rejected`: invalid command, invalid parameters, or not permitted.
- `failed`: command accepted but execution failed.
- `superseded`: replaced by a newer command.

Example acknowledgement:

```json
{
  "msg_id": "c7df2d70-2f5c-4cf1-90d7-73520f0a7f10",
  "ack_ts": "2026-05-10T01:05:01Z",
  "status": "applied",
  "command": "set_heater",
  "result": {
    "enabled": true
  }
}
```

Example rejection:

```json
{
  "msg_id": "8b76a4a7-8e26-4e68-ae3b-59f9c8b42a2f",
  "ack_ts": "2026-05-10T01:05:31Z",
  "status": "rejected",
  "command": "fill",
  "error": {
    "code": "INTERLOCK_ACTIVE",
    "message": "Fill operation is not permitted while drain valve is open."
  }
}
```

Implementation recommendation:

- Require `msg_id` in every command.
- Cache recently processed `msg_id` values for duplicate detection.
- Publish resulting state update after `applied`.

## 6. Status telemetry

### 6.1 What to report and how often

Telemetry should provide enough visibility for remote support and automation without flooding the broker.

Recommended status categories:

- Connectivity status.
- Current actuator states.
- Water temperature and setpoint.
- Estimated heating state and demand.
- Supply voltage and internal health metrics.
- Fault and warning status.
- Runtime counters.
- Energy metering values from ADS131M08.

Suggested publication rates:

- State changes: immediately on change.
- Main status summary: every 30 to 60 seconds.
- Fast-changing electrical telemetry: every 5 to 10 seconds, if needed.
- Energy summary totals: every 60 seconds.
- Faults: immediately on raise and clear.
- Availability: on connect and disconnect.

Example `state` payload:

```json
{
  "ts": "2026-05-10T01:06:00Z",
  "heater": true,
  "jet": false,
  "circulation": true,
  "uv": true,
  "fill_valve": false,
  "drain_valve": false,
  "light": true,
  "temperature_c": 37.8,
  "setpoint_c": 38.5,
  "mode": "normal",
  "fault_active": false
}
```

Example `telemetry` payload:

```json
{
  "ts": "2026-05-10T01:06:00Z",
  "wifi_rssi_dbm": -59,
  "uptime_s": 86400,
  "heap_free_bytes": 182304,
  "ip": "192.168.1.45",
  "fw_version": "1.3.2",
  "water_temp_c": 37.8,
  "pcb_temp_c": 41.2
}
```

### 6.2 Energy metering data publication

The ADS131M08 can provide valuable verification that loads behave as expected. Publish both instantaneous values and summary totals.

Recommended energy telemetry fields:

- RMS voltage.
- RMS current per measured channel.
- Active power.
- Apparent power.
- Power factor.
- Frequency, if derived.
- Accumulated energy in Wh or kWh.
- Derived load identification or verification flags.

Example `telemetry/energy` payload:

```json
{
  "ts": "2026-05-10T01:06:05Z",
  "mains": {
    "voltage_rms_v": 229.8,
    "frequency_hz": 50.0
  },
  "channels": {
    "heater": {
      "current_rms_a": 8.6,
      "active_power_w": 1970,
      "power_factor": 0.99,
      "energy_wh_today": 6420
    },
    "jet": {
      "current_rms_a": 0.0,
      "active_power_w": 0,
      "power_factor": 0.00,
      "energy_wh_today": 320
    },
    "circulation": {
      "current_rms_a": 0.7,
      "active_power_w": 120,
      "power_factor": 0.76,
      "energy_wh_today": 910
    }
  },
  "verification": {
    "heater_commanded": true,
    "heater_power_detected": true,
    "jet_commanded": false,
    "jet_power_detected": false
  }
}
```

Practical recommendation:

- Publish raw metering at a controlled rate such as every 5 seconds.
- Publish daily or hourly summaries separately if long-term trending is needed.
- Use metering to detect stuck relays, failed heaters, or pumps that are commanded on but not drawing expected current.

### 6.3 Fault and error reporting

Fault reporting should be structured and machine-readable.

Recommended fault conditions:

- Overtemperature.
- Temperature sensor failure.
- Dry-run protection event.
- Fill timeout.
- Drain timeout.
- Heater current mismatch.
- Pump current mismatch.
- Wi-Fi reconnect storm.
- MQTT authentication failure.
- OTA verification failure.

Example fault message:

```json
{
  "ts": "2026-05-10T01:07:00Z",
  "level": "error",
  "code": "HEATER_CURRENT_MISMATCH",
  "message": "Heater command is active but measured power is below expected threshold.",
  "active": true,
  "details": {
    "commanded": true,
    "active_power_w": 42,
    "expected_min_w": 1500
  }
}
```

Use the same fault code when clearing the condition, with `active: false`.

## 7. Over-the-air update procedure

OTA is required and should be treated as a first-class operational function.

### 7.1 How firmware images are served

Recommended approaches:

- HTTPS download from a trusted update server.
- Pre-signed URL with limited lifetime.
- Update manifest JSON describing available firmware.

Example manifest URL:

```text
https://updates.example.com/sommerhusspa/spa-01/manifest.json
```

Example manifest:

```json
{
  "product": "sommerhusspa-controller",
  "hardware_rev": "A",
  "current_min_version": "1.2.0",
  "latest_version": "1.3.3",
  "url": "https://updates.example.com/sommerhusspa/firmware/1.3.3.bin",
  "sha256": "3d7d6f8b4c5a2b9d0f9f25c0fbb9f78e3b6d1f4e5f2d6a9a4d8c7b1e2f3a4b5c",
  "size": 1834528,
  "signed": true
}
```

### 7.2 Version negotiation

Recommended OTA flow:

1. Device publishes current firmware version and hardware revision in `info`.
2. Device periodically checks manifest, for example once every 24 hours, or on explicit `ota_check` command.
3. Device compares:
   - product name,
   - hardware revision,
   - semantic version,
   - minimum compatible version.
4. Device downloads update metadata.
5. Device verifies signature and hash before activation.
6. Device performs OTA only when system state is safe.

Safe-state recommendations:

- Do not begin OTA during active fill or drain.
- Prefer not to start OTA while a critical heating transition is in progress.
- If necessary, wait for idle or maintenance-safe mode.

### 7.3 Rollback strategy if update fails

Use ESP32 dual-partition OTA with image validation and rollback.

Recommended rollback behaviour:

- Download image into inactive OTA partition.
- Verify hash and signature before marking bootable.
- Reboot into new firmware.
- New firmware must perform self-test and confirm health within a timeout.
- If boot fails, watchdog resets, or health confirmation is not sent, automatically roll back to the previous image.

Health confirmation checks should include:

- NVS configuration readable.
- Main control task starts.
- Sensor interfaces initialise.
- Wi-Fi connection succeeds within a reasonable window.
- MQTT reconnect succeeds, if configured.

Recommended OTA status reporting on MQTT:

```json
{
  "ts": "2026-05-10T01:10:00Z",
  "status": "downloading",
  "target_version": "1.3.3",
  "progress_pct": 42
}
```

Additional statuses:

- `available`
- `downloading`
- `verifying`
- `scheduled`
- `rebooting`
- `success`
- `rollback`
- `failed`

## 8. Wi-Fi power management considerations

### 8.1 Keeping the ESP connected during normal operation

For a spa controller, always-on connectivity is usually the correct design choice.

Recommendations:

- Keep Wi-Fi active continuously during powered operation.
- Use modem sleep only if it does not materially harm MQTT responsiveness.
- Disable aggressive sleep modes that increase latency or destabilise broker connectivity.
- Tune keepalive intervals to suit the broker and network, for example 30 to 60 seconds.

Because this device controls heating and water handling equipment, responsiveness and reliable telemetry matter more than saving a small amount of power.

### 8.2 Disconnection handling and reconnect strategy

The controller should degrade gracefully when connectivity is interrupted.

Recommended reconnect strategy:

- Detect Wi-Fi disconnect events immediately.
- Continue local autonomous control.
- Retry Wi-Fi with exponential backoff and jitter, for example 1 s, 2 s, 5 s, 10 s, 30 s, then capped retries every 60 s.
- Once Wi-Fi is restored, reconnect MQTT and resubscribe.
- Republish retained `availability`, `info`, and `state` after reconnect.

Add recovery escalation:

- If Wi-Fi fails for an extended period, log a warning and optionally re-enable provisioning mode only through a physical service action.
- Avoid automatically exposing a setup AP during transient outages, as that can create security problems and confusion.

### 8.3 Deep-sleep versus always-on tradeoffs

Deep sleep is generally not appropriate for this product during normal operation.

Why always-on is preferred:

- The spa controller must react to commands promptly.
- OTA updates require a stable online session.
- Continuous telemetry and fault reporting are expected.
- Reconnection delays after waking are unnecessary on mains power.

Deep sleep may be acceptable only for specialised modes such as:

- warehouse storage mode,
- shipping mode,
- service bench mode.

## 9. Network security recommendations

### 9.1 TLS for MQTT if accessible from the internet

If MQTT traffic ever traverses the public internet or an untrusted network, use TLS.

Recommendations:

- Use MQTTS on port 8883.
- Validate the broker certificate against a pinned CA or known trust anchor.
- Prefer per-device credentials.
- Consider mutual TLS for higher-security installations.

If the broker is local-only on a trusted LAN, non-TLS MQTT may be acceptable in some installations, but TLS is still preferable when feasible.

### 9.2 API authentication options

Recommended authentication options:

- MQTT username and password per device.
- Token-based authentication for app-to-backend layers.
- mTLS for device-to-broker in high-security deployments.
- Local REST API token or password for service access.

Practical recommendation:

- Separate device credentials from user credentials.
- Do not let mobile apps connect directly as an all-powerful broker admin.
- Use broker ACLs so each device can only publish and subscribe to its own namespace.

### 9.3 Firewall considerations

Recommended firewall policy:

- Allow outbound connections from the controller to:
  - DNS,
  - NTP,
  - MQTT or MQTTS broker,
  - HTTPS OTA server.
- Block unsolicited inbound internet access to the controller.
- Expose local REST UI only on the LAN.
- Avoid port forwarding directly to the controller.

Best practice:

- Remote access should go through a secure broker, VPN, or reverse proxy under operator control.
- Do not rely on an internet-exposed device web server as the primary remote management path.

## 10. Local API fallback (REST/HTTP for same-network access)

A local HTTP API is useful as a secondary interface for service tools, local dashboards, and fallback control when MQTT infrastructure is unavailable.

Recommended scope of the local API:

- Read device info.
- Read current state.
- Read recent faults.
- Trigger safe commands.
- Trigger provisioning reset or network test, if authenticated.
- Trigger OTA check, if authenticated.

Example endpoints:

```text
GET  /api/v1/info
GET  /api/v1/state
GET  /api/v1/telemetry
GET  /api/v1/faults
POST /api/v1/command
POST /api/v1/ota/check
POST /api/v1/network/test
```

Example command request:

```json
{
  "command": "set_light",
  "params": {
    "enabled": true
  }
}
```

Example state response:

```json
{
  "device_id": "spa-01",
  "fw_version": "1.3.2",
  "wifi": {
    "connected": true,
    "ip": "192.168.1.45",
    "rssi_dbm": -59
  },
  "state": {
    "heater": true,
    "jet": false,
    "circulation": true,
    "uv": true,
    "fill_valve": false,
    "drain_valve": false,
    "light": true,
    "temperature_c": 37.8,
    "setpoint_c": 38.5
  }
}
```

Recommendations for the local API:

- Bind only to the LAN interface.
- Require authentication for write operations.
- Use HTTPS if practical; otherwise keep it LAN-only and document the risk.
- Keep the API simple and stable.
- Treat REST as a fallback and service interface, not the primary event transport.

## Summary of recommended defaults

For implementation, the following defaults are a sensible starting point:

- Wi-Fi mode: station, always connected.
- BLE: enabled only for commissioning.
- Fallback AP: only on first boot, explicit service action, or repeated setup failure.
- MQTT broker: local or securely remote, QoS 1 for commands and state.
- Retained topics: `availability`, `state`, and `info`.
- Telemetry cadence: 30 to 60 second summaries, 5 to 10 second energy updates where useful.
- OTA: HTTPS manifest plus signed firmware, dual-slot rollback enabled.
- Security: per-device credentials, topic ACLs, TLS whenever traffic leaves the trusted LAN.
- Local API: HTTP or HTTPS on LAN for fallback diagnostics and service use.

This approach gives the Sommerhusspa controller reliable remote control, practical field commissioning, safe local autonomy, and a clean path for diagnostics and firmware maintenance.