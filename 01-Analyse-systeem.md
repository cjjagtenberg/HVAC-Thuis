# Analyse — HVAC Thuis · finale topologie

Dit is de canonical beschrijving van het systeem, vastgelegd nadat 04c-CV-water-layout volledig gevalideerd is met de gebruiker. Alle ontwerpkeuzes en hydraulische routings zijn hier verankerd.

---

## 1. Componenten-overzicht

| Component | Type | Locatie | Specificatie | Aansluitingen |
|-----------|------|---------|--------------|---------------|
| **CV ketel** | Remeha Avanta 25 KW CW4 | zolder | combiketel · 6,2 – 26,7 kW · CW4 · 7,5 L/min @ 60°C · OpenTherm · modulatie 23–100% | onderzijde 4 nippels: tap-koud, CV-aanv, CV-retour, tap-warm |
| **Restwarmte boiler** | indirect-fired tank | zolder, onder ketel | 200 L tapwater in tank · CV-spiraal er DOOR (warmtewisselaar) · 2× temp-sensor (boven + onder) | links: tapwater koud-in (onder), warm-uit (boven) · rechts: CV-spiraal in (onder), uit (boven) |
| **Mengkraan** | thermostatische 3-poorts | zolder, links naast boiler | beveiligt op ≤ 65°C · live output ~56°C | warm-in van boiler · koud-in van KW-net (T1b aftakking) · output naar CV ketel tap-koud |
| **Verdeler boven** | manifold | zolder | aanv-rail + retour-rail · voedt afnemers boven | aanvoer + retour links |
| **Afnemers boven** | radiatoren + badkamer-VV (eigen pomp) | bovenverdieping | thermostaatkranen op radiatoren | risers vanaf verdeler boven |
| **Verdeler BG** | manifold | kruipruimte | aanv + retour · voedt benedenradiatoren | aanvoer + retour links |
| **Radiatoren BG** | benedenverdieping | woonkamer/keuken etc. | géén thermostaatkraan — Toon-thermostaat-geregeld | risers vanaf verdeler BG |
| **Pellet-CV** | Naturela | schuur | 22 kW totaal: 4 kW stralingswarmte + 18 kW CV-water | aanvoer-uit (warm) top-rechts · retour-in (koud) bottom-rechts |
| **VLW Schuur** | manifold + 3 groepen vloerverwarming | schuur | aanv + retour rails · groep G1/G2/G3 | aanvoer links · retour rechts → 3-wegklep |
| **3-wegklep (gemotoriseerd)** | thermostatisch | schuur, naast VLW | schakelt op pomp-temp ≥ 35°C · live ~42°C | input van VLW-retour · output1 (warm) → CV-aanv injectie · output2 (bypass) → pellet-retour |
| **Oneway valve** | check valve | BG-aanvoer-leiding | mechanisch — voorkomt terugloop pellet-water richting CV ketel | inline op CV-aanvoer-leiding (x=425, y=372 in schema) |
| **Toon thermostaat** | gehackt, OpenTherm gateway | huiskamer | huiskamer-temp-meting + setpoint + CV-vraag aansturen | OpenTherm-bus naar Remeha |
| **Tado** | ambient temp sensor | schuur | luchttemperatuur-meting | Tado app/integration |
| **KW-net** | stadswater | inkomst | gegeven druk, gegeven temp ~10°C | hoofd → tap-koud-in boiler + aftak T1b → mengkraan koud-in |
| **Douchekop** | tap-afname | badkamer | tap-warm verbruik | tap-warm-uit ketel |

---

## 2. Hydraulische topologie

### 2.1 CV-circuit (gesloten lus)

```
CV ketel CV-aanvoer ─┬──> Verdeler boven aanv ──> afnemers boven
                     │
                     └──> [oneway valve] ──> Verdeler BG aanv ──> radiatoren BG
                          │                                          │
                          └─ tap pellet retour ↑                     │
                          └─ pellet-injectie inkomst ↑               │
                                                                     │
Verdeler boven retour ─┐                                             │
                       ├──> Boiler-spiraal IN ──> [door spiraal]     │
Verdeler BG retour ────┘                                             │
                                                       │             │
                                                       v             │
                                               Boiler-spiraal UIT    │
                                                       │             │
                                                       └──> CV ketel CV-retour
```

