# HA entities-mapping · HVAC Thuis

Dit document koppelt elke sensor-pil/data-tile in de visualisatie (04c en het dashboard) aan een concrete Home Assistant `entity_id`. Het is de single source of truth voor de Lovelace-config en automatiseringen.

---

## 1. Overzicht — bron-integraties

| Bron | Type | Geeft | Status |
|------|------|-------|--------|
| Toon (gehackt) | OpenTherm gateway | CV ketel modulatie/aanvoer/retour/brander/gas + huiskamer-temp + setpoint | bestaand |
| Remeha Avanta CW4 25 | via OpenTherm naar Toon | data komt van ketel via OT-protocol | bestaand |
| Naturela Pellet controller | RS485/Modbus of MQTT | pellet-vermogen, brander-status, pomp-temp, voorraad, klep-status | bestaand (zie ander project) |
| Tado | Tado integration | schuur-ambient temperatuur | bestaand |
| Boiler temp-sensoren (2x) | DS18B20 of vergelijkbaar via ESPHome | boven + onder tapwatertank | bestaand |
| Verdeler temp-sensoren | DS18B20 op aanvoer/retour rails | aanvoer + retour-temp boven & BG | **toevoegen** |
| VLW schuur groep-temps | DS18B20 op groep-aanvoer | T per groep G1/G2/G3 | **toevoegen** |
| 3-wegklep status | Naturela-controller of GPIO | pomp-temp + klep-positie | bestaand via Naturela |
| Mengkraan output-temp | DS18B20 op tap-warm leiding | output ≤ 65°C | **toevoegen** |
| Tap-warm bij douche | DS18B20 op tap-warm uitgang ketel | actuele tap-temp | **toevoegen** |
| Frank Energie | Frank Energie integration | dynamische gas + stroom prijzen | bestaand |
| Pellet-prijs | input_number helper | vaste prijs per kg | bestaand |

---

## 2. CV ketel · Remeha Avanta 25 KW CW4 (via Toon/OpenTherm)

| Sensor-pil in layout | Entity ID | Unit | Range | Klasse |
|---------------------|-----------|------|-------|--------|
| `mod 45%` | `sensor.cv_modulatie` | % | 0-100 | measurement |
| `aanv 62°C` | `sensor.cv_aanvoer_temp` | °C | 20-90 | measurement |
| `ret 47°C` | `sensor.cv_retour_temp` | °C | 20-80 | measurement |
| `● brander` | `binary_sensor.cv_brander_actief` | — | on/off | — |
| `0,9 m³/u` | `sensor.cv_gas_debiet` | m³/h | 0-3 | measurement |
| (nieuw) ketel-setpoint | `sensor.cv_setpoint` | °C | 20-90 | measurement |
| (nieuw) tap-flow setpoint | `sensor.cv_tap_setpoint` | °C | 40-65 | measurement |
| (nieuw) DHW actief | `binary_sensor.cv_tapwater_actief` | — | on/off | — |
| (nieuw) error-code | `sensor.cv_error_code` | — | text | — |

**Toon entiteiten al beschikbaar** (uit gehackte Toon): kamer-temp, setpoint, vraag, gas-meter (totaal m³).

```yaml
# Voorbeeld template in HA
- platform: template
  sensors:
    cv_aanvoer_retour_dt:
      friendly_name: "CV ΔT"
      unit_of_measurement: "°C"
      value_template: "{{ states('sensor.cv_aanvoer_temp')|float - states('sensor.cv_retour_temp')|float }}"
```

---

## 3. Restwarmte boiler · 200 L

| Pil | Entity ID | Unit | Toelichting |
|-----|-----------|------|-------------|
| `▲ 65°C` (boven) | `sensor.temp_boiler_top_temperatuur` | °C | top-sensor — warmste laag (stratificatie-top) |
| `▼ 38°C` (onder) | `sensor.temp_boiler_bottom_temperatuur` | °C | bottom-sensor — koudste laag |
| (afgeleid) ΔT stratificatie | `sensor.boiler_stratificatie_dt` | °C | template: boven − onder |
| (afgeleid) buffer-status | `sensor.boiler_buffer_status` | text | "vol" / "halfvol" / "leeg" o.b.v. boven >55°C / 40-55 / <40 |

