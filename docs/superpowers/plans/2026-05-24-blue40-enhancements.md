# Blue 40/SP4i Plugin Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix PM2.5/PM1 polling bugs, add fakegato-history Eve graphs for PM2.5, and expose the Blueair "Countdown to Clean Air" value as a custom HomeKit characteristic.

**Architecture:** Three independent PRs on separate branches off `main`. PR 1 fixes two sensor data propagation bugs on the existing `fix/blue40-sensor-telemetry` branch. PR 2 adds optional `fakegato-history` integration to log PM2.5 readings for Eve app graphs. PR 3 adds an optional custom HAP characteristic exposing the device's `aireta` (estimated minutes to clean air) value. All changes are opt-in via `DeviceConfig` booleans.

**Tech Stack:** TypeScript, Homebridge HAP API, fakegato-history npm package

**Spec:** `docs/superpowers/specs/2026-05-24-blue40-enhancements-design.md`

**No test suite exists in this project.** Verification is build-only (`npm run build`) plus manual deploy to media-server.

---

## PR 1: Bug Fixes + Telemetry (branch: `fix/blue40-sensor-telemetry`)

This branch already has the telemetry fallback commit. Two small fixes remain.

### Task 1: Fix pm2_5 case in AirPurifierAccessory

**Files:**
- Modify: `src/accessory/AirPurifierAccessory.ts:164`

The `updateCharacteristics` switch statement has `case 'pm25':` but `BlueAirDeviceSensorDataMap` maps to key `pm2_5`. The case never matches, so PM2.5 characteristic updates from polling never reach HomeKit after initial load.

- [ ] **Step 1: Fix the case label**

In `src/accessory/AirPurifierAccessory.ts`, change line 164:

```typescript
// Before:
case 'pm25':

// After:
case 'pm2_5':
```

- [ ] **Step 2: Build and verify**

Run: `npm run build`
Expected: Clean compilation, no errors.

### Task 2: Add pm1 case to updateCharacteristics

**Files:**
- Modify: `src/accessory/AirPurifierAccessory.ts:170` (after the pm10 case block)

PM1 data arrives from telemetry but has no case in the switch block. While HomeKit has no native PM1 characteristic, PM1 feeds into AQI calculation. Adding the case ensures `updateAirQuality` is set to `true` when PM1 changes.

- [ ] **Step 1: Add the pm1 case**

In `src/accessory/AirPurifierAccessory.ts`, add a new case after the `case 'pm10':` block (after line 170):

```typescript
case 'pm1':
  updateAirQuality = true;
  break;
```

- [ ] **Step 2: Build and verify**

Run: `npm run build`
Expected: Clean compilation, no errors.

### Task 3: Commit and prepare PR 1

- [ ] **Step 1: Commit the bug fixes**

```bash
git add src/accessory/AirPurifierAccessory.ts
git commit -m "fix: correct pm2_5 case label and add pm1 update path in AirPurifierAccessory

The switch statement in updateCharacteristics used 'pm25' but the sensor data
map produces key 'pm2_5', so PM2.5 polling updates never reached HomeKit.
Also add case for 'pm1' so AQI recalculates when PM1 values change."
```

- [ ] **Step 2: Push and create upstream PR**

```bash
git push origin fix/blue40-sensor-telemetry
```

Create PR targeting `kovapatrik/homebridge-blueair-purifier:main` with title: "fix: sensor data fallback and PM2.5 polling for Blue 40/SP4i" — body should summarize all commits on the branch (telemetry fallback + these fixes).

---

## PR 2: fakegato-history Eve Graphs (branch: `feat/fakegato-history`)

### Task 4: Create branch and add fakegato-history dependency

**Files:**
- Modify: `package.json`

- [ ] **Step 1: Create branch from main**

```bash
git checkout main
git checkout -b feat/fakegato-history
```

- [ ] **Step 2: Install fakegato-history**

```bash
npm install fakegato-history
```

This adds `fakegato-history` to `dependencies` in `package.json` and updates `package-lock.json`.

- [ ] **Step 3: Build to verify no breakage**

