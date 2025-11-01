# Heat Pump Integration in DAO_day_ahead.py

This document outlines the integration of the heat pump within the `DAO_day_ahead.py` script. The script optimizes energy consumption, and the heat pump is a significant component in this optimization.

## Overview

The script can control the heat pump in two main ways:
1.  **On/Off Control:** The script determines the optimal time to run the heat pump for a specific duration, based on energy prices and heat demand.
2.  **Power Control:** The script can modulate the power consumption of the heat pump, allowing for more granular control.

The choice between these two modes is determined by the `adjustment` setting in the `heating_options` configuration.

## Code Analysis

### 1. Initialization

In the `__init__` method of the `DaCalc` class, several attributes related to the heat pump are initialized. These attributes are used to store the configuration and state of the heat pump.

```python
class DaCalc(DaBase):
    def __init__(self, file_name=None):
        super().__init__(file_name=file_name)
        if self.config is None:
            return
        # ... (other initializations)
        self.heating_options = self.config.get(["heating"])
        # ...
        self.hp_present = False
        self.hp_enabled = False
        self.hp_adjustment = None
        self.hp_heat_demand = True
        # ...
```

### 2. Optimization Model

The core of the heat pump optimization is defined within the `calc_optimum` method. This is where the optimization model is built, and the constraints for the heat pump are defined.

#### Heat Pump Variables

The following variables are defined for the heat pump model:

```python
        p_hp = None
        h_hp = None
        c_hp = [
            model.add_var(var_type=CONTINUOUS, lb=0, ub=10) for _ in range(U)
        ]  # Electricity consumption per hour
        hp_on = [
            model.add_var(var_type=BINARY) for _ in range(U)
        ]  # If on the pump will run in that hour
```

#### Heat Pump Logic

The script first checks if the heat pump is present and enabled. If not, the consumption is set to zero.

```python
        self.hp_present = (
            self.config.get(["heater present"], self.heating_options, "False").lower()
            == "true"
        )
        if self.hp_present:
            entity_hp_enabled = self.config.get(
                ["entity hp enabled"], self.heating_options, None
            )
            self.hp_enabled = (entity_hp_enabled is None) or (
                self.get_state(entity_hp_enabled).state == "on"
            )
        else:
            self.hp_enabled = False
        if not self.hp_enabled:
            logging.info(
                "Warmtepomp niet aanwezig of enabled - warmtepomp wordt niet ingepland\n"
            )
            for u in range(U):
                model += c_hp[u] == 0
                model += hp_on[u] == 0
```

If the heat pump is enabled, the script calculates the heat demand and then applies one of two control strategies: "on/off" or "power".

##### On/Off Control

If `hp_adjustment` is set to `"on/off"`, the script calculates the number of hours the heat pump needs to run and then finds the cheapest hours to run it.

```python
            if self.hp_adjustment == "on/off":
                # ... (code for on/off control)
```

##### Power Control

If `hp_adjustment` is set to `"power"`, the script models the heat pump with multiple power stages, each with its own COP. The optimizer then determines the optimal power level for each interval.

```python
            else:
                # hp_adjustment == "power" or "heating curve"
                logging.info(f"Warmtepomp met power-regeling wordt ingepland\n")
                stages = self.heating_options["stages"]
                S = len(stages)
                c_hp = [
                    model.add_var(var_type=CONTINUOUS, lb=0, ub=stages[-1]["max_power"])
                    for _ in range(U)
                ]  # elektriciteitsverbruik in kWh/h
                # p_hp[s][u]: het gevraagde vermogen in W in dat uur
                p_hp = [
                    [
                        model.add_var(
                            var_type=CONTINUOUS, lb=0, ub=stages[s]["max_power"]
                        )
                        for _ in range(U)
                    ]
                    for s in range(S)
                ]

                # ... (constraints for power control)
```

### 3. Results Processing and Control

After the optimization is complete, the script processes the results and sends the appropriate commands to the heat pump via Home Assistant.

#### Logging Results