**Hardware**: 2× **Tuya temperatuursensoren** (al geïnstalleerd op de tank-mantel, locatie Zolder). Geen DS18B20-installatie nodig.

---

## 4. Pellet-CV · Naturela

| Pil/data | Entity ID | Unit | Bron |
|----------|-----------|------|------|
| `17 kW · mod 80%` | `sensor.pellet_vermogen` + `sensor.pellet_modulatie` | kW / % | Naturela controller |
| `aan 64°` aanvoer | `sensor.pellet_aanvoer_temp` | °C | NTC op aanvoer-leiding |
| `ret 38°` retour | `sensor.pellet_retour_temp` | °C | NTC op retour-leiding |
| `voorraad 28 kg` | `sensor.pellet_voorraad_kg` | kg | berekend uit verbruik (utility_meter) |
| (afgeleid) `~3d` | `sensor.pellet_voorraad_dagen` | dagen | template: voorraad / gem. dagverbruik |
| `● brander` | `binary_sensor.pellet_brander_actief` | — | controller |
| (nieuw) brand-fase | `sensor.pellet_fase` | text | "uit"/"opstart"/"branden"/"naverbranding"/"asla" |
| (nieuw) pomp-uren totaal | `sensor.pellet_pomp_uren` | h | counter |
| (nieuw) brander-uren totaal | `sensor.pellet_brander_uren` | h | counter |

**Voorraad berekening**: 
```yaml
sensor:
  - platform: template
    sensors:
      pellet_voorraad_kg:
        friendly_name: "Pellet voorraad"
        unit_of_measurement: kg
        value_template: >-
          {{ states('input_number.pellet_laatste_bestelling_kg')|float
             - states('sensor.pellet_verbruik_sinds_bestelling')|float }}
```

---

## 5. Verdelers — GESCHRAPT

**Beslissing**: geen aparte temp-sensoren op de verdelers. Reden:
- Verdelers zitten weggebouwd zonder spanning in de buurt — installatie is fysiek lastig
- CV-aanvoer/retour wordt al gemeten bij de ketel via OpenTherm (Toon)
- Verdeler-temps wijken hooguit 1-3°C af door pijp-warmteverlies — geen toegevoegde diagnostische waarde
- Afgifte-problemen (lucht, dichte thermostaatkraan) los je niet op via verdeler-temp maar via radiator-temp / kamertemp-trend

**Indien later toch gewenst**: gebruik battery-powered Bluetooth temp-sensoren (Aqara/Xiaomi) op de manifold-rails — geen spanning nodig, mantel-meting is voldoende voor trends.

---

## 6. VLW Schuur · 3 groepen

| Pil | Entity ID | Unit | Notitie |
|-----|-----------|------|---------|
| `G1 28°` | `sensor.vlw_schuur_g1_temp` | °C | aanvoer-temp groep 1 |
| `G2 27°` | `sensor.vlw_schuur_g2_temp` | °C | aanvoer-temp groep 2 |
| `G3 26°` | `sensor.vlw_schuur_g3_temp` | °C | aanvoer-temp groep 3 |
| (nieuw) aanvoer-rail temp | `sensor.vlw_schuur_aanvoer_temp` | °C | gemeenschappelijke aanvoer |
| (nieuw) retour-rail temp | `sensor.vlw_schuur_retour_temp` | °C | gemeenschappelijke retour |

---

## 7. 3-wegklep + bypass

| Pil | Entity ID | Unit | Bron |
|-----|-----------|------|------|
| `42°C` (binnen klep-box) | `sensor.driewegklep_pomp_temp` | °C | Naturela of NTC |
| (afgeleid) klep-positie | `binary_sensor.driewegklep_naar_systeem` | — | template: pomp-temp ≥ 35°C |
| bypass-status | `binary_sensor.driewegklep_bypass_actief` | — | inverse van bovenstaande |

---

## 8. Mengkraan + tapwater