Run: `npm run build`
Expected: Clean compilation.

- [ ] **Step 4: Commit**

```bash
git add package.json package-lock.json
git commit -m "chore: add fakegato-history dependency"
```

### Task 5: Add `history` config toggle

**Files:**
- Modify: `src/platformUtils.ts`
- Modify: `config.schema.json`

- [ ] **Step 1: Add history to DeviceConfig type**

In `src/platformUtils.ts`, add `history` to the `DeviceConfig` type:

```typescript
export type DeviceConfig = {
  id: string;
  name: string;
  model: string;
  serialNumber: string;
  filterChangeLevel: number;
  led: boolean;
  airQualitySensor: boolean;
  co2Sensor: boolean;
  temperatureSensor: boolean;
  humiditySensor: boolean;
  germShield: boolean;
  nightMode: boolean;
  history: boolean;
};
```

- [ ] **Step 2: Add default value**

In `src/platformUtils.ts`, add to `defaultDeviceConfig`:

```typescript
export const defaultDeviceConfig: DeviceConfig = {
  // ... existing fields ...
  nightMode: false,
  history: false,
};
```

- [ ] **Step 3: Add to config.schema.json**

In `config.schema.json`, add to `devices.items.properties` (after the `nightMode` entry):

```json
"history": {
  "title": "Eve History",
  "description": "Enable logging of PM2.5 data for Eve app graphs. Requires the Eve app on iOS.",
  "type": "boolean"
}
```

And add `"devices[].history"` to the `layout` items array (after `"devices[].nightMode"`).

- [ ] **Step 4: Build and verify**

Run: `npm run build`
Expected: Clean compilation.

- [ ] **Step 5: Commit**

```bash
git add src/platformUtils.ts config.schema.json
git commit -m "feat: add history config toggle for Eve app graphs"
```

### Task 6: Integrate fakegato-history into AirPurifierAccessory

**Files:**
- Modify: `src/accessory/AirPurifierAccessory.ts`

This is the main integration. The fakegato-history service attaches to the HAP accessory and logs PM2.5 on every `stateUpdated` event.

- [ ] **Step 1: Add the history service**

In `src/accessory/AirPurifierAccessory.ts`, add the import at the top of the file (after existing imports):

```typescript
import fakegato from 'fakegato-history';
```

Add a private field to the class:

```typescript
private historyService?: Service;
```

At the end of the constructor (before the `this.device.on('stateUpdated', ...)` line), add the fakegato-history setup:

```typescript
if (this.configDev.history && this.configDev.airQualitySensor) {
  const FakeGatoHistoryService = fakegato(this.platform.api);
  this.historyService = new FakeGatoHistoryService('room', this.accessory, {
    storage: 'fs',
    log: this.platform.log,
  });
}
```

- [ ] **Step 2: Log PM2.5 entries on sensor data updates**

In `updateCharacteristics`, inside the `if (updateAirQuality)` block (after the existing `updateCharacteristic` call), add:

```typescript
if (updateAirQuality) {
  this.airQualityService?.updateCharacteristic(this.platform.Characteristic.AirQuality, this.getAirQuality());
  if (this.historyService) {
    const ppm = this.device.sensorData.pm2_5 || 0;
    (this.historyService as any).addEntry({
      time: Math.round(Date.now() / 1000),
      ppm: ppm,
    });
  }
}
```

Note: `fakegato-history` does not ship TypeScript types. The `as any` cast on `addEntry` is necessary. The `fakegato()` factory returns a class constructor, and the instance has `addEntry()` at runtime.

- [ ] **Step 3: Build and verify**

Run: `npm run build`
Expected: Clean compilation. If there's a type error on the `fakegato` import, add a type declaration file.

- [ ] **Step 4: If needed, add type declaration for fakegato-history**

If `npm run build` fails with "Cannot find module 'fakegato-history'", create `src/types/fakegato-history.d.ts`:

```typescript
declare module 'fakegato-history' {
  import { API, Service } from 'homebridge';
  function fakegato(api: API): new (type: string, accessory: any, options: any) => Service;
  export = fakegato;
}
```

