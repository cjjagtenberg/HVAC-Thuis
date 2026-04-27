# Implementatie-roadmap · HVAC Thuis

Volgorde van uitvoering om het systeem volledig in Home Assistant te integreren. Elk fase-blok bouwt voort op de vorige; volg het in volgorde voor minimale teruggang.

---

## Fase 0 — Voorbereidend (administratie)

| Stap | Wat | Tijd |
|------|-----|------|
| 0.1 | `01-Analyse-systeem.md` doornemen — zorg dat topologie helder is | 30 min |
| 0.2 | `06-HA-entities.md` doornemen — naming-conventies vastpinnen | 15 min |
| 0.3 | Inkoop hardware: **6× DS18B20** watertight + 2 ESP8266/ESP32, OneWire-kabel, 4,7 kΩ pull-up resistor, geleid-pasta, hittebestendige tape | bestelling |
| 0.4 | Backup HA + Toon-config maken (Snapshot) | 5 min |

---

## Fase 1 — Bestaande integraties verifiëren

**Doel**: alle entiteiten die er al zijn/horen te zijn werkend krijgen vóór nieuwe sensoren toevoegen.

| Stap | Wat | Verificatie |
|------|-----|-------------|
| 1.1 | Toon (gehackt) integratie — entiteiten in HA zichtbaar? | check `sensor.cv_aanvoer_temp`, `sensor.cv_modulatie`, `binary_sensor.cv_brander_actief`, `sensor.toon_kamer_temp`, `sensor.toon_setpoint` |
| 1.2 | Naturela controller integratie — pellet-data komt binnen? | check `sensor.pellet_vermogen`, `sensor.pellet_aanvoer_temp`, `binary_sensor.pellet_brander_actief`, `sensor.driewegklep_pomp_temp` |
| 1.3 | Tado integratie — schuur-temp zichtbaar? | check `sensor.tado_schuur_temp` |
| 1.4 | Frank Energie integratie — gasprijs binnen? | check `sensor.frank_energie_gas_huidige_prijs` |
| 1.5 | Indien een van bovenstaande ontbreekt: fix integratie eerst | — |

**Output**: alle bestaande sensoren werken. Voeg ze toe aan een test-Lovelace-card om te bevestigen.

---

## Fase 2 — Nieuwe sensoren installeren

**Doel**: 6 DS18B20-sensoren fysiek monteren en via ESPHome aan HA koppelen.

### 2.1 — ESPHome node 1 (zolder, in meterkast)
- `boiler_temp_boven` (op tank-mantel boven, hoogte ~340 cm vanaf bodem)
- `boiler_temp_onder` (op tank-mantel onder, hoogte ~50 cm vanaf bodem)
- `tap_warm_temp` (op T4-leiding direct na ketel-uitgang)

= **3 sensoren** op 1 OneWire-bus.

ESPHome YAML:
```yaml
esphome:
  name: hvac-zolder
esp8266:
  board: nodemcuv2
wifi:
  ...
api:
  ...
ota:
  ...

one_wire:
  - platform: gpio
    pin: GPIO4

sensor:
  - platform: dallas_temp
    address: 0x...  # boiler boven
    name: "Boiler temp boven"
    id: boiler_top
    update_interval: 30s

  - platform: dallas_temp
    address: 0x...  # boiler onder
    name: "Boiler temp onder"
    id: boiler_bot
    update_interval: 30s

  # ... etc voor verdeler-boven aan/ret + mengkraan + tap-warm
```

### 2.2 — ESPHome node 2 (schuur)
- `vlw_schuur_g1_temp` (groep 1 aanvoer-leiding)
- `vlw_schuur_g2_temp` (groep 2)
- `vlw_schuur_g3_temp` (groep 3)

= **3 sensoren** op 1 OneWire-bus.

### 2.3 — Verifieer alle 6 sensoren in HA
Voor elk: check entiteit komt binnen, waarde plausibel (15-80°C bereik), update_interval ≤ 60s.

