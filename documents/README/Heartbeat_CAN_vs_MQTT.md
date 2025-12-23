# Heartbeat Mechanism: CAN Bus vs MQTT/Victron

## Overview

The heartbeat mechanism ensures reliable communication between YamBMS and the inverter. This document explains how it works in both CAN bus and MQTT/Victron implementations.

## CAN Bus Heartbeat (Bidirectional)

### How It Works

```
YamBMS              CAN Bus              Inverter
  |                                         |
  |--- Send battery data (0x351, etc.) --->|
  |                                         |
  |<-- ACK (0x305) -------------------------|
  |                                         |
  [Restart timeout timer]                  |
  [Enable data transmission]               |
  |                                         |
  |--- Continue sending data ------------->|
  |                                         |
  [If no ACK for 10 seconds...]            |
  [Disable data transmission]              |
  [Set status: OFFLINE]                    |
```

### Implementation Details

**Key Components:**
- **Inverter ACK**: 0x305 CAN frame sent by inverter
- **Timeout Timer**: Configurable (default ~10 seconds)
- **Send Enable Flag**: `send_canbus_data` boolean
- **Retry Mechanism**: Every 60 seconds

**Code Flow** (yambms_canbus.yaml):

```cpp
// When 0x305 received from inverter:
on_frame:
  - can_id: 0x305
    then:
      - lambda: |-
          // Restart timeout timer
          id(canbus_script_link_timer).execute();

          // Enable transmission
          id(send_canbus_data) = true;
          id(canbus_status).publish_state(true);
          id(inverter_com_status).publish_state(true);

          // Track heartbeat interval
          uint32_t interval_ms = millis() - previous_ack_ms;
          id(inverter_heartbeat).publish_state(interval_ms);

// Timeout script (runs after delay with no ACK):
script:
  - id: canbus_script_link_timer
    mode: restart
    then:
      - delay: 10s  # Or configured timeout
      - lambda: |-
          // Disable transmission
          id(send_canbus_data) = false;
          id(canbus_status).publish_state(false);
          id(inverter_com_status).publish_state(false);

// Retry mechanism:
interval:
  - interval: 60s
    then:
      - lambda: |-
          // If disabled, try re-enabling
          if (id(send_canbus_data) == false) {
            id(send_canbus_data) = true;
            id(canbus_script_link_timer).execute();
          }
```

**Behavior:**
1. YamBMS sends CAN frames at 100ms intervals
2. Inverter receives data and sends 0x305 ACK (typically every 1-5 seconds)
3. On ACK reception:
   - Timer restarts (10-second countdown)
   - Transmission stays enabled
   - Heartbeat interval tracked
4. If no ACK for 10 seconds:
   - Timer expires
   - Transmission disabled (saves CPU)
   - Status set to offline
5. Every 60 seconds:
   - Retry by re-enabling transmission
   - If inverter comes back online, ACK resumes cycle

**Sensors Provided:**
- `Inverter Heartbeat` (ms): Interval between ACKs
- `CANBUS Status` (binary): Online/offline
- `Inverter Com Status` (binary): Communication active
- `Inverter Heartbeat Monitoring` (switch): Enable/disable tracking

## MQTT/Victron Heartbeat (Unidirectional)

### How It Works

```
YamBMS              MQTT Broker          Victron Cerbo
  |                      |                     |
  |--- Publish data ---->|                     |
  |     (1 Hz)           |                     |
  |                      |<-- Subscribe -------|
  |                      |--- Data ----------->|
  |                      |                     |
  [Track publish success]                     |
  [Monitor MQTT connection]                   |
  |                      |                     |
  [If broker offline...]                      |
  [Disable publishing]                        |
  [Set status: OFFLINE]                       |
```

### Key Differences

| Aspect | CAN Bus | MQTT/Victron |
|--------|---------|--------------|
| Direction | Bidirectional | Unidirectional |
| ACK Source | Inverter (0x305) | MQTT Broker (publish success) |
| Failure Detection | No ACK received | MQTT disconnect or publish fail |
| Timeout | 10 seconds | 30 seconds |
| Retry | Every 60 seconds | Automatic on reconnect |
| Victron Feedback | Direct (CAN ACK) | Indirect (broker only) |

