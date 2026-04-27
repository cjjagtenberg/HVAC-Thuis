# HVAC Thuis · projectoverzicht

Volledige HA-integratie voor het verwarmingssysteem: CV ketel (Remeha Avanta 25 KW CW4) + Restwarmte boiler (200 L) + Pellet-CV (Naturela) + VLW Schuur + dynamische gas-tarieven (Frank Energie). Met sensor-pillen, kostenberekening, voorraad-tracking en insights.

---

## Bestanden in dit project

| # | Bestand | Beschrijving | Status |
|---|---------|--------------|--------|
| 00 | **README.md** *(dit bestand)* | Projectoverzicht, deployment-volgorde | — |
| 01 | **01-Analyse-systeem.md** | Canonical topologie, componentenlijst, hydraulische schema's, decisions log (D1-D12) | ✓ finaal |
| 04c | **04c-CV-water-layout.html** | Visualisatie: SVG-schema + dashboard met live data + verbruik tegels | ✓ finaal |
| 06 | **06-HA-entities.md** | Entity-mapping per component, naming-conventies, integratie-bronnen | ✓ finaal |
| 07 | **07-Implementatie-roadmap.md** | 8-fasen uitvoeringsplan, hardware-inkoop, ESPHome-config, validatie | ✓ finaal |
| 08 | **08-Lovelace-HVAC-tab.yaml** | Volledige Lovelace view voor Future dashboard (HACS-versie) | ✓ klaar voor deployment |
| 08b | **08b-Lovelace-HVAC-tab-minimal.yaml** | Minimal Lovelace zonder HACS-dependencies (fallback) | ✓ alternatief |
| 09 | **09-configuration-snippets.yaml** | Backend: utility_meters, templates, automations, helpers | ✓ klaar voor deployment |

Oude bestanden die mogelijk geïntegreerd of gearchiveerd kunnen worden:
- `04-Dashboard-concept.html` — eerder concept, vervangen door 04c
- `04b-Schema-v2.html` — eerder schema, vervangen door 04c
- `05-Lovelace-config.yaml` — eerdere YAML, vervangen door 08

---

## Deployment-volgorde (voor jou als teamleider)

### Fase A — Voorbereiding (lezen + verifieren)
1. **01-Analyse-systeem.md** doornemen — zorg dat de topologie in je hoofd zit
2. **04c-CV-water-layout.html** openen in browser — visuele referentie
3. **06-HA-entities.md** doornemen — naming-conventies onthouden

### Fase B — Bestaande HA-integraties verifiëren
Open Developer Tools → Toestanden → check:
- `sensor.cv_*` van Toon-integratie
- `sensor.toon_*` van Toon-thermostaat
- `sensor.pellet_*` of `sensor.naturela_*` van pellet-controller
- `sensor.tado_schuur_*` van Tado
- `sensor.frank_energie_*` van Frank Energie