**Topologie-details:**
- Hoofdleiding gaat onderuit de ketel naar BG (via T-stuk op zolder).
- T-stuk splitst af naar verdeler boven (parallel) en BG verdeler (verticaal omlaag).
- Oneway valve in BG-aanvoer-leiding net boven etage-niveau — voorkomt terugloop pellet-water richting CV ketel.
- Pellet-retour-tappunt ligt **boven** de oneway (water dat pellet-CV gaat opwarmen).
- Pellet-aanvoer-injectiepunt ligt **onder** de oneway (warm pellet-water gaat richting BG).
- BG-retour en boven-retour komen samen op zolder via T-stuk (samenkomstpunt op x=220, y=357) en gaan **gezamenlijk** door de boiler-spiraal in.
- Het CV-water dat door de spiraal stroomt geeft warmte af aan het tapwater rondom (in de 200 L tank).
- CV-water verlaat de spiraal aan de bovenkant en gaat — nog warm — terug naar de CV-ketel-retour. Hierdoor krijgt de ketel voorverwarmd retourwater (lagere ΔT, hoger rendement bij modulerende werking).

### 2.2 Tapwater-circuit

```
KW-net (T1) ────> Boiler tap-koud-in (onderzijde)
              │
              └─ T1b aftakking ──> Mengkraan koud-in
                                            │
Boiler tap-warm-uit (T2, bovenzijde) ──────> Mengkraan warm-in
                                            │
                                            └──> Mengkraan output (≤ 65°C, T3)
                                                         │
                                                         v
                                            CV ketel tap-koud-in
                                                         │
                                            ┌────────────┘
                                            v
                                            tap-wisselaar (in CV ketel)
                                            │
                                            v
                                   CV ketel tap-warm-uit
                                            │
                                            └──> (T4) ──> Douchekop / overige tappunten
```

**Werking:**
- KW-net stadswater stroomt onderaan de 200 L tank in.
- Het water in de tank wordt continu opgewarmd door de CV-spiraal (passieve restwarmte, levert "voorverwarmd" tapwater).
- Het warme tapwater verlaat de tank bovenin (stratificatie zorgt dat het warmste water bovenin staat).
- Mengkraan mengt warm-uit-tank (kan 65-70°C zijn) met koud KW-net (10°C) tot **maximaal 65°C** — voorkomt verbranding bij de douche en kalk-/levensduurschade aan de combiketel-wisselaar.
- Mengkraan-output gaat naar de **onderzijde** van de CV ketel (tap-koud-in nippel), routing T3 loopt over zolder rond de ketel en komt onderlangs binnen — gelijk aan de andere ketel-aansluitingen.
- In de CW4-combiketel passeert het al-voorverwarmde water de tap-wisselaar en wordt verder verwarmd naar setpoint (typisch 55-60°C voor de douche).
- Tap-warm-uit gaat naar de douchekop (en andere tappunten).

### 2.3 Pellet-circuit

```
CV-aanvoer-leiding tap (boven oneway, bij y=340)
  │
  v
[pad 8: aanvoer naar pellet (= retour pellet-CV)]
  │
  v
Pellet-CV retour-in (onderzijde)
  │
  v
Pellet kachel verbrand pellets · verwarmt water
  │
  v
Pellet-CV aanvoer-uit (top)
  │
  v
[pad 9] VLW Schuur aanvoer-rail
  │
  v (door 3 groepen vloerverwarming, geeft warmte af)
  │
VLW Schuur retour-rail
  │
  v [pad 10]
3-wegklep input
  │
  ├── T-pomp ≥ 35°C → [pad 11] omhoog → injectiepunt onder oneway op CV-aanv
  │
  └── T-pomp < 35°C → [bypass, solid] → terug naar pellet-retour-IN → recirculatie
```

**Werking:**
- Pellet-pomp draait wanneer brander aanstaat. Trekt water uit het CV-systeem via tap-off boven de oneway valve.
- Water gaat door de pellet-kachel-wisselaar, opgewarmd, eruit als aanvoer.
- Door VLW-vloerverwarming (3 groepen) — geeft warmte af aan de schuur-vloer.
- Retour van VLW gaat naar 3-wegklep.
- Pomp-temperatuur sensor bepaalt: ≥ 35°C → klep stuurt water het huis in (CV-aanvoer-injectiepunt onder oneway). Onder 35°C → bypass terug naar pellet-IN voor recirculatie (geen koud water in CV-systeem).
- Oneway valve voorkomt dat warm pellet-water terug-stroomt richting CV-ketel; het MOET dwingend richting BG-verdeler.
- Pad 8 (pellet-retour-IN) en pad 11 (pellet-injectie omhoog) **delen één verticale leiding op x=460** — reduceert pijp-kruisingen tot één enkele over de blauwe BG-retour (5b).

