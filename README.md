# Anker Solix Tibber Blueprint for Home Assistant

[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://github.com/hacs/integration)
[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FAdcompro%2Fha-anker-solix-tibber-blueprint%2Fmain%2Fblueprints%2Fautomation%2Ftibber_charge_discharge.yaml)

Smart charge/discharge automation for an **Anker Solix Solarbank** controlled by **Tibber** (or compatible) dynamic electricity prices, with automatic fallback to the Anker app's "smart" mode.

## What it does

- **Charges** the battery from the grid during the **cheapest** part of the day
  (default: 1.5 hours before through 1.5 hours after the cheapest Tibber slot).
- **Discharges** to the grid during the **most expensive** part of the day
  (default: 1.0 hour before through 1.0 hour after the most expensive Tibber slot).
- **Falls back to `smart` mode** the rest of the time — the Anker app keeps doing
  its normal balancing.
- **Overlap-safe**: if the charge and discharge windows would overlap, the
  blueprint does nothing and leaves the Solarbank in `smart`.
- **Idempotent**: runs every minute but only issues API calls when something
  actually needs to change.

## Requirements

- [Home Assistant](https://www.home-assistant.io/) (tested on 2026.5.x)
- [Anker Solix custom integration](https://github.com/thomluther/ha-anker-solix)
  installed via HACS
- A dynamic-price sensor whose state attribute contains a list of
  `{ start: <ISO timestamp>, price: <float> }` objects.
  - Out-of-the-box compatible with
    [Tibber Smart](https://github.com/JonkerJ/tibber_smart): use
    `sensor.tibber_smart_prices_today` and attribute `prices_today`.
  - Other integrations work if they expose the same shape — set the attribute
    name in the blueprint inputs.

## Installation

### 1. Import the blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FAdcompro%2Fha-anker-solix-tibber-blueprint%2Fmain%2Fblueprints%2Fautomation%2Ftibber_charge_discharge.yaml)

Or manually: **Settings → Automations → Blueprints → Import Blueprint** and paste:

```
https://raw.githubusercontent.com/Adcompro/ha-anker-solix-tibber-blueprint/main/blueprints/automation/tibber_charge_discharge.yaml
```

### 2. Create an automation from it

**Settings → Automations → Create Automation → Use Blueprint → Anker Solix - Tibber slim laden en terugleveren**

Fill in:

| Input | Example |
| --- | --- |
| Prices sensor | `sensor.tibber_smart_prices_today` |
| Prices attribute | `prices_today` |
| Usage mode select | `select.solarbank_3_e2700_pro_usage_mode_2` |
| Smart mode option | `smart` |
| Manual mode option | `manual` |
| Output preset number | `number.solarbank_3_e2700_pro_system_output_preset_2` |
| Backup option switch | `switch.solarbank_3_e2700_pro_backup_option_2` |
| Grid export switch | `switch.solarbank_3_e2700_pro_allow_grid_export` |
| Hours before cheapest slot | `1.5` |
| Hours after cheapest slot | `1.5` |
| Hours before most-expensive slot | `1.0` |
| Hours after most-expensive slot | `1.0` |
| Max discharge power | `2400` |
| Enable boolean (optional) | leave empty, or your own `input_boolean.*` toggle |

### 3. Set the Anker app to `smart` mode

The Solarbank will only be temporarily flipped to `manual` during the charge or
discharge windows and immediately reset to `smart` afterwards. So keep the
Anker app on **smart** — Home Assistant takes over only when needed.

## How it works

Every minute (and on price updates) the automation:

1. Reads `prices_today` from the configured sensor.
2. Finds the slot with the lowest and the slot with the highest price.
3. Computes the charge window `[cheapest − before, cheapest + after]` and the
   discharge window `[expensive − before, expensive + after]`.
4. Decides the target state — `charge`, `discharge`, or `smart`:
   - `charge` if now is in the charge window and windows don't overlap.
   - `discharge` if now is in the discharge window and windows don't overlap.
   - Otherwise `smart`.
5. Compares against the current Solarbank state (mode + switches + output) and
   only issues service calls if a change is actually needed.

| Target | `usage_mode` | `backup_option` | `allow_grid_export` | `output_preset` |
| --- | --- | --- | --- | --- |
| `charge`    | `manual` | `on`  | `off` | `0 W` |
| `discharge` | `manual` | `off` | `on`  | configurable max W |
| `smart`     | `smart`  | `off` | `off` | (unchanged) |

## Alternative: stand-alone package

If you prefer **visible state** entities (charge window active, discharge window
active, windows overlap) and **separate** start/stop automations instead of one
blueprint-driven automation, copy `battery_optimizer_package.yaml` from this
repository into your Home Assistant `packages/` directory.

That version exposes:

- `sensor.solix_goedkoopste_slot` / `sensor.solix_duurste_slot`
- `binary_sensor.solix_laadvenster_actief`
- `binary_sensor.solix_ontlaadvenster_actief`
- `binary_sensor.solix_vensters_overlappen`

Edit the entity_ids inside the file to match your Solarbank.

## Caveats

- Tibber's `prices_today` attribute is in **15-minute slots**. "Cheapest slot"
  means the cheapest 15-minute interval — the charge window then extends
  symmetrically around the START of that interval.
- Edge case: when the cheapest/most-expensive slot is near midnight, part of
  the window may fall on the previous/next day and won't activate as expected.
  Acceptable for the common case but worth knowing.
- The blueprint sets the Solarbank to `manual` mode during windows. Make sure
  you're OK with that — the Anker app will show "manual" briefly.

## License

MIT — see [LICENSE](LICENSE).

## Contributing

PRs welcome. Especially helpful: adapters for other price sources (Nordpool,
EnergyZero, ENTSO-e, Frank Energie) and tests against more Solarbank models.
