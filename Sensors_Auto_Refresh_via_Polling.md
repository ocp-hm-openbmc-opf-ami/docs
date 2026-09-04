# Sensors Page Auto-Refresh via Polling

**Target:** OpenBMC — OpenBMC Web UI & bmcweb  
**Last updated:** 2026-08-25

## Summary

This design adds automatic periodic data refresh (polling) to the Web UI
**Sensors page** (Hardware Status → Sensors). A new OEM Redfish endpoint
aggregates all sensor readings, thresholds, and health into a single response.
The frontend polls this endpoint every 10 seconds when the existing Sync
toggle is active, applying incremental updates to avoid full-page re-renders.

## Background

The existing Dashboard polling feature introduced:

- A **Sync button** in the AppHeader that toggles 10-second polling
- A backend mechanism to **skip session-timer reset** for polling URLs
- Persistence of the polling toggle state in `localStorage`

The Sensors page currently loads data once at page open using the standard
Redfish path, which issues one request per chassis plus one request per
individual sensor to retrieve threshold details. For systems with 100+ sensors,
this produces hundreds of HTTP requests and provides no auto-refresh.

## Requirements

- Provide a single API endpoint that returns all sensor data (readings,
  units, thresholds, health, availability) in one response.
- Automatically refresh sensor data every 10 seconds when the Sync toggle is
  enabled, reusing the existing toggle from the Dashboard feature.
- Apply incremental frontend updates so only changed sensors trigger
  re-renders, preserving table scroll position, sort state, and selections.
- Ensure sensor polling does not reset the session-activity timestamp,
  consistent with Dashboard polling behavior.
- Clean up polling intervals when the user navigates away from the Sensors
  page or disables the Sync toggle.

## Proposed Design

### OEM SensorsSummary Endpoint

bmcweb will expose a new OEM endpoint:

`GET /redfish/v1/Oem/Ami/SensorsSummary`

The endpoint queries D-Bus for all objects under `/xyz/openbmc_project/sensors`
implementing `xyz.openbmc_project.Sensor.Value`, retrieves all properties in
bulk, and returns a single JSON response. Login privilege is required.

**Response structure:**

```json
{
  "@odata.id": "/redfish/v1/Oem/Ami/SensorsSummary",
  "@odata.type": "#OemSensorsSummary.v1_0_0.OemSensorsSummary",
  "Name": "Sensors Summary",
  "Sensors": [
    {
      "Name": "inlet_temp",
      "Reading": 28.5,
      "ReadingUnits": "Cel",
      "Status": { "State": "Enabled", "Health": "OK" },
      "Thresholds": {
        "UpperCaution":  { "Reading": 40.0 },
        "UpperCritical": { "Reading": 45.0 },
        "UpperFatal":    { "Reading": 50.0 },
        "LowerCaution":  { "Reading": 5.0 },
        "LowerCritical": { "Reading": 0.0 },
        "LowerFatal":    { "Reading": -5.0 }
      }
    }
  ]
}
```

**D-Bus to Redfish property mapping:**

| D-Bus Property | Redfish JSON Field |
|---|---|
| `Value` | `Reading` |
| `Unit` (suffix matched) | `ReadingUnits` |
| `Available` | `Status.State` (`Enabled` / `Disabled`) |
| `WarningHigh` / `WarningLow` | `Thresholds.UpperCaution` / `LowerCaution` |
| `CriticalHigh` / `CriticalLow` | `Thresholds.UpperCritical` / `LowerCritical` |
| `NonRecoverableHigh` / `NonRecoverableLow` | `Thresholds.UpperFatal` / `LowerFatal` |
| `WarningAlarmHigh` / `WarningAlarmLow` | `Status.Health` → `"Warning"` |
| `CriticalAlarmHigh` / `CriticalAlarmLow` | `Status.Health` → `"Critical"` |

**Unit mapping:** `DegreesC` → `Cel`, `Volts` → `V`, `Amperes` → `A`,
`Watts` → `W`, `RPMS` → `RPM`, `Percent` → `%`.

**Health priority:** Critical overrides Warning. If no alarm properties are
asserted, health defaults to `OK`.

### Session-Keepalive Exclusion

The authentication layer must be extended to recognize polling URLs. When a
request targets any of the following, the session `lastUpdated` timestamp must
**not** be reset:

- `/redfish/v1/Oem/Ami/Dashboard` (existing)
- `/redfish/v1/Oem/Ami/SensorsSummary` (new)
- `/redfish/v1/Chassis/{ChassisId}/Sensors` (new, pattern match)

URL normalization must strip trailing slashes and query parameters before
comparison.

### Frontend Polling Integration

The AppHeader Sync toggle must broadcast a global `polling-toggled` event so
that any page — not just the Dashboard — can start or stop its own polling
cycle.

The Sensors page must:

1. On creation, check `localStorage('pollingEnabled')` and start a 10-second
   polling interval if active. Register a listener for the `polling-toggled`
   event.
2. On each poll, call the SensorsSummary endpoint and apply **incremental
   updates**: compare each sensor's reading, status, and thresholds against
   the cached value, and commit to the Vuex store only if something changed.
   Remove sensors that are no longer present in the response.
3. On destroy (page navigation), clear the polling interval and remove the
   event listener.

The initial page load must also use the SensorsSummary endpoint instead of the
existing multi-request `getAllSensors` path.

The polling flow (Sync toggle → event broadcast → interval → API call →
session exclusion) is identical to the existing Dashboard polling mechanism.

## Alternatives Considered

- **Poll individual Redfish sensor resources** — Rejected because it produces
  one HTTP request per sensor, which is the same N+1 problem the feature is
  solving.
- **Full array replacement on each poll** — Simpler frontend logic but causes
  the entire table to re-render, losing scroll/sort/selection state.

## Validation Plan

- **Polling interval confirmed** — Login, open the Sensors page, enable Sync,
  and open the browser Network tab. Verify that a
  `GET /redfish/v1/Oem/Ami/SensorsSummary` request is made every 10 seconds.
- **Threshold update reflected** — With polling active, modify a sensor's
  threshold value on the BMC. Verify the Sensors page displays the updated
  threshold within the next poll cycle without a manual page reload.
- **Sensor rename reflected** — With polling active, rename a sensor on the
  BMC. Verify the old name disappears and the new name appears on the Sensors
  page within the next poll cycle.
- **Sensor removal reflected** — With polling active, delete a sensor from the
  BMC. Verify the sensor row is removed from the Sensors page within the next
  poll cycle.
- **New sensor reflected** — With polling active, add a new sensor on the BMC.
  Verify the new sensor appears on the Sensors page within the next poll
  cycle.
- **Polling stops on disable** — With polling active, click Sync to disable
  it. Verify in the Network tab that no further SensorsSummary requests are
  made and the sensor values on the page stop updating.