→ noteer afwijkende entity-IDs (search&replace later in de YAML's).

### Fase C — Hardware (DS18B20 sensoren)
4 nieuwe sensoren via 1-2 ESPHome-nodes (boiler al via Tuya):
- **Zolder** (1): tap_warm_temp
- **Schuur** (3): vlw_schuur_g1, g2, g3

Zie **07-Implementatie-roadmap.md §2** voor ESPHome YAML-templates.

### Fase D — HA backend-config deployen
1. Plak `09-configuration-snippets.yaml` in je `configuration.yaml`
2. Check Configuration → Restart HA
3. Helpers via UI: Settings → Devices & services → Helpers (zie §2-3 in 09)

### Fase E — Lovelace dashboard
**HACS-versie (aanbevolen)** als je mushroom-cards, apexcharts-card en mini-graph-card hebt:
- Open Future dashboard → ⋮ → Edit → + Add view → Raw configuration → plak `08-Lovelace-HVAC-tab.yaml`

**Minimal versie** als HACS niet beschikbaar:
- Idem maar plak `08b-Lovelace-HVAC-tab-minimal.yaml`

### Fase F — Testen & finetuning
- Een week monitoren: zijn alle waarden plausibel?
- Brander-uren tellen op?
- Voorraad-tracking klopt met werkelijke pellet-aankoop?
- Frank Energie-prijzen update correct?

---

## Architectuur in één diagram

```
┌─────────────────────────────────────────────────────────────┐
│            ZOLDER                                            │
│                                                              │
│   ┌──KW NET                  ┌──CV KETEL──┐                 │
│   │   │                      │ Remeha     │   sensor pills  │
│   │   ▼                      │ Avanta 25  │   ──→ live data │
│   │  ┌──MENGKRAAN──┐         │ CW4        │                 │
│   │  │ ≤ 65°C       │        └─┬─┬─┬─┬────┘                 │
│   │  └──┬──┬──┬─────┘          tap-koud-in (T3)             │
│   │     │  │  └─→ T3 → ─────────────┘                       │
│   │     │  └─ T2 ← ──┐  ┌──BOILER 200L──┐                   │
│   │     │            └──┤ tapwater + CV-├── retour-IN spiraal│
│   │     └─T1b cold      │ spiraal       ├── retour-UIT      │
│   │                     └───────────────┘                    │
│   └──→ T1 → tap-koud-in boiler                              │
│                                                              │
├──── etage ──────────────────────────────────────────────────┤
│            BG / KRUIPRUIMTE                  (verdeler BG)  │
│            oneway valve  ●                                   │
│            inject-punt   ●←── pellet-injectie 11             │
│                          │                                   │
│            tap-pellet-OUT ──→ pad 8 (apart van pad 11)       │
│                                                              │
├──── schuur ─────────────────────────────────────────────────┤
│   ┌──PELLET CV──┐   ┌──VLW──┐   ┌─3-wegklep─┐               │
│   │ Naturela    │   │ G1    │   │ T > 35°C  │               │
│   │ 22 kW       │   │ G2    │   │ → BG      │               │
│   │ ● live data │   │ G3    │   │           │               │
│   └─────────────┘   └───────┘   └───────────┘               │
│                                       │                      │
│                                       └─bypass→ pad 8 retour│
└─────────────────────────────────────────────────────────────┘
```

---

## Belangrijkste design-keuzes (zie 01-Analyse-systeem.md §5 voor details)

| # | Keuze | Waarom |
|---|-------|--------|
| D1-2 | Boiler in CV-retour, CV-water DOOR de spiraal | Restwarmte-recovery: ketel krijgt voorverwarmd retour |
| D3 | Mengkraan ≤ 65°C beveiliging | Veiligheid douche + bescherming combiketel-wisselaar |
| D5-6 | CV ketel links, pad 8 + 11 op aparte verticalen (x=460/x=500) | Twee verschillende fluids = twee aparte pijpen, technisch correct |
| D7 | Bypass solid line naar pellet-retour | Geen vage stippellijn, duidelijke verbinding |
| D11-12 | Verdeler-temps + mengkraan-output sensoren GESCHRAPT | Redundant (verdeler-temps), mechanisch fixed (mengkraan) |
| D8 | Live temps IN componenten ipv aparte pillen | Compact, geen pijp-overlap |
| D10 | Component-data IN schema, aggregate data ONDER schema | Scheidt technische van financiële context |

---

## Sensoren-overzicht

**Bestaand** (uit integraties):
- Toon → CV ketel modulatie/aanvoer/retour/brander/gas + huiskamer-temp + setpoint
- Naturela → pellet vermogen/aanvoer/retour/brander/voorraad + 3-wegklep pomp-temp
- Tado → schuur-ambient
- Frank Energie → dynamische gas + stroom tarieven

**Reeds beschikbaar via Tuya:**
- `sensor.temp_boiler_top_temperatuur` (boven)
- `sensor.temp_boiler_bottom_temperatuur` (onder)

**Nieuw te installeren** (4× DS18B20):
- VLW Schuur G1, G2, G3 (per groep aanvoer-temp)
- Tap-warm bij douche (toont buffer-leeg-effect)

**Geschrapt**:
- Verdeler-temps × 4 (redundant met ketel-temps)
- Mengkraan-output (mechanisch fixed, template-berekend)

---

## HA Entiteiten — naming-conventies

```
sensor.cv_*               CV ketel
sensor.boiler_*           restwarmte boiler
sensor.pellet_*           pellet-CV
sensor.toon_*             huiskamer thermostaat
sensor.tado_*             schuur ambient
sensor.frank_energie_*    dynamische tarieven
sensor.vlw_schuur_*       VLW schuur groepen
sensor.driewegklep_*      3-wegklep status
sensor.tap_warm_*         tap-water bij douche
sensor.gas_kosten_*       afgeleid uit utility_meter × tarief
sensor.pellet_kosten_*    afgeleid uit utility_meter × pellet-prijs
sensor.totaal_kosten_*    gas + pellet
sensor.besparing_pellet_* template berekening
binary_sensor.*_actief    aan/uit-statussen
counter.*_uren            brander-uren counters
input_number.pellet_*     handmatige helpers
input_datetime.*_datum    datums (bestelling, onderhoud)
```

---

## Energie-equivalenten (gebruikt in templates)

- 1 m³ Slochteren-gas = **9,77 kWh**
- 1 kg pellets klasse A1 = **4,8 kWh**
- 1 kg pellet ≈ **0,49 m³ gas-equivalent**

---

## Tijdsinschatting totaal (uit 07-roadmap)

| Fase | Tijd |
|------|------|
| Voorbereidend + integraties verifiëren | 2-3 uur |
| 6× DS18B20 sensoren installeren | 1 dag (incl. test) |
| HA backend-config + helpers | 2-3 uur |
| Automatiseringen | 2 uur |
| Lovelace dashboard | 4-6 uur (mooi maken) |
| Testen passief monitoren | 1-2 weken |
| Documentatie & handover | 1-2 uur |
| **Totaal actieve werk** | ~2-3 dagen verdeeld over weekenden |

---

## Volgende stappen (status 2026-04-27)

- [x] Topologie vastgesteld en gevalideerd
- [x] Visualisatie compleet (04c)
- [x] Entity-mapping (06)
- [x] Implementatie-roadmap (07)
- [x] Lovelace YAML — HACS-versie (08)
- [x] Lovelace YAML — minimal versie (08b)
- [x] Backend config snippets (09)
- [x] Project-README (00)
- [ ] **HARDWARE INSTALLATIE** — 6 DS18B20 plaatsen
- [ ] **HA BACKEND DEPLOY** — configuration.yaml + restart
- [ ] **LOVELACE DEPLOY** — Future dashboard tabblad toevoegen
- [ ] **VALIDATIE** — een week monitoren

---

*Stand 2026-04-27. Alle planning + design klaar. Implementatie wacht op uitvoeringswerk in HA + hardware-installatie.*
