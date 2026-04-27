# 03 — Entities, sensoren en helpers

Volledige inventaris van alles wat in HA aanwezig moet zijn voor het dashboard.

## 1. Bestaande entities (zoals aangegeven door Cor)

| Entity | Bron | Beschrijving |
|--------|------|--------------|
| `sensor.zolder_boiler_temp_boven` | ESPHome / 1‑Wire | T1 — bovenste sensor restwarmte‑boiler |
| `sensor.zolder_boiler_temp_onder` | ESPHome / 1‑Wire | T2 — onderste sensor restwarmte‑boiler |

## 2. Nog te realiseren entities (voorgesteld)

### 2.1 Toon
| Entity | Type | Bron | Status |
|--------|------|------|--------|
| `climate.toon_woonkamer` | climate | Toon‑MQTT | nog te checken |
| `sensor.toon_huidige_temperatuur` | sensor | Toon‑MQTT | nog te checken |
| `sensor.toon_setpoint` | sensor | Toon‑MQTT | nog te checken |
| `binary_sensor.toon_brander_actief` | binary_sensor | Toon‑MQTT | **kritiek** — vlam status |
| `sensor.toon_modulatie_niveau` | sensor (%) | Toon‑MQTT | optioneel |
| `sensor.toon_gasverbruik_dag` | sensor (m³) | Toon‑MQTT | optioneel — alt: P1 |

### 2.2 Pellet (Naturela)
| Entity | Type | Bron |
|--------|------|------|
| `sensor.pellet_temp_aanvoer` | sensor | Naturela MQTT/Modbus |
| `sensor.pellet_temp_retour` | sensor | Naturela |
| `sensor.pellet_temp_kachel` | sensor | Naturela |
| `binary_sensor.pellet_pomp_actief` | binary_sensor | Naturela |
| `binary_sensor.pellet_3weg_klep_open` | binary_sensor | **Naturela (echte feedback)** — niet als proxy |
| `sensor.pellet_voorraad_kg` | sensor | Naturela (interne weegregeling/teller) |
| `sensor.pellet_verbruik_dag_kg` | sensor (state_class: total_increasing) | **Naturela** — direct uitleesbaar in kg per dag |
| `sensor.pellet_verbruik_totaal_kg` | sensor | Naturela — cumulatief lifetime |
| `sensor.pellet_brand_status` | sensor | Naturela (idle / ontsteken / branden / uit) |

### 2.3 CV‑ketel
| Entity | Type | Bron |
|--------|------|------|
| `sensor.cv_aanvoer_temp` | sensor | DS18B20 op aanvoerleiding |
| `sensor.cv_retour_temp` | sensor | DS18B20 op retour |
| `binary_sensor.cv_pomp_hoofdcircuit` | binary_sensor | Shelly Plug S of OpenTherm |
| `sensor.cv_modulatie` | sensor | OpenTherm gateway (optioneel) |

### 2.4 Vloerverwarming (twee zones — beide HA‑bestuurbaar)

#### Badkamer
| Entity | Type | Bron |
|--------|------|------|
| `switch.vloerverwarming_badkamer` | **switch** (lezen + schakelen) | Shelly Plug S / relay |
| `sensor.vloerverwarming_badkamer_power` | sensor (W) | Shelly Plug S |
| `sensor.vloerverwarming_badkamer_aanvoer` | sensor | DS18B20 verdeler |
| `sensor.badkamer_temperatuur` | sensor | losse Aqara/ESPHome sensor |
| `sensor.vloerverwarming_badkamer_runtime_dag` | utility_meter | template op switch‑on tijd |

