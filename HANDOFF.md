# Session Handoff — Blue 40/SP4i Plugin Enhancements

## Start here

Read `docs/superpowers/specs/2026-05-24-blue40-enhancements-design.md` for the full design spec. This handoff provides the current state and what to do next.

## What this repo is

A fork of [kovapatrik/homebridge-blueair-purifier](https://github.com/kovapatrik/homebridge-blueair-purifier) at [seanpmgallagher/homebridge-blueair-purifier](https://github.com/seanpmgallagher/homebridge-blueair-purifier). Homebridge plugin for Blueair air purifiers, written in TypeScript.

The user has two **Blueair Blue Signature SP4i** units (hw: `l_blue40`) running via Homebridge on a macOS media-server at `ssh media-server` (10.0.0.43). Homebridge is installed natively (LaunchDaemon) using nvm Node v24.11.0. Plugin is globally installed at `/Users/seangallagher/.nvm/versions/node/v24.11.0/lib/node_modules/homebridge-blueair-purifier`.

## What's been done

### Branch: `fix/blue40-sensor-telemetry` (pushed to origin)

Two commits:
1. **Telemetry fallback** — The Blue 40/SP4i returns empty `sensordata[]` from the `/initial` REST endpoint. Added `getDeviceTelemetry()` method that calls `GET /{accountUuid}/r/telemetry/5m/historical` to fetch 5-minute aggregated PM readings when sensordata is empty. Also fixed `pm25` → `pm2_5` key bug in `BlueAirDevice.ts` line 173. **Tested and working** — Air Quality shows "Excellent" in HomeKit.
2. **Design spec** committed to `docs/superpowers/specs/`.

### What still needs to happen on this branch before upstream PR

- Fix `case 'pm25':` → `case 'pm2_5':` in `src/accessory/AirPurifierAccessory.ts` line 164 (PM2.5 polling updates never reach HomeKit — same typo pattern)
- Add `case 'pm1': updateAirQuality = true; break;` in the same switch block (PM1 data feeds AQI calc but has no update path)
- Remove temporary info-level log line if any remain (check `this.logger.info` calls in `dist/api/BlueAirAwsApi.js` on media-server — the source files are clean, only the deployed compiled JS may have temp debug lines)
- Submit PR to upstream `kovapatrik/homebridge-blueair-purifier`

## Three PRs planned (approach B — separate branches)

| PR | Branch | Status | Target |
|----|--------|--------|--------|
| 1. Bug fixes + telemetry | `fix/blue40-sensor-telemetry` | In progress — needs pm2_5 accessory fix + pm1 case | Upstream (kovapatrik) |
| 2. fakegato-history | `feat/fakegato-history` off `main` | Not started | Fork (maybe upstream later) |
| 3. aireta countdown | `feat/aireta-countdown` off `main` | Not started | Fork (maybe upstream later) |

## Next step

Invoke the `superpowers:writing-plans` skill (or manually create an implementation plan) based on the design spec, then execute. Start with PR 1 since it's partially done.

## Deployment to media-server

After building (`npm run build`), deploy to media-server:
```bash
scp -r . media-server:/Users/seangallagher/.nvm/versions/node/v24.11.0/lib/node_modules/homebridge-blueair-purifier/
```
Then restart the Blueair child bridge from the Homebridge UI at http://media-server:8581.

Note: `npm install -g .` creates dangling symlinks from git repos — use `scp -r` to copy the built output directly. The plugin directory must contain `package.json` or Homebridge refuses to load it.

## Key technical context

- **Blue 40/SP4i has NO temperature or humidity sensors** — only PM1, PM2.5, PM10, and RSSI
- The `/initial` endpoint's `sensordata[]` is **always empty** for this model (confirmed by dahlb/blueair_api test fixtures). Sensor data comes via MQTT or the historical telemetry REST endpoint we're using.
- The `aireta` (Countdown to Clean Air) value IS returned in the `states` array from `/initial` — no telemetry call needed for it
- `fakegato-history` npm package is the standard for Eve app graphs in the Homebridge ecosystem
- The existing plugin already exposes: LED as Lightbulb+Brightness, Night Mode as Switch, Filter Maintenance with configurable threshold, AirQuality sensor with PM2.5/PM10/VOC
- The `config.schema.json` file controls the Homebridge UI config form — update it for new config toggles
