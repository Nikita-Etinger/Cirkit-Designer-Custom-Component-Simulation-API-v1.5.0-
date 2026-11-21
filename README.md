# Cirkit Designer Custom Component Simulation API (v1.5.0)

Unofficial community reference for authoring custom simulated components in **Cirkit Designer** (`@cirkit/simulation/logic` and `@cirkit/simulation/ui`), reconstructed from public TypeScript declaration contracts (`COMPONENT_API_VERSION = "1.5.0"`).

---

## Architecture Overview

Every custom simulated component consists of 5 files:

| File | Purpose | Runtime / Module |
| :--- | :--- | :--- |
| **Simulation Logic (`*.ts`)** | Electrical SPICE-like netlist & behavioral logic | `@cirkit/simulation/logic` |
| **UI Script (`*.ts`)** | Interactive overlay controller & state sync | `@cirkit/simulation/ui` |
| **UI HTML (`*.html`)** | Interactive visual layer rendered over static SVG/PNG | Angular-like template bindings |
| **Runtime State (`*.json`)** | Reactive state variables, UI-to-Logic inputs, Logic-to-UI outputs | JSON schema |
| **Component Properties (`*.json`)** | Static pre-simulation parameters (resistance, capacity, etc.) | JSON schema |

---

## 1. Simulation Logic API (`@cirkit/simulation/logic`)

### Base Class

```typescript
import { AbstractSimulationComponentLogic } from '@cirkit/simulation/logic';

export class SimulationComponentLogic extends AbstractSimulationComponentLogic {
  public init(): void {
    // Called once when simulation starts
  }
}
```

### Instance Members (`this` & `this.simulation`)

* `this.componentInstanceID: string` — Unique component instance identifier on the canvas.
* `this.getAPIVersion(): string` — Returns `"1.5.0"`.
* `this.simulation.api` — Hardware & electrical primitives (`pin`, `connection`, `timer`, `i2c`, `spi`, `uart`, `audio`, `network`, `radio`, `infrared`).
* `this.simulation.runtimeState` — Shared reactive variables (`SimulationStateManager`).
* `this.simulation.componentProperties` — Static component configuration (`ComponentPropertiesManager`).
* `this.simulation.inputs` — Event/audio/video/packet receivers from UI (`ComponentInputsManager`).
* `this.simulation.outputs` — Event/audio/packet emitters to UI (`ComponentOutputsManager`).
* `this.simulation.componentAssets` — Read-only attached binary files (`ComponentAssetsManager`).
* `this.simulation.hideStaticPicture(): void` — Hides the static PNG/SVG artwork while simulation is running.
* `this.simulation.log(message: string, level?: LogLevelEnum): void` — Logs to simulator console (`DEBUG`, `INFO`, `WARN`, `ERROR`).

---

### 1.1. Pin Management (`this.simulation.api.pin` — `PinUtils`)

#### Enums
```typescript
enum PinModeEnum {
  Input = 0,
  Output = 1,
  InputPullup = 2,
  InputPulldown = 3,
  Analog = 4
}

enum DigitalVoltageLevelEnum {
  LOW = 0,
  HIGH = 1
}

enum EdgeEnum {
  Rising = 0,
  Falling = 1,
  Both = 2
}
```

#### Pin Creation & Reconfiguration
* `createInputPin(pinName: string): IPin`
* `createInputPinWithPullUp(pinName: string, resistor?: number, pullVoltage?: number): IPin` *(defaults: `10000` Ω, `5` V)*
* `createInputPinWithPullDown(pinName: string, resistor?: number): IPin` *(default: `10000` Ω)*
* `createDigitalOutputPin(pinName: string, highVoltage?: number, value?: DigitalVoltageLevelEnum): IPin` *(defaults: `5` V, `LOW`)*
* `createAnalogOutputPin(pinName: string, voltage?: number): IPin` *(default: `0` V)*
* `createCustomPin(pinConfig: IPinConfig): IPin`
* `updatePinToInput(pin: IPin): void`
* `updatePinToInputWithPullUp(pin: IPin, resistor?: number, pullVoltage?: number): void`
* `updatePinToInputWithPullDown(pin: IPin, resistor?: number): void`
* `updatePinToDigitalOutput(pin: IPin, highVoltage?: number, value?: DigitalVoltageLevelEnum): void`
* `updatePinToAnalogOutput(pin: IPin, voltage?: number): void`