#### Schuur (passieve restwarmte‑afnemer)
| Entity | Type | Bron |
|--------|------|------|
| `switch.vloerverwarming_schuur` | **switch** (pomp‑relay, met dependency‑guard) | Shelly / relay |
| `sensor.vloerverwarming_schuur_power` | sensor (W) | Shelly EM/Plug |
| `climate.schuur_tado` | climate | **Tado V3+** — bestaat in HA via Tado integratie. Wordt **niet** gebruikt om te sturen — `preset_mode: 'off'` of "Smart Schedule uit". |
| `sensor.schuur_temperatuur` | sensor | Afgeleid van `state_attr('climate.schuur_tado','current_temperature')` of native Tado temperatuur‑sensor |
| `sensor.schuur_luchtvochtigheid` | sensor | Tado interne hygrometer (mooi meegenomen) |
| `binary_sensor.schuur_warmte_beschikbaar` | template binary_sensor | klep open AND (pellet > 35° OR cv_vlam) |
| `binary_sensor.schuur_via_cv` | template binary_sensor | klep open AND cv_vlam AND pellet brandt niet (= **risico flag**) |
| `sensor.schuur_warmte_bron` | template sensor | "Pellet" / "Pellet (afkoelend)" / "CV (ongewenst!)" / "Geen" |
| `sensor.vloerverwarming_schuur_runtime_dag` | utility_meter | template op switch‑on tijd |

**Belangrijk — Tado configuratie:**

- Tado V3+ thermostaat hangt in de schuur, alleen voor **uitlezing** (temperatuur + luchtvochtigheid).
- In Tado app: **Smart Schedule UIT** of **Manual Heating: off** zodat de Tado niets probeert aan te sturen (er hangt geen ventiel achter).
- In HA: `climate.schuur_tado` wel zichtbaar maken op dashboard, maar niet gebruiken in automations.
- Optioneel een template sensor `sensor.schuur_temperatuur` als "schone" temperatuur‑bron:

```yaml
template:
  - sensor:
      - name: "Schuur temperatuur"
        unique_id: schuur_temp_clean
        unit_of_measurement: "°C"
        device_class: temperature
        state_class: measurement
        state: "{{ state_attr('climate.schuur_tado','current_temperature') }}"
      - name: "Schuur luchtvochtigheid"
        unique_id: schuur_humidity_clean
        unit_of_measurement: "%"
        device_class: humidity
        state_class: measurement
        state: "{{ state_attr('climate.schuur_tado','current_humidity') }}"
```

**Templates — schuur bron en risico‑detectie:**

```yaml
template:
  # 1) Is er überhaupt warmte beschikbaar voor schuur?
  - binary_sensor:
      - name: "Schuur warmte beschikbaar"
        unique_id: hvac_schuur_warmte_beschikbaar
        state: >
          {% set klep = is_state('binary_sensor.pellet_3weg_klep_open','on') %}
          {% set pellet_warm = states('sensor.pellet_temp_aanvoer')|float(0) > 35 %}
          {% set cv_vlam = is_state('binary_sensor.toon_brander_actief','on') %}
          {{ klep and (pellet_warm or cv_vlam) }}

  # 2) RISICO: schuur wordt momenteel via CV (gas) verwarmd ipv pellet
  #    Treedt op tijdens nablusfase (pellet uit, water nog warm, klep open, CV brandt)
      - name: "Schuur via CV (risico)"
        unique_id: hvac_schuur_via_cv
        state: >
          {% set klep = is_state('binary_sensor.pellet_3weg_klep_open','on') %}
          {% set pellet_brand = is_state('sensor.pellet_brand_status','branden') %}
          {% set cv_vlam = is_state('binary_sensor.toon_brander_actief','on') %}
          {{ klep and cv_vlam and not pellet_brand }}
        icon: mdi:alert-octagon
        attributes:
          beschrijving: >
            Klep open + CV brandt + pellet niet actief.
            Schuur wordt mogelijk via aardgas verwarmd.

  # 3) Tekstuele bron-aanduiding voor dashboard
  - sensor:
      - name: "Schuur warmte bron"
        unique_id: hvac_schuur_bron
        state: >
          {% set klep = is_state('binary_sensor.pellet_3weg_klep_open','on') %}
          {% set pellet_brand = is_state('sensor.pellet_brand_status','branden') %}
          {% set pellet_warm = states('sensor.pellet_temp_aanvoer')|float(0) > 35 %}
          {% set cv_vlam = is_state('binary_sensor.toon_brander_actief','on') %}
          {% if not klep %} Geen
          {% elif pellet_brand %} Pellet
          {% elif pellet_warm and not cv_vlam %} Pellet (afkoelend)
          {% elif cv_vlam %} CV (ongewenst!)
          {% else %} Klep open · onbekend
          {% endif %}
```

**Hard‑mode interlock automation (optioneel — schuur strikt alleen op pellet):**