| Pil/data | Entity ID | Unit | Toelichting |
|----------|-----------|------|-------------|
| Mengkraan-output | `sensor.mengkraan_output_calculated` | °C | **Template-sensor**, geen fysieke meting. Mengkraan is thermostatisch — output is mechanisch begrensd op 65°C. Berekening: `min(boiler_boven, 65)`. |
| `tap 55°C` (bij douche) | `sensor.tap_warm_temp` | °C | DS18B20 op tap-warm-uitgang ketel — toont buffer-leeg-effect bij gebruiker |
| (optioneel) KW-net druk | `sensor.kw_net_druk` | bar | low priority |
| (optioneel) tap-flow actief | `binary_sensor.tap_actief` | — | komt al uit Toon DHW-status |
| (optioneel) tap-water L vandaag | `sensor.tap_water_dag_liter` | L | flow-meter nodig |

**Template voor mengkraan-output**:
```yaml
template:
  - sensor:
      - name: "Mengkraan output (berekend)"
        unique_id: mengkraan_output_calculated
        unit_of_measurement: "°C"
        state: >-
          {{ [states('sensor.boiler_temp_boven')|float, 65]|min }}
```

---

## 9. Toon huiskamerthermostaat

| Pil | Entity ID | Unit |
|-----|-----------|------|
| Huiskamer 21,2°C | `sensor.toon_kamer_temp` | °C |
| Setpoint 21,5°C | `sensor.toon_setpoint` | °C |
| Warmtevraag | `binary_sensor.toon_warmtevraag` | — |
| Programma actief | `sensor.toon_programma` | text |

---

## 10. Tado · schuur-ambient

| Pil | Entity ID | Unit |
|-----|-----------|------|
| `12,4°C` | `sensor.tado_schuur_temp` | °C |
| Luchtvochtigheid | `sensor.tado_schuur_humidity` | % |
| Status | `sensor.tado_schuur_battery` | % |

---

## 11. Frank Energie · dynamische tarieven

| Tile | Entity ID | Unit |
|------|-----------|------|
| Gas € 1,42/m³ | `sensor.frank_energie_gas_huidige_prijs` | €/m³ |
| Stroom € 0,28/kWh | `sensor.frank_energie_stroom_huidige_prijs` | €/kWh |
| Gas vandaag gem | `sensor.frank_energie_gas_vandaag_gem` | €/m³ |
| Stroom volgend uur | `sensor.frank_energie_stroom_volgend_uur` | €/kWh |

---

## 12. Pellet-prijs (handmatig)

| Tile | Entity ID | Unit |
|------|-----------|------|
| € 425/ton | `input_number.pellet_prijs_per_ton` | €/ton |
| € 6,38/zak | (afgeleid) | €/zak |
| Laatste bestelling | `input_number.pellet_laatste_bestelling_kosten` | € |
| Datum laatste bestelling | `input_datetime.pellet_laatste_bestelling_datum` | date |

---

## 13. Utility meters (HA Energy Dashboard)

```yaml
utility_meter:
  cv_gas_dag:
    source: sensor.cv_gas_totaal_m3
    cycle: daily
  cv_gas_maand:
    source: sensor.cv_gas_totaal_m3
    cycle: monthly
  cv_gas_jaar:
    source: sensor.cv_gas_totaal_m3
    cycle: yearly
  pellet_kg_dag:
    source: sensor.pellet_verbruik_totaal_kg
    cycle: daily
  pellet_kg_maand:
    source: sensor.pellet_verbruik_totaal_kg
    cycle: monthly
  pellet_kg_jaar:
    source: sensor.pellet_verbruik_totaal_kg
    cycle: yearly
```

---

## 14. Berekende kosten-sensoren