#### Reading, Writing & Interrupt Watches
* `readAnalog(pin: IPin): number` — Reads resolved node voltage in Volts.
* `readDigital(pin: IPin, thresholdVoltage?: number): DigitalVoltageLevelEnum` *(default threshold: `2.5` V)*.
* `writeAnalog(pin: IPin, voltage: number): void`
* `writeDigital(pin: IPin, value: DigitalVoltageLevelEnum): void`
* `addDigitalPinWatch(pin: IPin, edge: EdgeEnum, callback: (pin: IPin, voltage: number) => void): boolean`
* `removeDigitalPinWatch(pin: IPin): boolean`
* `addAnalogPinWatch(pin: IPin, deltaThreshold: number, callback: (pin: IPin, voltage: number) => void): boolean`
* `removeAnalogPinWatch(pin: IPin): boolean`

#### `IPin` Properties
* `pin.name: string`
* `pin.voltage: number`
* `pin.referenceVoltage: number`
* `pin.mode: PinModeEnum`
* `pin.getIsPinInUse(): boolean` — Returns `true` if a wire is attached to this pin in the schematic.

> **Avoiding Net Solver Conflicts:** Connecting two ideal voltage drivers (`DigitalOutputPin`, `AnalogOutputPin`, or `addVoltageSource`) with different voltages to the same net throws:
> `Uncaught Error: Conflict detected on net(s): multiple driving pins with different voltages.`
> Always decouple internal voltage sources using an internal virtual pin (`createInputPin('__INT')`) bridged to the external pin via `addResistor(...)`.

---

### 1.2. Internal Circuit & SPICE Primitives (`this.simulation.api.connection` — `ConnectionUtils`)

Build dynamic internal equivalent circuits between pins:

#### Wires & Resistors
* `addWire(pin1: IPin, pin2: IPin): IWireConnection`
* `removeWire(wire: IWireConnection): boolean`
* `addResistor(pin1: IPin, pin2: IPin, resistance: number): IResistorConnection`
* `updateResistor(resistor: IResistorConnection, newResistance: number): boolean`
* `removeResistor(resistor: IResistorConnection): boolean`

#### Ideal DC Voltage Source
* `addVoltageSource(positivePin: IPin, referencePin: IPin, options?: { voltage?: number }): IVoltageSourceHandle | null`
* `removeVoltageSource(handle: IVoltageSourceHandle): boolean`
* **`IVoltageSourceHandle`:** `getVoltage(): number`, `setVoltage(v: number): void`, `onChange(cb: (voltage: number) => void): void`

#### Diodes & LEDs
* `addDiode(pinAnode: IPin, pinCathode: IPin, options?: { Vf?: number; RsOn?: number }): IDiodeHandle | null`
* `removeDiode(handle: IDiodeHandle): boolean`
* **`IDiodeHandle`:** `isConducting(): boolean`, `onConductingChange(cb: (conducting: boolean) => void): void`
* `addLED(pinAnode: IPin, pinCathode: IPin, forwardVoltageThreshold?: number, onResistance?: number): ILEDConnection | null`
* `removeLED(led: ILEDConnection): boolean`
* **`ILEDConnection`:** `isEmittingLight(): boolean`, `onStateChange(cb: (isEmittingLight: boolean) => void): void`

#### Transistors (BJT & MOSFET)
* `addBJT(collector: IPin, base: IPin, emitter: IPin, options?: IBjtModelOptions): IBjtHandle | null`
  * `options`: `{ type?: BjtTypeEnum.NPN | BjtTypeEnum.PNP; betaF?: number; betaR?: number; VbeOn?: number; VceSat?: number }`
  * **`IBjtHandle`:** `getRegion(): BjtRegionEnum`, `isConducting(): boolean`, `onRegionChange(cb): void`
* `removeBJT(handle: IBjtHandle): boolean`
* `addMOSFET(drain: IPin, gate: IPin, source: IPin, options?: IMosfetModelOptions): IMosfetHandle | null`
  * `options`: `{ type?: MosfetTypeEnum.NMOS | MosfetTypeEnum.PMOS; Vth?: number; RdsOn?: number }`
  * **`IMosfetHandle`:** `getRegion(): MosfetRegionEnum`, `isConducting(): boolean`, `onRegionChange(cb): void`
* `removeMOSFET(handle: IMosfetHandle): boolean`