```yaml
automation:
  - alias: "Schuur uit als bron CV wordt"
    description: "Strikt: schuur mag alleen op pellet-restwarmte draaien, niet op gas."
    mode: single
    trigger:
      - platform: state
        entity_id: binary_sensor.schuur_via_cv
        to: "on"
        for: "00:01:00"   # korte tolerance om flikkers te voorkomen
    condition:
      - condition: state
        entity_id: switch.vloerverwarming_schuur
        state: "on"
      - condition: state
        entity_id: input_boolean.schuur_strict_pellet_only
        state: "on"
    action:
      - service: switch.turn_off
        target: { entity_id: switch.vloerverwarming_schuur }
      - service: notify.persistent_notification
        data:
          title: "Schuur uit (CV‑risico)"
          message: "Bron is naar CV geswitcht — strict mode actief, schuur uit gezet."

  - alias: "Schuur warning bij CV-bron (soft mode)"
    description: "Soft: alleen melden, niet ingrijpen."
    trigger:
      - platform: state
        entity_id: binary_sensor.schuur_via_cv
        to: "on"
        for: "00:02:00"
    condition:
      - condition: state
        entity_id: switch.vloerverwarming_schuur
        state: "on"
      - condition: state
        entity_id: input_boolean.schuur_strict_pellet_only
        state: "off"
    action:
      - service: notify.mobile_app
        data:
          title: "Schuur op gas?"
          message: "Klep open + CV brandt + pellet uit. Schuur wordt mogelijk via gas verwarmd."
```

Bijbehorende helper voor de mode‑keuze:

```yaml
input_boolean:
  schuur_strict_pellet_only:
    name: "Schuur alleen pellet (strict)"
    icon: mdi:shield-lock
```

**Schuur dependency‑guard automation:**

```yaml
automation:
  - alias: "Schuur pomp uit als geen warmte beschikbaar"
    description: "Voorkomt rondpompen van koud water als pellet niet draait of klep dicht."
    trigger:
      - platform: state
        entity_id: binary_sensor.schuur_warmte_beschikbaar
        to: "off"
        for: "00:02:00"   # 2 min tolerance bij overgangen
    condition:
      - condition: state
        entity_id: switch.vloerverwarming_schuur
        state: "on"
    action:
      - service: switch.turn_off
        target: { entity_id: switch.vloerverwarming_schuur }
      - service: notify.persistent_notification
        data:
          title: "Schuur pomp uit"
          message: "Pellet niet beschikbaar (pomp/klep) — schuur pomp veilig uit gezet."

  - alias: "Schuur pomp blokkeren bij inschakelen zonder warmte"
    description: "Als gebruiker de switch aanzet maar er is geen warmte → direct terug uit + melding."
    trigger:
      - platform: state
        entity_id: switch.vloerverwarming_schuur
        to: "on"
    condition:
      - condition: state
        entity_id: binary_sensor.schuur_warmte_beschikbaar
        state: "off"
    action:
      - service: switch.turn_off
        target: { entity_id: switch.vloerverwarming_schuur }
      - service: notify.persistent_notification
        data:
          title: "Schuur niet ingeschakeld"
          message: "Pellet draait niet of klep dicht — schuur kan nu geen warmte krijgen."
```

**Let op:** beide pompen / circuits zijn **bestuurbaar vanuit HA**. Dit maakt eigen automations mogelijk:

**Let op:** pomp is **bestuurbaar vanuit HA**. Dit maakt eigen automations mogelijk:

```yaml
# Voorbeeld: badkamer thermostaat-light
automation:
  - alias: "Vloerverw. badkamer auto"
    trigger:
      - platform: numeric_state
        entity_id: sensor.badkamer_temperatuur
        below: 21
    condition:
      - condition: state
        entity_id: input_boolean.hvac_modus_zomer
        state: "off"
    action:
      - service: switch.turn_on
        target: { entity_id: switch.vloerverwarming_badkamer }

  - alias: "Vloerverw. badkamer uit bij target"
    trigger:
      - platform: numeric_state
        entity_id: sensor.badkamer_temperatuur
        above: 23.5
    action:
      - service: switch.turn_off
        target: { entity_id: switch.vloerverwarming_badkamer }

  - alias: "Vloerverw. boost timer"
    trigger:
      - platform: state
        entity_id: input_boolean.hvac_boost_badkamer
        to: "on"
    action:
      - service: switch.turn_on
        target: { entity_id: switch.vloerverwarming_badkamer }
      - delay: "00:30:00"
      - service: switch.turn_off
        target: { entity_id: switch.vloerverwarming_badkamer }
      - service: input_boolean.turn_off
        target: { entity_id: input_boolean.hvac_boost_badkamer }
```