### 2.4 Schema-keuzes (visualisatie 04c)

- **CV ketel verschoven naar links** (x=380-520) — geeft pellet-routing rechts van de rode CV-aanv-lijn ruimte, en bovenin meer plek voor sensor-data.
- **Pellet-circuit rechts van rode lijn** — pad 8 (retour-in) en pad 11 (injectie) delen één verticale leiding op x=460. Slechts één kruising met de blauwe BG-retour (5b) op (460, 469) — netter dan de oude opzet met twee separate verticalen.
- **Bypass solid (niet gestippeld)** — verbindt 3-wegklep onderzijde met pellet-retour (pad 8) op (460, 686) als nette doorlopende leiding. Eerder was dit een vage stippellijn naar een ondefinieerbare plek.
- **Oneway valve als symbool** met richtingspijltje binnenin (cirkel + driehoek), niet als platte driehoek.
- **3-wegklep** weergegeven met live pomp-temperatuur "42°C" in groen IN het kastje (groene rand = actief richting BG).
- **Mengkraan** met live output "56°" IN het 3-poorts kastje, plus "≤ 65°C" beveiligingslabel eronder.
- **Verdeler-labels** boven de boxen voor beide verdelers (boven & BG); VLW-titel boven de schuur-box.
- **Sensor-pillen** bij elke component: temps, modulatie, brander-status, voorraad. Alle aggregate/financiële data in een aparte dashboard-tegel onder het schema.

---

## 3. Sensor-architectuur

### 3.1 Bestaande integraties (al in HA)
- **Toon (gehackt)** — geeft via OpenTherm-gateway: CV ketel modulatie, brander-status, aanvoer-temp, retour-temp, gas-debiet (m³/u), tap-water-actief. Plus huiskamer-temp en setpoint vanuit Toon-thermostaat zelf.
- **Naturela controller** — geeft via MQTT/Modbus: pellet-vermogen, modulatie, brander-status, aanvoer-temp, retour-temp, pomp-temp (= 3-wegklep schakeltemperatuur), klep-positie, brand-fase, voorraad-counter.
- **Tado** — geeft schuur-ambient-temperatuur en luchtvochtigheid.
- **Frank Energie** — geeft dynamische gas- en stroomtarieven (uurprijzen).

### 3.2 Toe te voegen sensoren (DS18B20 via ESPHome) — gerationaliseerd

Na kritische review: 11 → **6 sensoren** (verdeler-temps en mengkraan-output geschrapt).

1. `sensor.boiler_temp_boven` — bovenkant 200 L tank · stratificatie-top
2. `sensor.boiler_temp_onder` — onderkant tank · stratificatie-bottom
3. `sensor.vlw_schuur_g1_temp` — aanvoer groep 1
4. `sensor.vlw_schuur_g2_temp` — aanvoer groep 2
5. `sensor.vlw_schuur_g3_temp` — aanvoer groep 3
6. `sensor.tap_warm_temp` — tap-warm-uit ketel (toont buffer-leeg-effect bij douche)

**Geschrapt** (ratio uitgewerkt):
- ~~Verdeler-temps (4×)~~ — verdelers zitten weggebouwd, geen spanning, en CV-aanvoer/retour wordt al gemeten bij de ketel via OpenTherm. Verdeler-temps wijken hooguit 1-3°C af door pijp-warmteverlies, geen extra diagnostische waarde.
- ~~Mengkraan output~~ — mengkraan is thermostatisch, output is mechanisch begrensd op 65°C. Wordt afgeleid via template: `sensor.mengkraan_output_calculated = min(boiler_boven, 65)`.

**Implementatie**: 6 DS18B20 verspreid over **1-2 ESPHome-nodes**:
- Node 1 (zolder): boiler boven + onder, tap-warm = **3 sensoren**
- Node 2 (schuur): VLW G1/G2/G3 = **3 sensoren**

OneWire-bus per node, mantel-meting met geleidpasta + tape-isolatie.

