# Voorstel: Delta-Temperatuur Output voor Warmtepomp in DAO

Dit voorstel voegt een tweede aansturingsoptie (Delta‑Temperatuur) toe aan de
warmtepomp-integratie in `DAO_day_ahead.py`. De optimizer blijft het optimale
schema bepalen, maar in plaats van de warmtepomp direct te schakelen, publiceert
de code een fictieve doeltemperatuur naar Home Assistant. Een losse
automatisering schakelt op basis van de actuele binnentemperatuur t.o.v. deze
doeltemperatuur.

## Doelen

- Comfort en kosten: optimaliseer nog steeds t.o.v. prijs/thermische vraag.
- Bedieningsvrijheid: gebruiker kan eenvoudig ingrijpen via target‑helper.
- Minimale impact: geen breuk met bestaande “on/off” of “power/heating curve”.
- Duidelijke configuratie en veilige defaults.

## Configuratie (nieuwe en bestaande velden)

Nieuwe velden in `heating` (JSON‑voorbeeld):

```json
{
  "heating": {
    "output mode": "delta_temperature",        
    "baseline temperature": 20.0,               
    "delta when on": 1.5,                       
    "delta when off": -1.5,                     
    "hard min temperature": 19.0,               
    "entity hp delta target": "number.hp_setpoint_planned"
  }
}
```

- `output mode`: `"delta_temperature" | "direct"` (default: `"direct"`).
  - `direct` laat het huidige gedrag intact (schakelen/power/curve).
  - `delta_temperature` publiceert een doeltemperatuur en schakelt zelf niet.
- `baseline temperature` (°C): basis setpoint waar de delta op wordt toegepast.
- `delta when on` (°C): positief getal; wordt opgeteld als advies “aan”.
- `delta when off` (°C): negatief getal; wordt opgeteld als advies “uit”.
- `hard min temperature` (°C): ondergrens voor comfort.
- `entity hp delta target`: HA-helper (Number) waar de fictieve doeltemperatuur
  naar wordt wegschreven.

Entiteit-alternatieven (voorkeursvolgorde: entiteit > literal > default):

- `entity output mode`: string‑entiteit; wanneer aanwezig overschrijft deze `output mode`.
- `entity baseline temperature`: Number-entiteit (°C) voor baseline.
- `entity delta when on`: Number-entiteit (°C) voor delta bij “aan”.
- `entity delta when off`: Number-entiteit (°C) voor delta bij “uit”.
- `entity hard min temperature`: Number-entiteit (°C) voor harde ondergrens.

Voorbeeld met entiteiten:

```json
{
  "heating": {
    "output mode": "direct",                              
    "entity output mode": "input_select.hp_output_mode",  
    "entity baseline temperature": "input_number.hp_base", 
    "entity delta when on": "input_number.hp_delta_on",    
    "entity delta when off": "input_number.hp_delta_off",  
    "entity hard min temperature": "input_number.hp_min",  
    "entity hp delta target": "number.hp_setpoint_planned"
  }
}
```

Opmerking bestaande velden (reeds in gebruik):
- `heating.adjustment`: `"on/off" | "power" | "heating curve"` (blijft bestaan).
- Entiteiten voor schakelen/vermogen/stooklijn blijven werken in `direct`‑modus.

## Algoritme Delta‑Temperatuur

De optimizer levert voor elk interval `u` de binaire inzet `hp_on[u]` (of in het
modulerende pad: een afgeleide `hp_on[u]`). Op basis hiervan wordt een
Delta-T over de volledige horizon wordt per stap naar Home Assistant gestuurd.

Per interval `u`:
1. Bepaal advies: `is_on[u] = (hp_on[u] == 1)`.
2. Voorlopige doeltemperatuur:
   - Als `is_on[u]`: `temp[u] = baseline + (delta when on)`
   - Anders: `temp[u] = baseline + (delta when off)`
3. Comfortbeperking per interval: `temp[u] = max(hard_min, temp[u])`.

Output en runtime‑aansturing:
- De volledige `temp[u]`‑reeks vormt het geplande Delta‑T profiel over de
  horizon (planning). Deze kan gelogd worden voor inzicht.
- Voor de aansturing wordt uitsluitend de eerstvolgende stap (`u = 0`)
  naar Home Assistant gepubliceerd (rolling horizon).

Debug‑modus: alleen loggen, geen writes.

## Verwerking Randvoorwaarden

De randvoorwaarden worden als volgt verwerkt of optioneel gemaakt:

- Thermisch gebouwmodel (hybride met energiedoel)
  - Voorzie een schakelaar: `heating.thermal model enabled` (bool/entiteit) om het lineaire model te activeren:
    `T_in[u+1] = T_in[u] - (T_in[u] - T_out[u]) * C_loss + P_th[u] * C_gain`.
  - Config/entiteiten:
    - `loss coefficient` (C_loss) en `gain coefficient` (C_gain) of entiteiten: `entity loss coefficient`, `entity gain coefficient`.
    - Initiële toestand: `entity indoor temperature` (T_in[0]) of `indoor temperature` (literal).
    - Comfortgrenzen: `comfort temp min`/`comfort temp max` of entiteiten: `entity comfort temp min`/`entity comfort temp max`.
  - Koppeling warmtepomp: `P_th[u]` is de geproduceerde warmte en kan rechtstreeks uit de bestaande variabele `h_hp[u]` komen (kWh per interval). Voor consistentie met de formule in vermogen kan men werken met interval‑genormaliseerde grootheden; in de praktijk volstaat het lineair verband met `h_hp[u]`.
  - MILP‑invoeging: voeg continue variabelen `T_in[u]` toe en de bovenstaande lineaire gelijkheden en grenzen `T_min <= T_in[u] <= T_max`.

- Hybride koppeling energie + temperatuur (blijft gekoppeld aan energiebehoefte)
  - Aanbevolen: gebruik een ondergrens i.p.v. strikte gelijkheid om infeasibility te vermijden en flexibiliteit te behouden:
    - Ondergrens: `sum(h_hp[u]) >= heat_needed` en bovengrens `sum(h_hp[u]) <= max_heat_prod`.
    - Optioneel: introduceer `excess_heat >= 0` met `excess_heat >= sum(h_hp[u]) - heat_needed` en voeg een kleine penalty toe in de objective: `epsilon_excess * excess_heat` om overproductie te ontmoedigen maar toe te staan wanneer comfort/prijsprofiel dat rechtvaardigt.
  - Voeg het thermische model + comfortband toe zodat het plan ook thermisch consistent is en comfort garandeert.

- Prestaties warmtepomp
  - In het voorstel blijft de huidige DAO‑implementatie met `stages` en per‑stage COP in stand (lineair), wat functioneel overeenkomt met een tabelmatige `COP(T_out)` benadering.

- Comforteisen
  - Met het optionele thermische model: afdwingen via `T_min`/`T_max` per interval.
  - Zonder thermisch model: comfort wordt operationeel geborgd door de delta‑temperatuuroutput en HA‑automatisering. Aanvullend kan een absolute ondergrens (`hard min temperature`) worden gehanteerd in de output.

- Praktische beperkingen (schakelen/modulatie)
  - Minimale looptijd: reeds opgenomen via `min run length` in on/off.
  - Schakelstraf (optioneel): introduceer hulpvariabelen `sw[u] >= |hp_on[u] - hp_on[u-1]|` en voeg `lambda_switch * sum(sw)` toe aan de doelstelling, of beperk `sum(sw) <= max_switches`.
    - Config/entiteiten: `switch penalty` (lambda), `max switches per day` (int) en `entity switch penalty`, `entity max switches per day`.
  - Modulatie‑gating: blijft zoals geïmplementeerd via `p_hp[s][u] <= max_power_s * hp_s_on[s][u]` en “exact één stage actief”.

- Initiële conditie
  - Indien het thermische model is geactiveerd: T_in[0] vanuit `entity indoor temperature` of fallback literal.

## Integratiepunten in DAO_day_ahead.py

Referentie op basis van huidige code:
- Initialisatieveld/opties bij het inlezen van `heating_options` (rond
  `DAO_day_ahead.py:42`).
- Eindsprint “heatpump” aansturing na de optimalisatie (rond
  `DAO_day_ahead.py:3184`).

### 1) Config lezen en defaults

Python (indicatief):

```python
# in __init__ na self.heating_options = ...

# Output mode: entiteit > literal > default
entity_output_mode = self.config.get(["entity output mode"], self.heating_options, None)
if entity_output_mode:
    self.hp_output_mode = self.get_state(entity_output_mode).state.lower()
else:
    self.hp_output_mode = self.config.get(["output mode"], self.heating_options, "direct").lower()

# Numerieke instellingen via helper: entiteit > literal > default
self.hp_baseline_temp = float(self.get_setting_state("baseline temperature", self.heating_options, default=20.0))
self.hp_delta_on = float(self.get_setting_state("delta when on", self.heating_options, default=1.5))
self.hp_delta_off = float(self.get_setting_state("delta when off", self.heating_options, default=-1.5))
self.hp_hard_min = float(self.get_setting_state("hard min temperature", self.heating_options, default=19.0))
self.entity_hp_delta_target = self.config.get(["entity hp delta target"], self.heating_options, None)

# (Optioneel) Thermisch model en comfort
self.thermal_model_enabled = (self.config.get(["thermal model enabled"], self.heating_options, "false").lower() == "true")
self.loss_coeff = float(self.get_setting_state("loss coefficient", self.heating_options, default=0.0))
self.gain_coeff = float(self.get_setting_state("gain coefficient", self.heating_options, default=0.0))
self.indoor_temp0 = float(self.get_setting_state("indoor temperature", self.heating_options, default=20.0))
self.comfort_min = float(self.get_setting_state("comfort temp min", self.heating_options, default=None) or 0.0)
self.comfort_max = float(self.get_setting_state("comfort temp max", self.heating_options, default=None) or 100.0)

# (Optioneel) Schakelstraf/limiet
self.switch_penalty = float(self.get_setting_state("switch penalty", self.heating_options, default=0.0))
self.max_switches = int(self.get_setting_state("max switches per day", self.heating_options, default=None) or 0)
```