### 2.5 Energie en kosten
| Entity | Type | Bron |
|--------|------|------|
| `sensor.gas_verbruik_actueel` | sensor (m³/h) | DSMR P1 |
| `sensor.gas_verbruik_meter` | sensor (m³, total_increasing) | DSMR P1 — meterstand |
| `sensor.gas_verbruik_dag` | sensor (m³, utility_meter) | utility_meter daily reset |
| `sensor.pellet_verbruik_dag_kg` | sensor (kg) | **Naturela** — direct |
| `sensor.elektra_pompen_actueel` | sensor (W) | Shelly EM |
| `sensor.frank_energie_gas_actueel` | sensor (€/m³) | **Frank Energie integratie** (HACS) |
| `input_number.pellet_prijs_per_kg` | input_number | handmatig (€/kg) |
| `sensor.kosten_gas_dag` | template sensor (€) | gas_dag × frank_gasprijs |
| `sensor.kosten_pellet_dag` | template sensor (€) | pellet_dag × pelletprijs |
| `sensor.kosten_totaal_dag` | template sensor (€) | gas + pellet |
| `sensor.besparing_pellet_dag` | template sensor (€) | hypothetisch alleen‑gas − werkelijk |

**Verbruik & kosten templates:**

```yaml
template:
  - sensor:
      # Verbruik samenvatting
      - name: "Verbruik vandaag samenvatting"
        unique_id: hvac_verbruik_vandaag
        state: >
          {{ states('sensor.gas_verbruik_dag') }} m³ gas ·
          {{ states('sensor.pellet_verbruik_dag_kg') }} kg pellet

      # Kosten gas vandaag
      - name: "Kosten gas dag"
        unique_id: hvac_kosten_gas_dag
        unit_of_measurement: "€"
        device_class: monetary
        state_class: total
        state: >
          {% set m3 = states('sensor.gas_verbruik_dag')|float(0) %}
          {% set prijs = states('sensor.frank_energie_gas_actueel')|float(1.45) %}
          {{ (m3 * prijs) | round(2) }}

      # Kosten pellet vandaag
      - name: "Kosten pellet dag"
        unique_id: hvac_kosten_pellet_dag
        unit_of_measurement: "€"
        device_class: monetary
        state_class: total
        state: >
          {% set kg = states('sensor.pellet_verbruik_dag_kg')|float(0) %}
          {% set prijs = states('input_number.pellet_prijs_per_kg')|float(0.45) %}
          {{ (kg * prijs) | round(2) }}

      # Totale kosten vandaag
      - name: "Kosten totaal dag"
        unique_id: hvac_kosten_totaal_dag
        unit_of_measurement: "€"
        device_class: monetary
        state_class: total
        state: >
          {{ (states('sensor.kosten_gas_dag')|float(0)
            + states('sensor.kosten_pellet_dag')|float(0)) | round(2) }}

      # Besparing door pellet (vergelijk met scenario alles-op-gas)
      # 1 kg pellet ≈ 4,8 kWh ≈ 0,52 m³ aardgas (energie-equivalent)
      - name: "Besparing pellet dag"
        unique_id: hvac_besparing_pellet_dag
        unit_of_measurement: "€"
        device_class: monetary
        state_class: total
        state: >
          {% set pellet_kg = states('sensor.pellet_verbruik_dag_kg')|float(0) %}
          {% set gas_prijs = states('sensor.frank_energie_gas_actueel')|float(1.45) %}
          {% set gas_equiv = pellet_kg * 0.52 %}
          {% set hypothetisch_gas = states('sensor.kosten_gas_dag')|float(0) + (gas_equiv * gas_prijs) %}
          {% set werkelijk = states('sensor.kosten_totaal_dag')|float(0) %}
          {{ (hypothetisch_gas - werkelijk) | round(2) }}
```

**Helper voor pelletprijs:**

