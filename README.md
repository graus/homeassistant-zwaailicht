<p align="center">
  <img src="custom_components/zwaailicht/brand/icon.png" alt="Zwaailicht" width="80" />
</p>

# Zwaailicht P2000 — Home Assistant Integration

A HACS-compatible Home Assistant custom integration that surfaces live P2000 emergency alerts from [zwaailicht.nl](https://zwaailicht.nl) as sensor entities. Set a radius around your home and get notified about nearby incidents on your dashboards, NSPanel displays, and in automations.

## Installation

### HACS (recommended)

1. Open HACS in Home Assistant
2. Click the three dots menu → **Custom repositories**
3. Add `https://github.com/graus/homeassistant-zwaailicht` as an **Integration**
4. Search for "Zwaailicht P2000" and install
5. Restart Home Assistant

### Manual

Copy the `custom_components/zwaailicht` folder into your Home Assistant `config/custom_components/` directory and restart.

## Configuration

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for "Zwaailicht P2000"
3. Set a **radius** (km) around your HA home location — only alerts within this radius are shown
4. Toggle **significant only** — filters out routine/low-priority alerts server-side (default: on)
5. Toggle **pieken** on/off — notable multi-unit incidents nationwide (default: on)
6. Optionally adjust the update interval (default: 60s, minimum: 30s)

That's it. The integration polls the national feed and filters by distance from your home. All settings can be changed later via **Settings → Devices & Services → Zwaailicht P2000 → Configure**.

## Sensors

All sensors are grouped under a single **Zwaailicht P2000** device. Each can be individually enabled or disabled.

### Laatste melding (`sensor.zwaailicht_laatste_melding`)

The most recent P2000 alert within your configured radius.

- **State**: alert title (e.g. "🚑 A1 Spoed — Brouwersgracht, Amsterdam"), or "Geen meldingen"
- **Icon**: dynamic per dienst (fire truck, ambulance, police badge, lifebuoy)
- **Attributes**: `dienst`, `stad`, `timestamp`, `link`, `prioriteit_code`, `prioriteit`, `locatie`, `distance_km`

### Aantal meldingen (`sensor.zwaailicht_aantal_meldingen`)

Number of recent alerts within your radius (from the last feed poll).

- **State**: integer (e.g. `3`)
- **State class**: measurement (enables history graphs)

### Laatste piek (`sensor.zwaailicht_laatste_piek`)

The most recent notable multi-unit incident nationwide. Created when pieken is enabled. Pieken are not filtered by radius — they are curated national incidents that may span multiple locations.

- **State**: piek title (e.g. "Woningbrand Gezellelaan Roosendaal"), or "Geen pieken"
- **Attributes**: `stad`, `timestamp`, `link`, `summary`, `distance_km` (when coordinates available)

## Events & Triggers

The integration fires Home Assistant events for each genuinely new entry. These only fire once per alert (not on every poll), making them ideal for automations.

### `zwaailicht_new_alert`

Fires when a new P2000 melding appears within your radius.

**Event data:**

| Field | Type | Example | Always present |
|---|---|---|---|
| `id` | string | `https://zwaailicht.nl/amsterdam/medisch/2026-04-06/4712c0` | yes |
| `title` | string | `🚑 A1 Spoed — Brouwersgracht, Amsterdam` | yes |
| `timestamp` | string | `2026-04-06T13:01:29Z` | yes |
| `link` | string | `https://zwaailicht.nl/amsterdam/medisch/...` | yes |
| `dienst` | string | `ambulance`, `brandweer`, `politie`, `knrm` | yes |
| `stad` | string | `amsterdam` | yes |
| `latitude` | float | `52.3781` | yes |
| `longitude` | float | `4.8870` | yes |
| `distance_km` | float | `2.3` | yes |
| `prioriteit_code` | string | `A1`, `A2`, `B1`, `P1`, `P2` | when parseable |
| `prioriteit` | string | `Spoed`, `Urgent`, `Gepland vervoer` | when parseable |
| `locatie` | string | `Brouwersgracht` | when parseable |
| `summary` | string | `Ambulance melding. Prioriteit: Spoed. Type: Medisch.` | when available |
| `eenheid` | string | `BAD-01` | when in summary |
| `type` | string | `Medisch`, `Brand` | when in summary |

### `zwaailicht_new_piek`

Fires when a new piek (notable multi-unit incident) appears nationwide. Not filtered by radius.

- `dienst` is always `piek`
- `summary` contains a longer narrative description of the incident
- `prioriteit_code`, `prioriteit`, `locatie`, `eenheid`, `type` are not present
- `latitude`, `longitude`, `distance_km` may be absent (pieken may not have exact coordinates)

## Automation Examples

### Flash NSPanel on nearby brandweer alert

```yaml
automation:
  - alias: "P2000 Brandweer Alert"
    trigger:
      - platform: event
        event_type: zwaailicht_new_alert
        event_data:
          dienst: brandweer
    action:
      - service: notify.nspanel
        data:
          message: "🔥 {{ trigger.event.data.title }} ({{ trigger.event.data.distance_km }}km)"
```

### Push notification for nearby ambulance spoed

```yaml
automation:
  - alias: "Ambulance Spoed Nearby"
    trigger:
      - platform: event
        event_type: zwaailicht_new_alert
        event_data:
          dienst: ambulance
    condition:
      - condition: template
        value_template: "{{ trigger.event.data.prioriteit_code == 'A1' }}"
    action:
      - service: notify.mobile_app_phone
        data:
          title: "🚑 Ambulance A1"
          message: "{{ trigger.event.data.locatie }}, {{ trigger.event.data.stad }} ({{ trigger.event.data.distance_km }}km)"
```

### Notify on pieken

```yaml
automation:
  - alias: "Piek Alert"
    trigger:
      - platform: event
        event_type: zwaailicht_new_piek
    action:
      - service: notify.mobile_app_phone
        data:
          title: "P2000 Piek — {{ trigger.event.data.stad }}"
          message: "{{ trigger.event.data.title }}"
```

### Only react to very close alerts (< 2km)

```yaml
automation:
  - alias: "Very Close Alert"
    trigger:
      - platform: event
        event_type: zwaailicht_new_alert
    condition:
      - condition: template
        value_template: "{{ trigger.event.data.distance_km < 2 }}"
    action:
      - service: light.turn_on
        target:
          entity_id: light.warning_lamp
        data:
          color_name: red
          flash: long
```

### Filter by priority — only spoed/P1

```yaml
    condition:
      - condition: template
        value_template: "{{ trigger.event.data.prioriteit_code in ['A1', 'P1'] }}"
```

### Sensor state change trigger

```yaml
automation:
  - alias: "New Nearby Alert"
    trigger:
      - platform: state
        entity_id: sensor.zwaailicht_laatste_melding
    action:
      - service: notify.mobile_app_phone
        data:
          message: "{{ states('sensor.zwaailicht_laatste_melding') }}"
```

## Links

- [zwaailicht.nu](https://zwaailicht.nu) — P2000 alerts for the Netherlands