### 2) Aansturingsfase (na optimalisatie)

In het bestaande blok "heatpump" (rond `DAO_day_ahead.py:3184`) wordt een
tak toe vóór het directe schakelen/power/curve‑schrijven, die actief wordt als
`output mode == 'delta_temperature'`.

```python
if self.hp_present and self.hp_enabled:
    if self.hp_output_mode == "delta_temperature":
        # Plan Delta‑T over de horizon op basis van hp_on[u]
        temp_plan = []
        for u in range(U):
            is_on = (hp_on[u].x == 1)
            t = self.hp_baseline_temp + (self.hp_delta_on if is_on else self.hp_delta_off)
            t = max(self.hp_hard_min, t)
            temp_plan.append(t)

        # Publiceer alleen de eerstvolgende stap (rolling horizon)
        if self.entity_hp_delta_target is None:
            logging.warning("Geen entity voor delta-temperature output geconfigureerd")
        else:
            if self.debug:
                logging.info(f"Fictieve doeltemperatuur (u=0) zou worden ingesteld op {temp_plan[0]:.1f} °C")
            else:
                self.set_value(self.entity_hp_delta_target, temp_plan[0])
                logging.info(f"Fictieve doeltemperatuur (u=0) ingesteld op {temp_plan[0]:.1f} °C")

        # Log het geplande Delta‑T profiel voor inzicht/debug
        logging.info("Gepland Delta‑T profiel (°C): " + ", ".join(f"{t:.1f}" for t in temp_plan))

        # In deze modus geen direct schakelen/power/curve om conflicten te voorkomen
    else:
        # bestaand gedrag: switch/power/curve
        ...
```

### 3) Buitentemperatuurreeks (T_out) per interval

Voor het thermische model is een tijdreeks van buitentemperaturen nodig, met één waarde per optimalisatie‑interval. Breid de `self.meteo` interface uit (of gebruik een bestaande) om een reeks op te halen. Voorstel van helper:

```python
# Verwacht: lijst met U waarden in °C, uitgelijnd met de optimalisatie‑intervallen
try:
    T_out = self.meteo.get_outdoor_series(start_dt=start_interval_dt, steps=U, seconds=self.interval_s)
except AttributeError:
    # Fallback: gebruik bekende gemiddelden (vandaag/morgen) of één constante
    if U > 24:
        avg_today = self.meteo.get_avg_temperature()
        avg_tomorrow = self.meteo.get_avg_temperature(
            date=dt.datetime.combine(dt.date.today() + dt.timedelta(days=1), dt.datetime.min.time())
        )
        avg = 0.5 * (avg_today + avg_tomorrow)
    else:
        avg = self.meteo.get_avg_temperature()
    T_out = [avg for _ in range(U)]
```

Gebruik `T_out[u]` in de T_in‑dynamica wanneer `thermal model enabled` actief is.

Belangrijk: de `else`‑tak behoudt ongewijzigd het huidige gedrag voor
achterwaartse compatibiliteit. In `delta_temperature` wordt er niet geswitcht
of power/curve gezet, zodat HA‑automatisering exclusief bepaalt of de pomp aan
gaat.

## Home Assistant‑automatisering (voorbeeld)

Doel: schakel de warmtepomp op basis van de fictieve doeltemperatuur t.o.v. de
actuele binnentemperatuur, met hysterese om pendelen te voorkomen.

```yaml
alias: HP schakelen op fictieve doeltemperatuur
mode: single
trigger:
  - platform: state
    entity_id: sensor.indoor_temperature
  - platform: state
    entity_id: number.hp_setpoint_planned
condition: []
action:
  - choose:
      - conditions:
          - condition: template
            value_template: >-
              {{ states('sensor.indoor_temperature') | float(0) <
                 (states('number.hp_setpoint_planned') | float(0)) - 0.1 }}
        sequence:
          - service: switch.turn_on
            target:
              entity_id: switch.heat_pump
      - conditions:
          - condition: template
            value_template: >-
              {{ states('sensor.indoor_temperature') | float(0) >=
                 (states('number.hp_setpoint_planned') | float(0)) + 0.1 }}
        sequence:
          - service: switch.turn_off
            target:
              entity_id: switch.heat_pump
```