The script logs the planned operation of the heat pump.

```python
        if self.hp_present and self.hp_enabled:
            logging.info("\nInzet warmtepomp")
            if self.hp_adjustment == "on/off":
                if self.hp_heat_demand:
                    logging.info(f"u     tar    cons")
                    for u in range(U):
                        logging.info(f"{uur[u]} {pl[u]:6.4f} {c_hp[u].x:6.2f}")
            else:
                df_hp = pd.DataFrame(columns = ["uur", "tar"])
                df_hp["uur"] = uur
                df_hp["tar"] = pl
                for s in range(len(self.heating_options["stages"])):
                    values = []
                    for u in range(U):
                        values.append(p_hp[s][u].x)
                    df_hp["p"+str(s)] = np.array(values, dtype=np.int32)
                heat = []
                cons = []
                for u in range(U):
                    heat.append(h_hp[u].x)
                    cons.append(c_hp[u].x)
                df_hp["heat"] = heat
                df_hp["cons"] = cons
                pd.options.display.float_format = "{:7.3f}".format
                logging.info(f"\n{df_hp.to_string(index=False)}\n")
```

#### Sending Commands to Home Assistant

Finally, the script sends commands to Home Assistant to control the heat pump based on the optimization results. This includes turning the heat pump on or off, setting the power level, and adjusting the heating curve.

```python
            if self.hp_present and self.hp_enabled:
                # als aan/uit entity er is altijd schakelen
                entity_hp_switch = self.config.get(
                    ["entity hp switch"], self.heating_options, None
                )
                if entity_hp_switch is None:
                    if self.hp_adjustment == "on/off":
                        logging.warning(
                            f"Geen entity om warmtepomp in/uit te schakelen"
                        )
                else:
                    logging.debug(f"Warmtepomp entity: {entity_hp_switch}")
                    switch_state = self.get_state(entity_hp_switch).state
                    if hp_on[0].x == 1:
                        if switch_state == "off":
                            if self.debug:
                                logging.info(f"Warmtepomp zou zijn ingeschakeld")
                            else:
                                logging.info(f"Warmtepomp ingeschakeld")
                                self.turn_on(entity_hp_switch)
                    else:
                        if switch_state == "on":
                            if self.debug:
                                logging.info(f"Warmtepomp zou zijn uitgeschakeld")
                            else:
                                logging.info(f"Warmtepomp uitgeschakeld")
                                self.turn_off(entity_hp_switch)
                #  power, als entity er is altijd doorzetten
                entity_hp_power = self.config.get(
                    ["entity hp power"], self.heating_options, None
                )
                if entity_hp_power is not None and self.hp_adjustment != "on/off":
                    #  elektrisch vermogen in W
                    hp_power = 1000 * c_hp[0].x / hour_fraction[0]
                    if self.debug:
                        logging.info(
                            f"Elektrisch vermogen warmtepomp zou zijn ingesteld "
                            f"op {hp_power:<0.0f} W"
                        )
                    else:
                        self.set_value(entity_hp_power, hp_power)
                        logging.info(
                            f"Elektrisch vermogen warmtepomp ingesteld "
                            f"op {hp_power:<0.0f} W"
                        )

                #  curve adjustment
                entity_curve_adjustment = self.config.get(
                    ["entity adjust heating curve"], self.heating_options, None
                )
                if entity_curve_adjustment is not None:
                    old_adjustment = float(
                        self.get_state(entity_curve_adjustment).state
                    )
                    #  adjustment factor (K/%) bijv 0.4 K/10% = 0.04
                    adjustment_factor = self.config.get(
                        ["adjustment factor"], self.heating_options, 0.0
                    )
                    adjustment = calc_adjustment_heatcurve(
                        pl[0], p_avg, adjustment_factor, old_adjustment
                    )
                    if self.debug:
                        logging.info(
                            f"Aanpassing stooklijn zou zijn: {adjustment:<0.2f}"
                        )
                    else:
                        logging.info(f"Aanpassing stooklijn: {adjustment:<0.2f}")
                        self.set_value(entity_curve_adjustment, adjustment)
```