And verify `tsconfig.json` includes the `src/types` directory (it should, since it compiles all of `src/`).

- [ ] **Step 5: Build again if declaration was needed**

Run: `npm run build`
Expected: Clean compilation.

- [ ] **Step 6: Commit**

```bash
git add src/accessory/AirPurifierAccessory.ts
# Include type declaration if it was created:
git add src/types/fakegato-history.d.ts 2>/dev/null
git commit -m "feat: integrate fakegato-history for Eve app PM2.5 graphs

When history is enabled in device config, PM2.5 readings are logged
via fakegato-history on each polling cycle. The Eve app displays
these as air quality line charts over time."
```

### Task 7: Push and create PR 2

- [ ] **Step 1: Push and create PR**

```bash
git push -u origin feat/fakegato-history
```

Create PR targeting `seanpmgallagher/homebridge-blueair-purifier:main` with title: "feat: add Eve app history graphs for PM2.5"

---

## PR 3: Countdown to Clean Air (branch: `feat/aireta-countdown`)

### Task 8: Create branch and add countdownToCleanAir config toggle

**Files:**
- Modify: `src/platformUtils.ts`
- Modify: `config.schema.json`

- [ ] **Step 1: Create branch from main**

```bash
git checkout main
git checkout -b feat/aireta-countdown
```

- [ ] **Step 2: Add countdownToCleanAir to DeviceConfig type**

In `src/platformUtils.ts`, add `countdownToCleanAir` to the `DeviceConfig` type:

```typescript
export type DeviceConfig = {
  id: string;
  name: string;
  model: string;
  serialNumber: string;
  filterChangeLevel: number;
  led: boolean;
  airQualitySensor: boolean;
  co2Sensor: boolean;
  temperatureSensor: boolean;
  humiditySensor: boolean;
  germShield: boolean;
  nightMode: boolean;
  countdownToCleanAir: boolean;
};
```

- [ ] **Step 3: Add default value**

In `src/platformUtils.ts`, add to `defaultDeviceConfig`:

```typescript
export const defaultDeviceConfig: DeviceConfig = {
  // ... existing fields ...
  nightMode: false,
  countdownToCleanAir: false,
};
```

- [ ] **Step 4: Add to config.schema.json**

In `config.schema.json`, add to `devices.items.properties` (after the `nightMode` entry):

```json
"countdownToCleanAir": {
  "title": "Countdown to Clean Air",
  "description": "Expose the device's estimated minutes to clean air as a custom characteristic. Visible in Eve and Controller for HomeKit.",
  "type": "boolean"
}
```

And add `"devices[].countdownToCleanAir"` to the `layout` items array (after `"devices[].nightMode"`).

- [ ] **Step 5: Build and verify**

Run: `npm run build`
Expected: Clean compilation.

- [ ] **Step 6: Commit**

```bash
git add src/platformUtils.ts config.schema.json
git commit -m "feat: add countdownToCleanAir config toggle"
```

### Task 9: Create custom characteristic and wire it up

**Files:**
- Modify: `src/accessory/AirPurifierAccessory.ts`

The `aireta` state value is already parsed by the generic state parser in `BlueAirAwsApi.getDeviceStatus()` — it reads all entries from the `states` array into `BlueAirDeviceState`, and the index signature `[key: string]: string | number | boolean | undefined` accommodates it. No API or device model changes needed.

- [ ] **Step 1: Define the custom characteristic class**

In `src/accessory/AirPurifierAccessory.ts`, add above the `AirPurifierAccessory` class:

```typescript
import { CharacteristicValue, PlatformAccessory, Service, Characteristic, Formats, Perms } from 'homebridge';

class CountdownToCleanAir extends Characteristic {
  static readonly UUID = '2c216842-f525-4640-b5fa-f1a0da82b8c6';

  constructor() {
    super('Countdown to Clean Air', CountdownToCleanAir.UUID, {
      format: Formats.UINT16,
      perms: [Perms.PAIRED_READ, Perms.NOTIFY],
      unit: 'minutes' as any,
      minValue: 0,
      maxValue: 1440,
    });
  }
}
```