Hysterese van ±0.1 °C is voorbeeld; pas aan naar wens.

## Edge-cases en gedrag

- Geen warmtevraag (heat demand off): optimizer kan 0 inplannen; in
  `delta_temperature` resulteert dit in de “uit”-delta t.o.v. de baseline,
  begrensd door `hard min temperature`.
- Niet aanwezig/disabled: volledige horizon wordt 0; er wordt niets geschreven
  (de hoofdcode heeft deze gating al), de beoogde target wordt in
  debug-modus gelogd.
- Intervalresolutie: uitsluitend het eerstvolgende interval wordt aangestuurd,
  (rolling horizon), in lijn met de bestaande switch/power/curve-aansturing.
## Achterwaartse compatibiliteit

- Standaard blijft `output mode = direct` en verandert er niets.
- De nieuwe modus gebruikt alleen aanvullende configuratie; ontbrekende keys
  krijgen veilige defaults of geven een duidelijke waarschuwing (en doen
  verder niets).

## Testplan (handmatig)

1. Config: voeg `output mode: delta_temperature` toe en configureer
   `entity hp delta target` (Number helper), baseline/delta/min.
2. Draai DAO met debug‑modus om logging te verifiëren; controleer dat er geen
   directe switch/power‑acties worden gedaan in deze modus.
3. Verifieer dat `number.hp_setpoint_planned` correct wordt geüpdatet t.o.v.
   geoptimaliseerd `hp_on[0]` en configuratie.
4. Activeer HA‑automatisering en controleer schakelen met hysterese.

## Open punten / uitbreidingen

- Optionele bovengrens (`hard max temperature`) om overshoot te temperen.
- Baseline uit entiteit i.p.v. constant: bijv. `entity baseline temperature`.
- Alternatief besliscriterium in modulerend pad: drempel op `c_hp[0] > 0` i.p.v.
  `hp_on[0]`, indien stage-indexing anders wordt ingericht.

- Comfort als entiteit: `entity comfort temp min/max` opnemen in de delta-modus
  om een dynamische onder-/bovengrens in de output te respecteren.

## Aanvullingen en preciseringen

- Energiebehoefte + comfort (hybride):
  - Gebruik een ondergrens i.p.v. strikte gelijkheid: `sum(h_hp[u]) >= heat_needed` en `sum(h_hp[u]) <= max_heat_prod`.
  - Voeg optioneel `excess_heat >= sum(h_hp) - heat_needed` toe en penaliseer licht in de objective.
- Interval-schaal en eenheden:
  - `C_loss` is per-interval (uur/kwartier). Bij wisselen van intervalduur herkalibreren.
  - `C_gain` in °C/kWh; `h_hp[u]` is al in kWh per interval (via `hour_fraction`).
- Schakelstraf (linearisatie):
  - Binaire `sw[u]` met constraints `sw[u] >= hp_on[u] - hp_on[u-1]` en `sw[u] >= hp_on[u-1] - hp_on[u]` (optioneel: strakkere bovengrenzen).
  - Objective-term: `switch_penalty * sum(sw)` of limiet `sum(sw) <= max_switches`.
- Modulatie “aan”-detectie voor Delta‑T:
  - Indien `hp_on[u]` niet dekkend is door stage-index 0 = off, gebruik `is_on[u] = (c_hp[u] > 0)` met kleine drempel.
- Buitentemperatuurreeks (T_out):
  - Implementeer `get_outdoor_series(start_dt, steps, seconds)` in `self.meteo` of hergebruik bestaande helper; align met interval-grid.

## Samenvatting wijzigingslocaties

- `DAO_day_ahead.py:42` (init): nieuwe configkeys lezen en defaults zetten.
- `DAO_day_ahead.py:3184` (aansturing): nieuwe `delta_temperature`‑tak vóór
  switch/power/curve‑aansturing.
- Documentatie: `warmtepomp_dao.md`—entiteiten/units aanvullen met
  `entity hp delta target`, en “Control and Timing” uitbreiden met
  delta‑temperatuur‑modus.

Optioneel (bij gebruik thermisch model en schakelstraf):
- In `calc_optimum`: voeg `T_in[u]` variabelen en lineaire dynamica toe onder
  het “heatpump” blok, vóór de overige apparaten.
- In de objective: voeg `switch_penalty * sum(sw)` toe; definieer hulpvariabelen
  `sw[u]` met lineaire begrenzingen op de absolute overgang.