## Heating Demand and Heat Balance

The optimizer derives how much heat still needs to be produced before planning the heat pump.

- Degree days (weighted) are computed for today and, when optimizing beyond 24 hours, also for tomorrow.
- A configurable conversion factor translates degree days into required thermal energy:
  - Key: `heating.degree days factor`
  - Unit: kWh/K.day
- Already produced heat can be subtracted via a Home Assistant entity:
  - Key: `heating.entity hp heat produced`
  - Unit: kWh (numeric)
- The remaining thermal demand is: `heat_needed = max(0, degree_days * factor - heat_produced)`.
- An optional gating signal defines whether there is an actual heat demand right now:
  - Key: `heating.entity hp heat demand` (binary on/off)
  - If this signal is off, the heat pump is not planned (the first interval’s consumption and on-state are set to zero).
  - Distinction: if the heat pump is not present or disabled (`heating.entity hp enabled` is off, or `heating["heater present"]` is false), the model fixes consumption and on-state to zero for the entire horizon.

## On/Off Planning (Details)

When `heating.adjustment` is set to `"on/off"`, the optimizer plans a number of whole intervals for the heat pump to run.

- Minimum run length
  - Key: `heating.min run length`
  - Range: 1–5 hours. The optimizer enforces contiguous runs of at least this length.
- COP and nominal power
  - `heating.entity hp cop` (numeric). Default used if missing: 4.0.
  - `heating.entity hp power` (numeric, kW). Default used if missing: 1.5 kW.
- Sizing and rounding
  - Electrical energy needed: `e_needed = heat_needed / COP` (kWh).
  - Required run hours: `hp_hours = ceil(e_needed / hp_power)`.
  - `hp_hours` is rounded up to a multiple of `min run length` and enforced via equality constraints.
  - For non-hour intervals, the equality uses the number of intervals implied by the interval duration.
  - Note on intervals: with non-hourly intervals (e.g., 15 minutes), per-interval `c_hp[u]` equals the nominal kW only when an interval represents a full hour; otherwise, the optimizer enforces the total number of intervals to meet the required runtime.
- Export of average outside temperature
  - The average outside temperature over the horizon is optionally published to `heating.entity avg outside temp` (°C) for monitoring.

## Modulating (Power/Heating Curve) Planning (Details)

When `heating.adjustment` is `"power"` or `"heating curve"`, the optimizer uses multiple power stages, each with its own COP.

- Stages configuration (example):

```json
{
  "heating": {
    "stages": [
      { "max_power": 500,  "cop": 2.5 },
      { "max_power": 1000, "cop": 3.0 },
      { "max_power": 2000, "cop": 3.5 }
    ]
  }
}
```

- Variables and relations
  - `p_hp[s][u]` (W): requested electrical power per stage and interval, bounded by `stages[s].max_power`.
  - `hp_s_on[s][u]` (binary): stage selection per interval. Exactly one stage is active per interval.
  - `hp_on[u]` (binary): derived on/off indicator from the set of active stages.
  - `c_hp[u]` (kWh): electrical energy per interval: `sum(p_hp) * hour_fraction / 1000`.
  - `h_hp[u]` (kWh): thermal energy per interval: `sum(p_hp * cop) * hour_fraction / 1000`.
- Heat balance constraint
  - Total thermal energy equals the smaller of the remaining need and the producible maximum: `sum(h_hp) == min(heat_needed, max_heat_prod)`.
  - `max_heat_prod` comes from the top stage’s maximum heat power integrated over the horizon.
- Exclusivity with boiler (if applicable)
  - When the boiler is heated by the heat pump, the optimizer enforces per-interval exclusivity: either a heat pump stage is active or the boiler heats, not both.
- Note on stage indexing
  - Internally, `hp_on` is derived from stages with index ≥ 1. If you want `hp_on` to reflect any non-zero operation, configure stage 0 as “off” (e.g., `max_power: 0`), and use stages 1..N for real power levels.
  - Example with explicit off-stage:

```json
{
  "heating": {
    "stages": [
      { "max_power": 0,   "cop": 0.0 },  
      { "max_power": 800, "cop": 3.0 },
      { "max_power": 1600, "cop": 3.4 }
    ]
  }
}
```
  - Boiler note: exclusivity activates only when boiler configuration indicates it is heated by the heat pump (see boiler settings in the main DAO documentation/config).

## Entities and Units (Overview)

- `heating.adjustment`: `"on/off" | "power" | "heating curve"` (string)
- `heating.degree days factor`: kWh/K.day (number)
- `heating.entity hp enabled`: on/off gate for planning (binary)
- `heating.entity hp switch`: entity to toggle the heat pump (switch)
- `heating.entity hp power`: electrical power setpoint (W, number)
  - Input vs output: in on/off mode this is read as nominal power in kW (if configured); in power/heating-curve mode this is written as an electrical power setpoint in W.
- `heating.entity hp heat produced`: already produced heat (kWh, number)
- `heating.entity hp heat demand`: current heat demand (binary)
- `heating.entity hp cop`: COP reading (number)
- `heating.entity avg outside temp`: published average outside temperature (°C, number)
- `heating.entity adjust heating curve`: setpoint for curve adjustment (K or %, number)
- `heating.adjustment factor`: conversion factor for curve adjustment (K/%)
- `heating.min run length`: minimum contiguous run in hours (1–5)

## Control and Timing

The integration applies the next-step control to Home Assistant after optimization.

- Switching
  - If `heating.entity hp switch` is configured, the script toggles it based on the first planned interval (`hp_on[0]`).
- Power setpoint
  - If `heating.entity hp power` is set and `adjustment != "on/off"`, the script publishes the electrical power for the first interval: `hp_power = 1000 * c_hp[0] / hour_fraction[0]` (W).
- Heating curve adjustment
  - If `heating.entity adjust heating curve` is configured, the script computes a new value using the current price `pl[0]`, the average price `p_avg`, and `heating.adjustment factor`, then publishes it.
  - Debug mode
  - In debug mode, the script logs what would be commanded but does not write to entities.

## Quick Configuration

Use the snippet below as a compact starting point. Adjust entity ids to match your
Home Assistant setup.

```json
{
  "heating": {
    "heater present": true,
    "adjustment": "power",                 
    "degree days factor": 8.0,              
    "min run length": 2,                    
    "adjustment factor": 0.04,              

    "entity hp enabled": "input_boolean.hp_enabled",
    "entity hp switch": "switch.heat_pump",
    "entity hp power": "number.hp_power",          
    "entity hp heat produced": "sensor.hp_heat_kwh",
    "entity hp heat demand": "binary_sensor.heat_demand",
    "entity hp cop": "sensor.hp_cop",
    "entity avg outside temp": "sensor.outdoor_temp_avg",
    "entity adjust heating curve": "number.hp_curve_adjustment",

    "stages": [
      { "max_power": 0,    "cop": 0.0 },   
      { "max_power": 800,  "cop": 3.0 },
      { "max_power": 1600, "cop": 3.4 }
    ]
  }
}
```

Checklist by mode:

- Common
  - `heating["heater present"]` = true
  - `heating.adjustment` = `"on/off" | "power" | "heating curve"`
  - `heating.degree days factor` (kWh/K.day)
  - Optional gating: `heating.entity hp enabled`, `heating.entity hp heat demand`
- On/Off
  - Optional input: `heating.entity hp cop` (fallback 4.0)
  - Optional input: `heating.entity hp power` in kW (fallback 1.5)
  - Optional: `heating.min run length` (1–5)
  - Optional: publish `heating.entity avg outside temp`
- Power/Heating Curve
  - Required: `heating.stages` (see example)
  - Optional: write `heating.entity hp power` in W (setpoint)
  - Optional: heating curve control via `heating.entity adjust heating curve` and `heating.adjustment factor`
