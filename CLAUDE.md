# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**peaqnext** is a Home Assistant custom component that suggests optimal times to run appliances based on electricity spot prices (Nordpool or EnergiDataService). It calculates the cheapest time window considering consumption patterns, duration, and hour constraints.

The project is in **maintenance mode** — only bug fixes are accepted, not feature PRs.

## Commands

### Lint
```bash
flake8 custom_components/peaqnext --count --select=E9,F63,F7,F82 --show-source --statistics
```

### Format
```bash
black custom_components/peaqnext
isort custom_components/peaqnext
autoflake --in-place --remove-all-unused-imports -r custom_components/peaqnext
```

### Test
```bash
pytest custom_components/peaqnext/test/
pytest custom_components/peaqnext/test/test_nextsensor.py  # single file
pytest custom_components/peaqnext/test/test_nextsensor.py::test_function_name -v  # single test
```

Tests use async pytest and mock datetime via `DTModel.set_hour()/set_date()/set_minute()`.

## Architecture

All code lives under `custom_components/peaqnext/`.

### Three-layer design

1. **HA Platform Layer** — `__init__.py`, `sensor.py`, `config_flow.py`, `services.py`
   - `PeaqNextSensor` (SensorEntity) in `sensors/next_sensor.py`
   - Two services: `override_sensor_data` and `cancel_override`

2. **Model Layer** — `service/models/`
   - `NextSensor` (dataclass) — main model, extends `NextSensorData`. Uses custom `__getattribute__` to resolve overrides transparently.
   - `PeriodModel` — time window with price, comparer for sorting
   - `DTModel` — datetime wrapper with mock support for testing
   - `ConsumptionType` enum — Flat, PeakIn, PeakOut, MidPeak, PeakInOut, Custom

3. **Service Layer** — `service/`
   - `Hub` — coordinator managing multiple sensors and spot price polling
   - `SpotPriceFactory` — factory creating Nordpool or EnergiDataService adapter (both implement `ISpotPrice`)
   - `hours.py` — interval sorting, cheapest window selection (15-min granularity)
   - `segments.py` — consumption pattern distribution across duration

### Spot price integration

Uses factory pattern (`spotprice/spotprice_factory.py`) with `ISpotPrice` interface. Adapters poll HA state machine for price entities and handle ore/cent ↔ kr/euro conversion. Prices are tuples of `(today_prices, tomorrow_prices)`.

### Key algorithm

The algorithm in `hours.py` works with **hourly** price arrays (24/23/25 entries per day). Sub-hourly prices (e.g. 96 entries for 15-min intervals from Nordpool) are normalized to hourly averages in `ISpotPriceDTO._normalize_to_hourly()` before reaching the algorithm. Consumption pattern is distributed across the duration, then a weighted sum gives total cost per candidate window. Windows are sorted by `comparer` (normalized price), then `dt_start` as tiebreaker. The `comparer` in `PeriodModel` intentionally rounds to 1 decimal (SEK) / 2 decimals (EUR) — this is **by design** so that near-equal prices favor an earlier start time rather than delaying for marginal savings. Do not increase the precision. Constraints (non_hours_start, non_hours_end) exclude invalid windows.

## Releasing

HACS requires a zip asset attached to each GitHub release (`hacs.json` has `zip_release: true`).

```bash
# 1. Bump version in manifest.json
# 2. Commit and push
# 3. Create release with zip:
python -c "
import zipfile, glob, os
base = 'custom_components/peaqnext'
with zipfile.ZipFile('peaqnext.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    for f in glob.glob(base + '/**', recursive=True):
        if os.path.isfile(f) and '__pycache__' not in f and os.sep + 'test' + os.sep not in f:
            zf.write(f, os.path.relpath(f, base).replace(os.sep, '/'))
"
gh release create vX.Y.Z --title "vX.Y.Z" --notes "..."
gh release upload vX.Y.Z peaqnext.zip
rm peaqnext.zip
```

- Zip must use **forward slashes** in paths (not Windows backslashes)
- Zip must **exclude** test files and `__pycache__`
- Files must be at the **root** of the zip (no `custom_components/peaqnext/` prefix)
