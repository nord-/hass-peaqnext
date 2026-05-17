# Changelog

## v0.6.3

### Fix: Nordpool entity discovery on newer Home Assistant

`homeassistant.helpers.template` was restructured into a package in recent Home Assistant versions and `integration_entities` is no longer a module-level function (it now lives as a Jinja method on `ConfigEntryExtension`). The old call raised `AttributeError`, left the spotprice entity unset, and crashed the Hub when `async_track_state_change_event` received `[None]`.

- Look up entities via `homeassistant.helpers.entity.entity_sources` directly — the same fallback logic the original helper used internally.
- Guard the Hub against a missing spotprice entity so a misconfigured environment logs an error instead of crashing setup.

Fixes elden1337/hass-peaqnext#34.

## v0.6.2

### Fix: Incorrect price mapping for sub-hourly Nordpool data

Nordpool now provides 96 prices per day (15-minute intervals) instead of 24 (hourly). The calculation algorithm indexed prices by hour (0-23), causing it to use completely wrong prices — e.g. the 05:00 price for the 20:00 time slot.

- At 19:00, the sensor would pick prices from indices 19-23 in the 96-element array, which correspond to ~04:45–05:45 prices instead of the actual 19:00–23:00 prices.
- This led to suboptimal recommendations — suggesting expensive hours while much cheaper slots were available.
- Sub-hourly prices are now automatically aggregated to hourly averages when loaded from the spot price source. Handles 24, 23, and 25-hour days (DST).

## v0.6.1

### Bug fixes

- **Fix Hub instance isolation** — class-level `sensors_dict` and `sensors` were shared across all Hub instances, now moved to `__init__` (#6, #8)
- **Fix singleton Hub pattern** — each config entry now gets its own Hub instance (#6)
- **Fix `async_cancel_override`** — use `object.__getattribute__` to bypass custom override resolution and read original values (#10)
- **Fix interval alignment** — aligned interval indexing to hourly granularity to match actual price data (#14, #16)
- **Fix `_blocked_interval` midnight wrap-around** — intervals spanning past midnight are now correctly checked against non_hours (#20)
- **Fix ZeroDivisionError in PeriodModel** — guard against zero `sum_consumption_pattern` (#22)
- **Fix `async_remove_config_entry_device`** — return `True` instead of empty body (#12)

### Code quality

- Narrow exception handling throughout (`except Exception` → specific types)
- Replace `print()` with `_LOGGER` in segments.py
- Replace bare `except:` with typed exceptions in override model
- Cache `datetime.now()` in `_make_hours_display`
- Fix attribute typo `_all_seqeuences` → `_all_sequences`
- Use `time.monotonic()` instead of `time.time()` in Hub
- Chain `ValueError` with `from e` for proper traceback

### Improvements

- Handle DST transitions (23/25-hour days) in `_create_prices_dict`
- Increase `SCAN_INTERVAL` from 4s to 60s to reduce unnecessary polling
- Replace `.format()` with f-strings in services.py
- Add tests for blocked interval spanning, DST price arrays, and tomorrow non_hours filtering