### 2.4 — Mengkraan-output via template (geen fysieke sensor)
```yaml
template:
  - sensor:
      - name: "Mengkraan output"
        unique_id: mengkraan_output_calculated
        unit_of_measurement: "°C"
        state: "{{ [states('sensor.boiler_temp_boven')|float, 65]|min }}"
```

---

## Fase 3 — HA configuratie templates + helpers

### 3.1 — `input_number` helpers
```yaml
input_number:
  pellet_prijs_per_ton:
    name: "Pellet prijs per ton"
    min: 200
    max: 700
    step: 5
    unit_of_measurement: "€/ton"
    initial: 425

  pellet_laatste_bestelling_kg:
    name: "Pellet laatste bestelling kg"
    min: 0
    max: 1000
    step: 15
    initial: 240
```

### 3.2 — `input_datetime` helpers
```yaml
input_datetime:
  pellet_laatste_bestelling_datum:
    name: "Pellet laatste bestelling"
    has_date: true
    has_time: false
```

### 3.3 — `utility_meter`
Zie `06-HA-entities.md §13`. Maak meters voor: cv_gas, pellet_kg in dag/maand/jaar cycles.

### 3.4 — Template-sensoren
Zie `06-HA-entities.md §14`. Maak: gas_kosten_dag, pellet_kosten_dag, totaal_kosten_dag, besparing_pellet_dag, voorraad_dagen, stratificatie_dt, cv_dt.

### 3.5 — Counters
Zie `06-HA-entities.md §15`. Maak counters voor brander-uren CV en pellet, met automation-triggers per uur.

---

## Fase 4 — Automatiseringen

| # | Naam | Trigger | Conditie | Actie |
|---|------|---------|----------|-------|
| A1 | Pellet voorraad waarschuwing | voorraad < 30 kg | — | mobile-notification "bestel pellet" |
| A2 | Pellet voorraad kritiek | voorraad < 15 kg | — | persistent_notification + sms |
| A3 | CV brander uur tellen | tijd /1u | brander = on | counter +1 |
| A4 | Pellet brander uur tellen | tijd /1u | pellet brander = on | counter +1 |
| A5 | Boiler-buffer leeg waarschuwing | boiler_temp_boven < 40°C | tap_actief = on (5 min) | melding "buffer leeg, gas-fallback" |
| A6 | Frank Energie gas piek | gas-prijs > €1,80/m³ | — | melding "hoge gasprijs, pellet-prioriteit aan" |
| A7 | Stratificatie verstoord | boiler_dt < 5°C | — | informatieve melding (mogelijk hot-cold-mix) |
| A8 | Schuur te koud | tado_schuur_temp < 8°C | tijd: 06:00-22:00 | pellet aanzetten via Naturela |

---

## Fase 5 — Lovelace dashboard

### 5.1 — Update `05-Lovelace-config.yaml`
Werk de bestaande Lovelace YAML uit zodat alle entities uit `06-HA-entities.md` zichtbaar zijn:

- **Hoofd-card**: het CV-water schema (custom card of HACS picture-elements). Plaats sensor-pillen op de exacte coördinaten zoals in 04c. 
- **Dashboard-card**: replica van het dashboard-blok uit 04c — verbruik per periode, energie-mix grafiek, voorraad+onderhoud, comfort+efficiency, energieprijzen, insights.

### 5.2 — Custom cards/HACS
- **picture-elements** voor het schematische deel (kan SVG als achtergrond gebruiken)
- **apexcharts-card** voor energie-mix bar chart (laatste 7 dagen)
- **mushroom-template-card** voor de live sensor-pillen (compact, kleur-coded)
- **mini-graph-card** voor temperatuur-trend per sensor

### 5.3 — Layout strategie
- Eerste view: "Live HVAC" met het schema + alle sensor-pillen
- Tweede view: "Verbruik & kosten" met dashboard-grid
- Derde view: "Trends" met grafieken (CV-temps over tijd, pellet-vermogen, kosten/dag)
- Vierde view: "Onderhoud" met voorraad, bestelling, onderhoudsbeurten

---

## Fase 6 — Testen & validatie