```yaml
template:
  - sensor:
      - name: "Gas kosten vandaag"
        unique_id: gas_kosten_dag
        unit_of_measurement: "€"
        state: >-
          {{ (states('sensor.cv_gas_dag')|float
              * states('sensor.frank_energie_gas_huidige_prijs')|float)|round(2) }}

      - name: "Pellet kosten vandaag"
        unique_id: pellet_kosten_dag
        unit_of_measurement: "€"
        state: >-
          {{ (states('sensor.pellet_kg_dag')|float
              * states('input_number.pellet_prijs_per_ton')|float / 1000)|round(2) }}

      - name: "Totaal energie kosten vandaag"
        unique_id: totaal_kosten_dag
        unit_of_measurement: "€"
        state: >-
          {{ (states('sensor.gas_kosten_dag')|float
              + states('sensor.pellet_kosten_dag')|float)|round(2) }}

      - name: "Besparing pellet vandaag"
        unique_id: besparing_pellet_dag
        unit_of_measurement: "€"
        state: >-
          {# pellet kWh × gasprijs/kWh-equivalent − pellet werkelijke kosten #}
          {% set pellet_kwh = states('sensor.pellet_kg_dag')|float * 4.8 %}
          {% set gas_alt_m3 = pellet_kwh / 9.77 %}
          {% set gas_alt_kosten = gas_alt_m3 * states('sensor.frank_energie_gas_huidige_prijs')|float %}
          {{ (gas_alt_kosten - states('sensor.pellet_kosten_dag')|float)|round(2) }}
```

**Energie-equivalenten** (vaste constanten):
- 1 m³ gas = 9,77 kWh (Slochteren-gas)
- 1 kg pellet = 4,8 kWh (gemiddeld witte pellets klasse A1)
- 1 kg pellet ≈ 0,49 m³ gas-equivalent

---

## 15. Counters voor brander-uren

```yaml
counter:
  cv_brander_uren:
    name: "CV ketel brander uren"
    initial: 0
    step: 1
    restore: true

  pellet_brander_uren:
    name: "Pellet brander uren"
    initial: 0
    step: 1
    restore: true

automation:
  - alias: "Tel CV brander uur"
    trigger:
      - platform: time_pattern
        hours: "/1"
    condition:
      - condition: state
        entity_id: binary_sensor.cv_brander_actief
        state: "on"
    action:
      - service: counter.increment
        target:
          entity_id: counter.cv_brander_uren

  - alias: "Tel pellet brander uur"
    trigger:
      - platform: time_pattern
        hours: "/1"
    condition:
      - condition: state
        entity_id: binary_sensor.pellet_brander_actief
        state: "on"
    action:
      - service: counter.increment
        target:
          entity_id: counter.pellet_brander_uren
```

---

## 16. Implementatie-checklist

- [ ] Toon-integratie verifieren (entiteiten beschikbaar in HA)
- [ ] Naturela MQTT/Modbus integratie installeren
- [ ] Tado integration koppelen
- [ ] **Hardware installeren** (na schrap verdelers + mengkraan, en Tuya boiler-sensoren al beschikbaar): 3× VLW groep-DS18B20, 1× tap-warm-DS18B20 = **4 nieuwe sensoren totaal**
- [ ] ESPHome node configureren voor de DS18B20-sensoren
- [ ] Frank Energie integration installeren + API-key
- [ ] `input_number.pellet_prijs_per_ton` als helper aanmaken
- [ ] Utility meters definiëren in `configuration.yaml`
- [ ] Template-sensoren voor kosten/besparing
- [ ] Counters voor brander-uren
- [ ] Lovelace-config updaten met alle nieuwe entities (zie 05-Lovelace-config.yaml)

---

## 17. Naming-conventies

Alle entiteiten volgen het patroon: `{type}.{component}_{eigenschap}`

- `sensor.cv_*` = CV-ketel-gerelateerd
- `sensor.boiler_*` = restwarmte boiler
- `sensor.pellet_*` = pellet CV
- `sensor.verdeler_boven_*`, `sensor.verdeler_bg_*` = verdelers
- `sensor.vlw_schuur_*` = vloerverwarming schuur
- `sensor.mengkraan_*`, `sensor.tap_*` = tapwater-systeem
- `binary_sensor.*_actief` = aan/uit-statussen
- `counter.*_uren` = uren-tellers

Friendly names in NL, entity_id in lowercase met underscores.