```yaml
input_number:
  pellet_prijs_per_kg:
    name: "Pellet prijs per kg"
    min: 0.20
    max: 1.50
    step: 0.01
    unit_of_measurement: "€"
    icon: mdi:cash
    initial: 0.45        # pas aan op je laatste inkoopprijs
```

**Frank Energie integratie installatie:**

1. HACS → Custom integrations → zoek "Frank Energie"
2. Repo: `frank-energie/frank_energie_homeassistant`
3. Levert sensoren zoals `sensor.frank_energie_gas` (huidige uurprijs) en `sensor.frank_energie_gas_today_avg`.
4. Gebruik de "all‑in" (incl. belasting + ODE + BTW) sensor in de kostenberekening.

## 3. Template sensoren

```yaml
# packages/hvac_templates.yaml
template:
  - sensor:
      - name: "Boiler Delta T"
        unique_id: hvac_boiler_delta_t
        unit_of_measurement: "°C"
        state: >
          {{ (states('sensor.zolder_boiler_temp_boven') | float
              - states('sensor.zolder_boiler_temp_onder') | float) | round(1) }}

      - name: "Boiler Laadstatus"
        unique_id: hvac_boiler_laad
        state: >
          {% set t1 = states('sensor.zolder_boiler_temp_boven') | float %}
          {% if t1 >= 60 %} Volledig geladen
          {% elif t1 >= 50 %} Goed
          {% elif t1 >= 40 %} Halfvol
          {% elif t1 >= 30 %} Bijna leeg
          {% else %} Leeg
          {% endif %}

      - name: "Actieve Warmtebron"
        unique_id: hvac_actieve_bron
        state: >
          {% set pellet = is_state('binary_sensor.pellet_pomp_actief','on') %}
          {% set pellet_t = states('sensor.pellet_temp_aanvoer') | float(0) %}
          {% set drempel = states('input_number.pellet_drempel') | float(35) %}
          {% set gas = is_state('binary_sensor.toon_brander_actief','on') %}
          {% if pellet and pellet_t > drempel %} Pellet
          {% elif gas %} Gas (CV)
          {% elif states('sensor.boiler_delta_t')|float > 8 %} Restwarmte
          {% else %} Geen
          {% endif %}

      - name: "Pellet Vermogen"
        unique_id: hvac_pellet_vermogen
        unit_of_measurement: "kW"
        state: >
          {% set ta = states('sensor.pellet_temp_aanvoer') | float(0) %}
          {% set tr = states('sensor.pellet_temp_retour') | float(0) %}
          {% set debiet = 0.25 %}  {# l/s — kalibreren #}
          {% if ta > tr %}
            {{ ((ta - tr) * debiet * 4.18) | round(2) }}
          {% else %} 0
          {% endif %}
```

## 4. Helpers (UI configureerbaar)

```yaml
input_boolean:
  hvac_modus_zomer:
    name: "Zomermodus (alleen tapwater)"
    icon: mdi:weather-sunny

  hvac_boost_badkamer:
    name: "Badkamer boost (30 min)"
    icon: mdi:shower-head

  # Geen schuur boost meer — schuur is geen geregelde zone

input_number:
  pellet_drempel:
    name: "Pellet 3-weg klep drempel"
    min: 25
    max: 50
    step: 1
    unit_of_measurement: "°C"
    icon: mdi:thermometer

input_select:
  warmte_modus:
    name: "Warmte strategie"
    options:
      - "Auto"
      - "Pellet voorrang"
      - "CV voorrang"
      - "Handmatig"
    icon: mdi:swap-horizontal
```

## 5. Utility meters

```yaml
utility_meter:
  # Pellet verbruik dag — alleen nodig als Naturela GEEN dagteller geeft.
  # Naturela levert sensor.pellet_verbruik_dag_kg direct, dus deze is meestal overbodig.
  # pellet_dag:
  #   source: sensor.pellet_verbruik_totaal_kg
  #   cycle: daily
  #   name: "Pellet verbruik dag (afgeleid)"

  gas_dag:
    source: sensor.gas_verbruik_meter
    cycle: daily
    name: "Gas verbruik dag"
```

## 6. Statistieken/long term

Aanbevolen attributen `state_class: measurement` op alle temperatuur‑sensoren en `state_class: total_increasing` op gas/pellet meters voor de Energie‑dashboard‑integratie.