### Implementation Details

**Key Components:**
- **Publish Success**: MQTT client return value
- **Broker Connection**: MQTT connected state
- **Publish Enable Flag**: `victron_publish_enabled` boolean
- **Heartbeat Monitor**: Every 5 seconds

**Code Flow** (inverter_victron_mqtt.yaml v1.1.0):

```cpp
// Main publishing interval (1 Hz):
interval:
  - interval: 1s
    then:
      - lambda: |-
          // Check publishing enabled
          if (!id(victron_publish_enabled)) {
            return;  // Skip if disabled
          }

          // Check MQTT connection
          if (!id(mqtt_client).is_connected()) {
            id(victron_publish_error_count)++;
            return;
          }

          // Build JSON payload
          std::string payload = {...};

          // Publish and track success
          bool success = id(mqtt_client).publish(topic, payload);

          if (success) {
            id(victron_last_publish_ms) = millis();
            id(victron_publish_count)++;
          } else {
            id(victron_publish_error_count)++;
          }

// Heartbeat monitoring (5 second interval):
interval:
  - interval: 5s
    then:
      - lambda: |-
          bool mqtt_connected = id(mqtt_client).is_connected();
          unsigned long now = millis();
          unsigned long last_publish = id(victron_last_publish_ms);

          // Auto-enable if broker available
          if (mqtt_connected && have_bms_data) {
            if (!id(victron_publish_enabled)) {
              // Re-enable publishing
              id(victron_publish_enabled) = true;
            }
          }
          // Auto-disable if offline too long
          else if (!mqtt_connected || (now - last_publish) > 30000) {
            if (id(victron_publish_enabled)) {
              // Disable publishing
              id(victron_publish_enabled) = false;
            }
          }

          // Update status
          id(victron_mqtt_active).publish_state(mqtt_connected && enabled);
```

**Behavior:**
1. YamBMS publishes JSON to MQTT at 1 Hz
2. MQTT broker receives and queues for subscribers
3. Victron Cerbo (if connected) subscribes and reads data
4. On each publish:
   - Success tracked (timestamp, counter)
   - Failure tracked (error counter)
5. Every 5 seconds:
   - Check MQTT connection status
   - Check time since last successful publish
   - Auto-disable if offline >30 seconds
   - Auto-enable when reconnected
6. Manual override:
   - User can disable/enable via switch
   - Useful for testing or debugging

**Sensors Provided:**
- `Victron MQTT Active` (binary): Publishing enabled and broker connected
- `Victron MQTT Publishing` (switch): Manual enable/disable
- `Victron Publish Count` (sensor): Total successful publishes
- `Victron Publish Errors` (sensor): Total failed publishes
- `Victron Heartbeat Interval` (ms): Time since last publish
- `Victron MQTT Status` (text): Detailed status with error rate

## Why No Direct Victron ACK?

### Technical Reasons

**MQTT is Publish/Subscribe:**
- Victron is a subscriber, not a publisher
- No bidirectional handshake in MQTT model
- Broker handles offline clients transparently

**venus-os_dbus-mqtt-battery Driver:**
- Purely consumes MQTT data
- Doesn't publish ACKs back
- Forwards to Venus DBus only

**MQTT Broker Handles Offline:**
- Queues messages for offline subscribers
- Retains last message (if configured)
- Automatic reconnection handling

### Functional Equivalent

Even without direct Victron ACK, we achieve similar functionality:

| Function | CAN Bus | MQTT/Victron |
|----------|---------|--------------|
| Detect inverter offline | No 0x305 ACK | MQTT disconnect |
| Stop wasting resources | Disable CAN send | Disable MQTT publish |
| Status feedback | CANBUS Status sensor | Victron MQTT Active sensor |
| Heartbeat tracking | ACK interval (ms) | Publish interval (ms) |
| Auto-recovery | 60s retry | Reconnect auto-enable |
| Manual control | Heartbeat monitoring switch | Publishing switch |

## Monitoring Comparison

### CAN Bus Sensors

```yaml
binary_sensor:
  - CANBUS Status: ON/OFF
  - Inverter Com Status: ON/OFF
  - Inverter Heartbeat Monitoring: Enable/disable switch

sensor:
  - Inverter Heartbeat: 1234 ms (interval between ACKs)
```