| # | Wat | Hoe |
|---|-----|-----|
| 6.1 | Sensor-data plausibiliteit | check 24u-trend per sensor, verwacht: aanvoer > retour, boiler boven > onder, etc. |
| 6.2 | Berekende sensoren consistent | gas_kosten_dag = m³ × prijs, sum klopt met meter-aflezing |
| 6.3 | Klep-status correlatie | binary_sensor.driewegklep_naar_systeem ↔ pomp_temp ≥ 35°C? |
| 6.4 | Buffer-werking | bij grote douche-vraag: zie boiler-temp-onder dalen, boven blijven steady, dan na douche herstel |
| 6.5 | Pellet-aandeel klopt | (pellet_kWh / totaal_kWh) realistisch in winter/zomer |
| 6.6 | Automation-triggers | test met fake-waarden (`service: input_number.set_value`) |

---

## Fase 7 — Energy Dashboard (HA Energy)

Configureer HA's ingebouwde Energy Dashboard:
- **Gas verbruik**: `sensor.cv_gas_totaal_m3` als gas-bron, met `sensor.frank_energie_gas_huidige_prijs` als kosten-tracker.
- **Pellet als "ander" bron**: HA Energy heeft geen native pellet-categorie. Gebruik `sensor.pellet_verbruik_totaal_kg` met `state_class: total_increasing`, koppel met `sensor.pellet_kosten_totaal` voor kosten.

---

## Fase 8 — Documentatie & handover

| # | Wat |
|---|-----|
| 8.1 | Update `01-Analyse-systeem.md` met installatie-foto's |
| 8.2 | Maak een korte gebruikershandleiding voor het dashboard |
| 8.3 | Documenteer welke sensoren waar hangen (foto's + ESPHome address-mapping) |
| 8.4 | Reservelijst sensoren + onderdelen voor onderhoud |

---

## Tijdsinschatting totaal

| Fase | Tijd |
|------|------|
| Fase 0 — voorbereidend | 1 uur (excl. levertijd hardware) |
| Fase 1 — bestaande verifiëren | 1-2 uur |
| Fase 2 — sensoren installeren | 1 dag (incl. test) |
| Fase 3 — HA config templates | 2-3 uur |
| Fase 4 — automatiseringen | 2 uur |
| Fase 5 — Lovelace dashboard | 4-6 uur (mooi maken) |
| Fase 6 — testen | 1-2 weken passief monitoren |
| Fase 7 — Energy Dashboard | 30 min |
| Fase 8 — documentatie | 1-2 uur |
| **Totaal actieve werk** | ~2-3 dagen verdeeld over enkele weekenden |

---

## Risk-log

| Risico | Mitigatie |
|--------|-----------|
| Toon-integratie breekt na firmware-update | Vergrendel firmware-versie of gebruik OpenTherm-gateway zonder Toon-tussenlaag |
| DS18B20 oppervlakte-meting onnauwkeurig (±2°C) | Trends zijn betrouwbaarder dan absolute waarden; gebruik mantel + isolatie |
| Pellet-voorraad-counter drift | Periodiek (maandelijks) handmatig calibreren tegen werkelijke voorraad |
| Frank Energie API-rate-limits | Cache prijzen lokaal, update elke 15 min ipv real-time |
| ESPHome OneWire-bus instabiliteit met >5 sensoren | Splits over 2 nodes (zolder + schuur) |

---

## Status (2026-04-27)

- [x] Topologie vastgesteld (`01-Analyse-systeem.md`)
- [x] Visualisatie compleet (`04c-CV-water-layout.html`)
- [x] Entities-mapping gedocumenteerd (`06-HA-entities.md`)
- [x] Implementatie-roadmap (dit document)
- [ ] **Fase 0-1** — bestaande integraties verifiëren
- [ ] **Fase 2** — DS18B20-sensoren installeren
- [ ] **Fase 3** — HA templates en utility meters
- [ ] **Fase 4** — automatiseringen
- [ ] **Fase 5** — Lovelace dashboard finaliseren
- [ ] **Fase 6** — testen
- [ ] **Fase 7** — Energy Dashboard
- [ ] **Fase 8** — documentatie & handover
