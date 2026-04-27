# 02 — Home Assistant architectuur

## 1. Lagenmodel

```
┌─────────────────────────────────────────────────────────────┐
│   PRESENTATIE                                                │
│   Lovelace dashboard (dark glassmorphism, custom cards)      │
└─────────────────────────────────────────────────────────────┘
                            ▲
┌─────────────────────────────────────────────────────────────┐
│   LOGICA / TEMPLATES                                         │
│   - Template sensors (actieve bron, besparing, COP-proxy)    │
│   - Helpers (input_boolean, input_number, threshold)         │
│   - Automations (alerts, modes, scenarios)                   │
└─────────────────────────────────────────────────────────────┘
                            ▲
┌─────────────────────────────────────────────────────────────┐
│   INTEGRATIE                                                 │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │ Toon     │  │ Naturela │  │ Boiler   │  │ Energie  │   │
│   │ (MQTT/   │  │ (MQTT/   │  │ ESPHome  │  │ DSMR/P1  │   │
│   │  REST)   │  │  Modbus) │  │ DS18B20  │  │          │   │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ▲
┌─────────────────────────────────────────────────────────────┐
│   FYSIEKE LAAG                                               │
│   CV-ketel · Boiler (T1/T2) · Pellet · Pompen · Sensoren    │
└─────────────────────────────────────────────────────────────┘
```

## 2. Integratie per bron

### 2.1 Toon thermostaat
- **Voorkeur:** Toon via MQTT (Toon Rooted + Toon2MQTT) → native HA via MQTT discovery.
- **Fallback:** REST sensor naar lokale Toon API.
- **Te ontsluiten:**
  - `climate.toon` (huidige + setpoint + modes)
  - `binary_sensor.toon_burner` (vlam aan/uit) — kritiek voor "actieve bron" logica
  - `sensor.toon_gas_today`, `sensor.toon_modulation_level`

### 2.2 Naturela pellet controller
- **Voorkeur:** Modbus RTU → ESP32 met ESPHome → MQTT/HA.
- **Alternatief:** als de Naturela al een MQTT bridge heeft, direct subscriben.
- **Te ontsluiten:**
  - `sensor.pellet_temp_aanvoer`
  - `sensor.pellet_temp_retour`
  - `binary_sensor.pellet_pomp` (pomp aan boven 35 °C)
  - `binary_sensor.pellet_3weg_klep` (proxy via pomp als nodig)
  - `sensor.pellet_voorraad` (indien gewogen of geschat)
  - `sensor.pellet_vermogen` (kW, berekend uit ΔT × debiet)

### 2.3 Restwarmte‑boiler
- **Hardware:** 2× DS18B20 1‑Wire op ESP8266/ESP32 met ESPHome.
- **Te ontsluiten:**
  - `sensor.boiler_temp_boven` (T1)
  - `sensor.boiler_temp_onder` (T2)
- **Afgeleid:**
  - `sensor.boiler_delta_t` = T1 − T2 (laadtoestand)
  - `sensor.boiler_laadstatus` (template: "Geladen" / "Halfvol" / "Leeg")

### 2.4 CV‑ketel (combi)
- Indien OpenTherm gateway aanwezig: `climate.cv_ketel` + modulatie.
- Anders: afgeleid via Toon (vlam‑status), gasmeter (P1) en eventueel Shelly EM op pomp.

### 2.5 Vloerverwarming badkamer
- Pomp‑status via slimme stekker (Shelly Plug S) of bestaande GPIO‑output.
- Optioneel: aanvoer/retour temp met DS18B20 op verdeler.

### 2.6 Energie
- **Gas:** P1 / DSMR reader → `sensor.gas_consumption_today`.
- **Pellet:** geschat (handmatige inboeking) of gewogen.
- **Elektra pompen:** Shelly EM aan totaalpomp‑groep.

## 3. Naming‑conventie

```
[domain].[locatie]_[functie]_[detail]

Voorbeelden:
sensor.zolder_boiler_temp_boven
sensor.zolder_boiler_temp_onder
binary_sensor.zolder_pomp_vloerverwarming
binary_sensor.bg_pellet_pomp
sensor.bg_pellet_temp_aanvoer
climate.bg_toon
sensor.zolder_cv_aanvoer_temp
```

## 4. Template sensoren (afgeleid)

| Entity | Formule (vereenvoudigd) |
|--------|-------------------------|
| `sensor.actieve_warmtebron` | `if pellet_pomp == on and pellet_aanvoer > 35: 'Pellet' elif toon_burner == on: 'Gas' elif boiler_delta_t < 5: 'Restwarmte' else 'Niets'` |
| `sensor.boiler_voorverwarming_winst` | `boiler_temp_boven - sensor.koud_water_temp` |
| `sensor.pellet_warmtevermogen` | `(pellet_aanvoer - pellet_retour) * debiet * 4180 / 3600` (W) |
| `sensor.gas_besparing_door_pellet` | `runtime_pellet_pomp_today * gemiddeld_vermogen / cv_efficiency` |

## 5. Helpers

| Helper | Type | Doel |
|--------|------|------|
| `input_boolean.modus_zomer` | toggle | Schakelt CV uit, alleen tapwater |
| `input_boolean.boost_badkamer` | toggle | 30 min vloerverwarming forceren |
| `input_number.pellet_drempel` | number | UI‑aanpasbare drempel (default 35) |
| `input_select.warmte_modus` | select | Auto / Pellet first / CV first / Manual |

## 6. Automations (high‑level)

1. **Pellet drempel detectie** — zodra `pellet_aanvoer ≥ input_number.pellet_drempel` → notify + log start.
2. **Boiler dreigt leeg** — `T1 < 40 °C` → notify als modus 'Restwarmte first' aanstaat.
3. **Tapwater piek** — gasverbruik spike > X liter/min in < 60 s → markeer als tapping voor analyse.
4. **Vloerverwarming auto‑off** — pomp aan > 4 uur zonder Toon‑setpoint verandering → uit (veiligheid).
5. **Daily summary** — 23:55 push: gasverbruik, pellet uren, boiler max T, gas‑besparing.

## 7. Datarententie

- Korte termijn (real‑time): standaard recorder, 14 dagen.
- Lange termijn: InfluxDB + Grafana addon óf HA Statistics (Long‑term).
- Pelletverbruik per dag: handmatig via input_number → utility_meter.

## 8. Beveiliging / fail‑safe

- Geen automation forceert pellet of CV in onveilige toestand.
- Watchdog op kritieke MQTT topics (last_seen > 5 min → notify).
- Lokale failover: alle automations werken zonder cloud.