**What you see:**
- "CANBUS Status: ON" = Inverter sending ACKs
- "Inverter Heartbeat: 2500 ms" = ACK every 2.5 seconds
- If offline: "CANBUS Status: OFF"

### MQTT/Victron Sensors (v1.1.0)

```yaml
binary_sensor:
  - Victron MQTT Active: ON/OFF

switch:
  - Victron MQTT Publishing: ON/OFF (manual control)

sensor:
  - Victron Publish Count: 3456 (total sent)
  - Victron Publish Errors: 12 (total failed)
  - Victron Heartbeat Interval: 1000 ms (time since last publish)

text_sensor:
  - Victron MQTT Status: "Publishing: 51.2V 12.5A 85% | Sent: 3456 | Errors: 12 (0.3%)"
```

**What you see:**
- "Victron MQTT Active: ON" = Broker connected, publishing enabled
- "Victron Heartbeat Interval: 1000 ms" = Published 1 second ago
- "Publish Count: 3456" = Total messages sent since boot
- "Publish Errors: 12" = Failed publishes (0.3% error rate)
- If offline: "Victron MQTT Active: OFF"

### Status Messages

**CAN Bus:**
```
[canbus] received can_id: 0x305 ack_interval 2500 ms
[canbus] send can_id: 0x351 hex: 40 02 38 01 64 00 ...
```

**MQTT/Victron:**
```
[victron_mqtt] Published battery data: 51.2V 12.5A 85% SOC
[victron_mqtt] MQTT reconnected - enabling publishing
[victron_mqtt] MQTT connection lost - disabling publishing
[victron_mqtt] Failed to publish battery data
```

## Use Cases

### Normal Operation

**CAN Bus:**
1. Inverter powered on
2. Sends 0x305 ACK every 1-5 seconds
3. YamBMS keeps sending data
4. Heartbeat shows stable interval

**MQTT/Victron:**
1. MQTT broker running
2. YamBMS publishes every 1 second
3. Victron subscribes and reads
4. Publish count increments steadily

### Inverter Powered Off

**CAN Bus:**
1. No 0x305 ACK received
2. After 10 seconds, timeout
3. Stop sending CAN data
4. "CANBUS Status: OFF"
5. Retry every 60 seconds