### 3.3 Berekende/template-sensoren
- `sensor.boiler_stratificatie_dt` — boven − onder (beoordeling buffer-status)
- `sensor.cv_aanvoer_retour_dt` — CV ΔT
- `sensor.gas_kosten_dag` / `_maand` / `_jaar` — utility_meter × Frank Energie tarief
- `sensor.pellet_kosten_dag` / `_maand` / `_jaar` — utility_meter × pellet-prijs (input_number)
- `sensor.totaal_kosten_dag` — gas + pellet
- `sensor.besparing_pellet_dag` — pellet kWh × gas-equivalent-prijs − pellet werkelijke kosten
- `sensor.pellet_voorraad_dagen` — voorraad / gem. dagverbruik

### 3.4 Energie-equivalenten (constanten)
- 1 m³ Slochteren-gas = **9,77 kWh**
- 1 kg pellets klasse A1 = **4,8 kWh**
- 1 kg pellets ≈ **0,49 m³ gas-equivalent**

---

## 4. Klep-nablusfase

Bij uitschakeling van de pellet-brander blijft het pomp-water nog warm — de klep-temperatuur kan boven 35°C blijven hangen tijdens de "naverbrandingsfase" (~10-30 min na laatste pellet-toevoer). 3-wegklep blijft in deze fase richting BG schakelen, en de pellet-pomp draait door om de restwarmte uit te stoten naar de BG. Dit is bewust — geeft maximaal rendement.

---

## 5. Belangrijke ontwerpkeuzes (decisions log)

| # | Keuze | Reden |
|---|-------|-------|
| D1 | Boiler in CV-retour i.p.v. CV-aanv | Maximaliseert restwarmte-recovery: warm retour-water draagt warmte over aan tank vóór het terug naar de ketel gaat. Ketel krijgt voorverwarmd retour, betere modulatie. |
| D2 | CV-water DOOR de spiraal, 200 L tapwater IN de tank | Standaard indirect-fired-boiler-topologie. CV-spiraal als wisselaar binnenin. |
| D3 | Mengkraan met 3 aansluitingen (warm + koud + output) | Veiligheid: tapwater nooit > 65°C, beschermt douche-gebruiker en wisselaar-kalkvorming. |
| D4 | Tapwater via onderzijde CV ketel | Visueel uniform met andere ketel-aansluitingen (CV-aanv, CV-retour). T3 routet rond linkerkant van ketel en komt onderlangs binnen. |
| D5 | CV ketel naar links, pellet rechts van rode lijn | Ruimte-economisch: minder pijp-kruisingen, schoner schema, plek voor sensor-cluster boven. |
| D6 | Pad 8 + pad 11 delen één verticale leiding (x=460) | Reduceert kruisingen met blauwe BG-retour (5b) van 2 naar 1. |
| D7 | Bypass als solid line naar pad 8 | Duidelijke verbinding klep-onderzijde naar pellet-retour, geen vage stippellijn. |
| D8 | 3-wegklep + mengkraan tonen live temp IN de box | Compact, geen extra pillen die kruisen met andere leidingen. |
| D9 | Verdeler-labels BOVEN de boxen | Consistente conventie; voor verdeler BG staat label boven omdat radiatoren BG nu eronder gestapeld staat. |
| D10 | Sensor-pillen (component-relevant) IN het schema, aggregate data ONDER het schema | Scheidt diagrammatische/technische context van financiële/totaal-overzicht. |
| D11 | Verdeler-temps geschrapt | Verdelers weggebouwd zonder stroom; CV-aanv/retour al gemeten bij ketel; pijp-warmteverlies hooguit 1-3°C — geen toegevoegde diagnostische waarde t.o.v. installatie-uitdaging. |
| D12 | Mengkraan-output geschrapt | Thermostatische mengkraan = output is mechanisch fixed op ≤65°C. Afleidbaar via template `min(boiler_boven, 65)` — fysieke meting redundant. |

---

## 6. Verwijzingen

- `04c-CV-water-layout.html` — canonical visuele weergave (incl. dashboard).
- `06-HA-entities.md` — entity-mapping voor Home Assistant.
- `05-Lovelace-config.yaml` — Lovelace dashboard config.
- `04b-Schema-v2.html` — initial schema, vervangen door 04c.
- `04-Dashboard-concept.html` — eerder concept, integreren met 04c.

---

*Laatst bijgewerkt: 2026-04-27 — finale topologie gevalideerd na uitgebreide iteratie met de gebruiker.*