Note: The UUID is a unique v4 UUID (not in the Eve E863F1xx range). `unit: 'minutes'` requires `as any` because HAP's `Units` enum doesn't include "minutes" but custom string units are accepted at runtime.

Update the existing import line to include `Formats` and `Perms`:

```typescript
// Before:
import { CharacteristicValue, PlatformAccessory, Service } from 'homebridge';

// After:
import { CharacteristicValue, Characteristic, Formats, Perms, PlatformAccessory, Service } from 'homebridge';
```

- [ ] **Step 2: Add the characteristic to AirQualitySensor service**

Add a private field to the class:

```typescript
private countdownCharacteristic?: Characteristic;
```

In the constructor, after the `airQualityService` setup block (after line 87's closing brace) and before the `temperatureService` block, add:

```typescript
if (this.configDev.countdownToCleanAir && this.airQualityService) {
  this.countdownCharacteristic = this.airQualityService.getCharacteristic(CountdownToCleanAir)
    || this.airQualityService.addCharacteristic(CountdownToCleanAir);
  this.countdownCharacteristic.onGet(this.getCountdownToCleanAir.bind(this));
}
```

- [ ] **Step 3: Add the getter method**

Add to the class:

```typescript
getCountdownToCleanAir(): CharacteristicValue {
  return (this.device.state.aireta as number) || 0;
}
```

- [ ] **Step 4: Add the update case in updateCharacteristics**

In the switch statement inside `updateCharacteristics`, add a case (e.g. after the `nightmode` case):

```typescript
case 'aireta':
  this.countdownCharacteristic?.updateCharacteristic(
    CountdownToCleanAir,
    this.getCountdownToCleanAir(),
  );
  break;
```

Wait — `updateCharacteristic` is on `Service`, not `Characteristic`. The `countdownCharacteristic` field is a `Characteristic`, not a `Service`. We need to use the service to update. Corrected approach:

```typescript
case 'aireta':
  if (this.countdownCharacteristic) {
    this.countdownCharacteristic.updateValue(this.getCountdownToCleanAir());
  }
  break;
```

- [ ] **Step 5: Build and verify**

Run: `npm run build`
Expected: Clean compilation.

- [ ] **Step 6: Commit**

```bash
git add src/accessory/AirPurifierAccessory.ts
git commit -m "feat: expose Countdown to Clean Air as custom HomeKit characteristic

Adds a custom HAP characteristic (UUID 2c216842-...) that shows the device's
estimated minutes until clean air. Attached to the AirQualitySensor service
when countdownToCleanAir is enabled. Visible in Eve and Controller for HomeKit;
silently ignored by Apple Home."
```

### Task 10: Push and create PR 3

- [ ] **Step 1: Push and create PR**

```bash
git push -u origin feat/aireta-countdown
```

Create PR targeting `seanpmgallagher/homebridge-blueair-purifier:main` with title: "feat: expose Countdown to Clean Air custom characteristic"

---

## Deployment (after all PRs are merged to main on fork)

### Task 11: Deploy to media-server

- [ ] **Step 1: Merge all branches locally**

```bash
git checkout main
git merge fix/blue40-sensor-telemetry
git merge feat/fakegato-history
git merge feat/aireta-countdown
```

- [ ] **Step 2: Build and deploy**

```bash
npm run build
scp -r . media-server:/Users/seangallagher/.nvm/versions/node/v24.11.0/lib/node_modules/homebridge-blueair-purifier/
```

- [ ] **Step 3: Restart child bridge**

Restart the Blueair child bridge from Homebridge UI at http://media-server:8581.

- [ ] **Step 4: Verify in HomeKit/Eve**

- PM2.5 updates across multiple polling cycles (not just initial load)
- AirQuality enum changes when PM values change
- Eve app shows PM2.5 history graph (if `history: true`)
- Eve app shows "Countdown to Clean Air" custom characteristic (if `countdownToCleanAir: true`)
