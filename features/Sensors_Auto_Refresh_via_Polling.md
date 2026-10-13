# Sensors Page Auto-Refresh via Polling

**Target:** OpenBMC — OpenBMC Web UI & bmcweb  
**Last updated:** 2026-09-17

## Summary

This design adds automatic periodic data refresh (polling) to the Web UI
**Sensors page** (Hardware Status → Sensors). Standard Redfish `$expand` queries
are utilized on the Chassis Sensors collections to retrieve all sensor readings,
thresholds, and health states in bulk. The frontend polls these standard
endpoints every 10 seconds when the existing Sync toggle is active, applying
incremental updates to avoid full-page re-renders.

## Background

The existing Dashboard polling feature introduced:

- A **Sync button** in the AppHeader that toggles 10-second polling
- A backend mechanism to **skip session-timer reset** for polling URLs
- Persistence of the polling toggle state in `localStorage`

The Sensors page loads data using standard Redfish endpoints. By leveraging
Redfish-standard expand queries (`?$expand=.($levels=1)`), all sensor data for
each chassis is fetched in a single request per chassis, eliminating multiple
per-sensor HTTP requests and enabling periodic auto-refresh.

## Requirements

- Use standard Redfish Chassis Sensors endpoints with level-1 `$expand`
  query (`/redfish/v1/Chassis/{chassisId}/Sensors?$expand=.($levels=1)`).
- Automatically refresh sensor data every 10 seconds when the Sync toggle is
  enabled, reusing the existing toggle from the Dashboard feature.
- Apply incremental frontend updates so only changed sensors trigger
  re-renders, preserving table scroll position, sort state, and selections.
- Ensure sensor polling does not reset the session-activity timestamp,
  consistent with Dashboard polling behavior.
- Clean up polling intervals when the user navigates away from the Sensors
  page or disables the Sync toggle.

## Proposed Design

### Standard Redfish Chassis Sensors with Expand

The Web UI queries the standard Redfish Chassis Sensors endpoint with expand:

`GET /redfish/v1/Chassis/{chassisId}/Sensors?$expand=.($levels=1)`

`bmcweb` handles this using its efficient expand implementation in
`redfish-core/lib/sensors.hpp`, returning complete sensor objects within the
`Members` array in a single response per chassis.

The WebUI discovers the `Sensors.@odata.id` link for every Chassis member and
caches valid sensor collection URLs. When `VUE_APP_ONETREE_PSM_ENABLED` is
enabled, it also includes the PowerShelf resource. Sync clears the cached URLs
so the current inventory is rediscovered after an Entity Manager configuration
change.

**Response structure:**

```json
{
  "@odata.id": "/redfish/v1/Chassis/chassis/Sensors",
  "@odata.type": "#SensorCollection.SensorCollection",
  "Name": "Sensors Collection",
  "Members@odata.count": 1,
  "Members": [
    {
      "@odata.id": "/redfish/v1/Chassis/chassis/Sensors/inlet_temp",
      "@odata.type": "#Sensor.v1_0_0.Sensor",
      "Id": "inlet_temp",
      "Name": "inlet_temp",
      "Reading": 28.5,
      "ReadingUnits": "Cel",
      "Status": {
        "State": "Enabled",
        "Health": "OK"
      },
      "Thresholds": {
        "UpperCaution":  { "Reading": 40.0 },
        "UpperCritical": { "Reading": 45.0 },
        "UpperFatal":    { "Reading": 50.0 },
        "LowerCaution":  { "Reading": 5.0 },
        "LowerCritical": { "Reading": 0.0 },
        "LowerFatal":    { "Reading": -5.0 }
      },
      "Oem": {
        "Ami": {
          "@odata.id": "/redfish/v1/Chassis/chassis/Sensors/inlet_temp/Oem/SensorHistory",
          "SensorThreshold": {
            "@odata.id": "/redfish/v1/Chassis/chassis/Sensors/Oem/Ami/Threshold/inlet_temp"
          }
        }
      }
    }
  ]
}
```

### Session-Keepalive Exclusion

The authentication layer recognizes polling URLs. When a request targets any of
the following, the session `lastUpdated` timestamp is **not** reset:

- `/redfish/v1/Oem/Ami/Dashboard` (existing)
- `/redfish/v1/Chassis/{ChassisId}/Sensors` (pattern match)

URL normalization strips trailing slashes and query parameters before
comparison.

### Frontend Polling Integration

The AppHeader Sync toggle broadcasts a global `polling-toggled` event so
that any page can start or stop its own polling cycle.

The Sensors page:

1. On creation, checks `localStorage.getItem('pollingEnabled')` and starts a
   10-second polling interval if active. Registers a listener for the
   `polling-toggled` event.
2. On each poll, queries each cached sensor collection with
  `?$expand=.($levels=1)` and applies **incremental updates**: compares each
  sensor's reading, status, units, and thresholds against the cached value,
  and commits to the Vuex store only if something changed. It preserves the
  OEM `id` and `thresholdsId` values required for sensor history and threshold
  editing.
3. On destroy (page navigation), clears the polling interval and removes the
   event listener.

On a successful response, new sensors are added and missing sensors are
removed. If a sensor collection request fails, existing rows remain visible. A
`404` clears the cached collection URLs so the next poll rediscovers the
current Redfish tree.

The initial page load also uses the `pollSensorUpdates` path to load sensor data
efficiently in a single pass.

The polling flow (Sync toggle → event broadcast → interval → API call →
session exclusion) is identical to the existing Dashboard polling mechanism.

## Alternatives Considered

- **Poll individual Redfish sensor resources** — Rejected because it produces
  one HTTP request per sensor (100+ requests per poll interval).
- **Full array replacement on each poll** — Simpler frontend logic but causes
  the entire table to re-render, losing scroll/sort/selection state.

## Validation Plan

- **Polling interval confirmed** — Login, open the Sensors page, enable Sync,
  and open the browser Network tab. Verify one
  `GET /redfish/v1/Chassis/{chassisId}/Sensors?$expand=.($levels=1)` request per
  valid sensor-owning chassis every 10 seconds.
- **Session idle timeout** — Enable Sync on the Sensors page and leave the
  browser untouched for the session timeout duration. Confirm the user is
  logged out due to inactivity (verifying background polling does not reset
  session activity).
- **Threshold update reflected** — With polling active, modify a sensor's
  threshold value on the BMC. Verify the Sensors page displays the updated
  threshold within the next poll cycle without a manual page reload.
- **Sensor reading update reflected** — Change a sensor value on the BMC.
  Verify the value updates in the table smoothly without table jumping or scroll
  reset.
- **Sensor removal / addition reflected** — Add or remove a sensor on the BMC.
  Verify the table dynamically updates to include or prune the sensor.
- **Sensor history and threshold editing** — Verify opening a supported sensor
  history graph and changing a threshold continue to use the `id` and
  `thresholdsId` returned by the expanded response.
- **Transient resource failure** — Restart Entity Manager while polling is
  enabled. Verify existing rows remain visible on a failed request, and changed
  data appears on the next successful poll.
- **Polling stops on disable** — With polling active, click Sync to disable
  it. Verify in the Network tab that no further requests are made and the sensor
  values on the page stop updating.