#### Brushed DC Motor
* `addDCMotor(pinA: IPin, pinB: IPin, options: { rWindings: number; kvRpmPerVolt: number; J?: number; friction?: number }): IDCMotorHandle | null`
* `removeDCMotor(handle: IDCMotorHandle): boolean`
* **`IDCMotorHandle`:** `getSpeed(): number` (rad/s), `getSpeedRpm(): number`, `setLoadTorque(fn: () => number): void`, `onSpeedChange(cb): void`

---

### 1.3. Simulation Timers (`this.simulation.api.timer` — `TimerUtils`)

Use simulator-managed timers instead of `setInterval`/`setTimeout` so execution stays synchronized with simulated time and pause/resume lifecycle:

* `createTimer(callback: () => void): ITimer`
* `startTimer(timer: ITimer, intervalMicros: number, repeat: boolean): void` *(interval in **microseconds**, e.g. `50000` = 50 ms)*
* `stopTimer(timer: ITimer): void`
* `getCurrentTimeNanos(): bigint` — Current simulation timestamp in nanoseconds.

---

### 1.4. Communication Buses (I2C, SPI, UART)

#### I2C (`this.simulation.api.i2c` — `I2CUtils`)
* `createI2CSlave(address: number, sclPin: IPin, sdaPin: IPin, callbacks: { connect(address: number, isWriteMode: boolean): boolean; writeByte(data: number): boolean; readByte(): number; disconnect?(): void }): I2CSlaveRegistrationHandle`
* `createTimedI2CMaster(sclPin: IPin, sdaPin: IPin, frequencyHz: number): ITimedI2CMaster`

#### SPI (`this.simulation.api.spi` — `SPIUtils`)
* `createTimedSPIMaster(pins: { sck: IPin; mosi?: IPin; miso?: IPin }, options: ITimedSPIMasterOptions): ITimedSPIMaster`
* `createSPISlave(sck: IPin, mosi: IPin, miso: IPin, done: (buffer: Uint8Array, count: number) => void, spiMode?: number): void`

#### UART (`this.simulation.api.uart` — `UARTUtils`)
* `createUART(config: IUARTConfig): IUART`

---

### 1.5. State, Properties, Events & Framebuffer

#### Static Properties (`this.simulation.componentProperties`)
* `getBooleanComponentProperty(propertyName: string): boolean`
* `getIntegerComponentProperty(propertyName: string): number`
* `getDecimalComponentProperty(propertyName: string): number`
* `getStringComponentProperty(propertyName: string): string`

#### Shared Runtime State (`this.simulation.runtimeState`)
* `getBooleanState(name: string): boolean` / `updateBooleanState(name: string, val: boolean): void` / `subscribeToBooleanStateUpdates(name: string, cb: (val: boolean) => void): void`
* `getIntegerState(name: string): number` / `updateIntegerState(name: string, val: number): void` / `subscribeToIntegerStateUpdates(name: string, cb: (val: number) => void): void`
* `getDecimalState(name: string): number` / `updateDecimalState(name: string, val: number): void` / `subscribeToDecimalStateUpdates(name: string, cb: (val: number) => void): void`
* `getStringState(name: string): string` / `updateStringState(name: string, val: string): void` / `subscribeToStringStateUpdates(name: string, cb: (val: string) => void): void`
* `getFrameBuffer(name: string): IFrameBuffer | null`
  * **`IFrameBuffer` (Logic):** `setPixel(x, y, r, g, b, a?)`, `fill(r, g, b, a)`, `clear()`, `getPixel(x, y)`, `getPixelRGBA(x, y)`, `getWidth()`, `getHeight()`

#### UI Events & Streams (`this.simulation.inputs` / `this.simulation.outputs`)
* `inputs.subscribeToEvent<TPayload>(inputId: string, callback: (event: ComponentInputEvent<TPayload>) => void): void`
* `inputs.openAudioStreamReader(inputId, callbacks)` / `openBinaryPacketReader(inputId)` / `openVideoFrameReader(inputId)`
* `outputs.sendEvent(outputId: string, payload?: unknown): boolean`
* `outputs.openAudioStream(outputId, format, options)` / `openBinaryPacketWriter(outputId)`

---

## 2. UI Script API (`@cirkit/simulation/ui`)

### Base Class