**MQTT/Victron:**
1. Broker still running (independent)
2. YamBMS continues publishing
3. Victron offline (can't subscribe)
4. Broker queues messages (if configured)
5. When Victron returns, catches up

### MQTT Broker Down

**CAN Bus:**
- Not affected (direct connection)
- Continues normally

**MQTT/Victron:**
1. MQTT connection fails
2. Publish errors increment
3. After 30 seconds, auto-disable
4. "Victron MQTT Active: OFF"
5. When broker returns, auto-enable

### Network Partition

**Scenario**: YamBMS can't reach broker, but Victron can

**CAN Bus:**
- Not applicable (direct physical connection)

**MQTT/Victron:**
1. YamBMS sees MQTT disconnect
2. Disables publishing
3. Victron waits for data
4. When network heals:
   - YamBMS reconnects
   - Auto-enables publishing
   - Victron receives current data

## Performance Impact

### CAN Bus

**Normal Operation:**
- Send: 10 frames @ 100ms = 10 frames/second
- Receive: 1 ACK @ 2500ms = 0.4 frames/second
- Total CAN bandwidth: ~200 bytes/second
- CPU: ~8-10%

**Offline (stopped):**
- Send: 0 frames
- Receive: 0 frames
- CPU: ~1% (timer checking)

### MQTT/Victron

**Normal Operation:**
- Publish: 1 message @ 1 Hz = ~500 bytes/second
- Network bandwidth: ~500 bytes/second
- CPU: ~5-8%

**Offline (broker down):**
- Publish attempts: Failed immediately
- Network bandwidth: ~0 (no connection)
- CPU: ~3% (connection retry)

**Offline (disabled):**
- Publish: Skipped
- CPU: ~1% (monitoring only)

## Best Practices

### CAN Bus

✅ **Do:**
- Monitor "Inverter Heartbeat" sensor
- Check logs for "received can_id: 0x305"
- Use heartbeat monitoring for diagnostics
- Adjust timeout for your inverter type

❌ **Don't:**
- Disable heartbeat monitoring without reason
- Set timeout too short (<5 seconds)
- Ignore "CANBUS Status: OFF" warnings

### MQTT/Victron

✅ **Do:**
- Monitor "Victron MQTT Active" sensor
- Check publish count increases
- Watch for publish errors
- Use detailed status text sensor
- Enable TLS/SSL for production
- Set up broker monitoring

❌ **Don't:**
- Disable publishing unless debugging
- Ignore high error rates (>5%)
- Run without broker reliability (HA, redundancy)
- Forget to check Victron side (driver logs)

## Troubleshooting

### CAN Bus: No Heartbeat

**Symptoms:**
- "Inverter Heartbeat: 0 ms"
- "CANBUS Status: OFF"
- Logs: No "received can_id: 0x305"

**Checks:**
1. Inverter powered on?
2. CAN bus wiring correct?
3. CAN termination resistors?
4. Correct baud rate (250kbps)?
5. Right CAN IDs (0x305 expected)?
6. Check inverter logs/display

**Fix:**
- Verify physical CAN connection
- Check inverter compatibility
- Try different protocol (PYLON, LuxPower, etc.)
- Increase timeout if inverter slow

### MQTT/Victron: Not Publishing

**Symptoms:**
- "Victron MQTT Active: OFF"
- "Publish Errors" increasing
- Logs: "MQTT disconnected"

**Checks:**
1. MQTT broker running?
2. Broker accessible (ping test)?
3. Correct credentials?
4. Topic permissions (ACLs)?
5. Network connectivity?
6. Firewall blocking port 1883/8883?

**Fix:**
- Restart MQTT broker
- Check secrets.yaml credentials
- Verify network connectivity
- Test with mosquitto_sub:
  ```bash
  mosquitto_sub -h broker -t "yambms/victron/battery" -v
  ```

### MQTT/Victron: Publishing but Victron Not Seeing

**Symptoms:**
- "Victron MQTT Active: ON"
- "Publish Count" increasing
- But Cerbo shows no battery

**Checks:**
1. venus-os_dbus-mqtt-battery driver installed?
2. Driver running? `svstat /service/dbus-mqtt-battery`
3. Correct topic in config.ini?
4. Driver logs: `tail -f /var/log/dbus-mqtt-battery/current`
5. JSON format correct?

**Fix:**
- Check driver configuration
- Verify topic matches (yambms/victron/battery)
- Restart driver: `svc -t /service/dbus-mqtt-battery`
- Test JSON format with MQTT Explorer

## Summary

Both implementations provide reliable heartbeat monitoring with different approaches:

**CAN Bus**: Traditional bidirectional handshake
- ✅ Direct inverter feedback
- ✅ Immediate offline detection
- ❌ Physical connection required
- ❌ Single inverter only

**MQTT/Victron**: Modern publish/subscribe model
- ✅ Network-based flexibility
- ✅ Multiple subscribers possible
- ✅ Broker handles offline gracefully
- ✅ Comprehensive error tracking
- ❌ No direct Victron ACK
- ✅ Broker connection monitoring equivalent

The MQTT implementation achieves **functionally equivalent** heartbeat monitoring through broker connection status, publish success tracking, and automatic enable/disable logic.

## References

**CAN Bus Implementation:**
- packages/yambms/yambms_canbus.yaml:120-180

**MQTT/Victron Implementation:**
- packages/inverter/inverter_victron_mqtt.yaml (v1.1.0)

**Documentation:**
- [venus-os_dbus-mqtt-battery](https://github.com/mr-manuel/venus-os_dbus-mqtt-battery)
- [Victron dbus-mqtt](https://github.com/victronenergy/dbus-mqtt)
- [Venus OS MQTT Guide](https://community.victronenergy.com/t/victron-venus-os-with-mqtt-sensors-switches-and-numbers/527931)

## Version History

- **1.0.0** (2025-12-23): Initial implementation (no heartbeat monitoring)
- **1.1.0** (2025-12-23): Added comprehensive heartbeat monitoring
  - Publish success/failure tracking
  - Auto-enable/disable on connection state
  - Manual publishing control switch
  - Error rate calculation
  - Detailed status sensors