```typescript
import { AbstractSimulationComponentUI } from '@cirkit/simulation/ui';

export class SimulationComponentUI extends AbstractSimulationComponentUI {
  public init(): void {}
  public onSimulationPause(): void | Promise<void> {}
  public onSimulationResume(): void | Promise<void> {}
  public destroy(): void | Promise<void> {}
}
```

### UI Methods (`this.simulation`)
* `this.simulation.api.triggerUIRefresh(): void` — Forces template re-evaluation after updating public properties on `SimulationComponentUI`.
* `this.simulation.api.getComponentRotation(): number` — Component rotation angle in degrees.
* `this.simulation.api.addDocumentEventListener(type, listener)` / `removeDocumentEventListener(type, listener)`
* `this.simulation.api.log(message: string, level?: LogLevelEnum): void`
* `this.simulation.runtimeState` — Same `get*State`, `update*State`, `subscribeTo*StateUpdates`, and `getFrameBuffer` methods as Logic.
* `this.simulation.inputs.sendEvent(inputId: string, payload?: unknown): boolean` — Dispatches a discrete event to Logic.
* `this.simulation.outputs.subscribeToEvent(outputId: string, callback)` — Subscribes to discrete events from Logic.
* `this.simulation.componentAssets.getComponentAssetUrl(assetId: string): string | null` — Returns a temporary session URL for an attached asset.

---

## 3. UI HTML & JSON Schemas

### UI HTML Bindings
* **DOM Events:** `(click)="component.onToggle()"`, `(mousedown)="component.onPress()"`, `(mouseup)="component.onRelease()"`
* **Text Interpolation:** `{{ component.voltageText }}`
* **Dynamic Styles & Attributes:** `[style.opacity]="component.isLedOn ? '1' : '0.15'"`, `[style.width]="component.soc + '%'"`, `[attr.fill]="component.color"`

### Runtime State JSON (`runtimeStateVariables`)
```json
{
  "runtimeStateVariables": [
    {
      "id": "socPercent",
      "type": "decimal",
      "initialValue": "50",
      "min": "0",
      "max": "100",
      "isVisibleToUser": true,
      "label": "State of Charge",
      "units": "%",
      "userDisplayFormat": "range"
    }
  ],
  "componentInputs": [
    { "id": "toggleButton", "type": "event" }
  ],
  "componentOutputs": []
}
```
* Supported `type` values: `"boolean"`, `"integer"`, `"decimal"`, `"string"`, `"display"` (`display` requires `"width"` and `"height"`).
* Setting `"isVisibleToUser": true` exposes the variable in the component's runtime popup panel (`"userDisplayFormat": "range"` renders an interactive slider).

### Component Properties JSON (`componentProperties`)
```json
{
  "componentProperties": [
    {
      "version": 2,
      "id": "resistance",
      "label": "Resistance",
      "description": "Internal series resistance",
      "units": "Ω",
      "type": "decimal",
      "value": "0.05",
      "displayFormat": "input",
      "showOnComp": false
    }
  ]
}
```
* Required fields: `version` (`2`), `id`, `label`, `description`, `units`, `type` (`"boolean"` | `"integer"` | `"decimal"` | `"string"`), `displayFormat` (`"input"` | `"dropdown"`), `value` (always serialized as a string), `showOnComp`.

---

## 4. Minimal Working Pattern: Voltage Source with ESR (No Net Conflicts)

```typescript
import {
  AbstractSimulationComponentLogic,
  IPin,
  IVoltageSourceHandle,
  IResistorConnection
} from '@cirkit/simulation/logic';

export class SimulationComponentLogic extends AbstractSimulationComponentLogic {
  private pinPos!: IPin;
  private pinNeg!: IPin;
  private pinInternal!: IPin;
  private source!: IVoltageSourceHandle;
  private esr!: IResistorConnection;

  public init(): void {
    this.pinPos = this.simulation.api.pin.createInputPin('OUT+');
    this.pinNeg = this.simulation.api.pin.createInputPin('OUT-');
    this.pinInternal = this.simulation.api.pin.createInputPin('__INTERNAL_EMF');

    // Isolate ideal voltage source behind internal resistance (0.05 Ohm)
    this.source = this.simulation.api.connection.addVoltageSource(
      this.pinInternal,
      this.pinNeg,
      { voltage: 3.7 }
    )!;
    this.esr = this.simulation.api.connection.addResistor(
      this.pinInternal,
      this.pinPos,
      0.05
    );
  }
}
```
