# 06 — Team, rollen en planning

## 1. Team-samenstelling

| Rol | Verantwoordelijkheden | Tools / Skills |
|-----|----------------------|----------------|
| **Cor — Teamleider / Product Owner** | Visie, prioriteiten, beslissingen, praktijkkennis installatie. Test in‑situ. | Toon, Naturela, fysieke installatie, HA host |
| **Frontend (HA Lovelace)** | Dashboard YAML, custom cards, card‑mod styling, picture‑elements schema, responsive layout. | YAML, HACS cards, Card‑mod, SVG/CSS |
| **Backend (Integraties & Logica)** | MQTT/Modbus brokers, ESPHome firmware, template sensors, automations, helpers, statistieken. | ESPHome, MQTT, Python templates, AppDaemon (optie) |
| **Tester / QA** | Scenarios valideren (zomer/winter, pellet aan/uit, tapwater piek), edge‑cases, regressie na updates, alerts validatie. | HA Logbook, Influx, lange-termijn statistieken |

## 2. Rolverdeling per onderdeel

| Onderdeel | Frontend | Backend | Tester | Cor |
|-----------|---------|---------|--------|-----|
| Toon koppeling | — | ✓ | ✓ | review |
| Naturela koppeling | — | ✓ | ✓ | review |
| Boiler ESPHome | — | ✓ | ✓ | hardware |
| Vloerverw. pomp meting | — | ✓ | ✓ | hardware |
| Template sensoren | — | ✓ | ✓ | review |
| Automations | review | ✓ | ✓ | review |
| Dashboard YAML | ✓ | review | ✓ | review |
| SVG schema | ✓ | — | — | review |
| Card‑mod styling | ✓ | — | ✓ | review |
| Documentatie | ✓ | ✓ | ✓ | eindredactie |

## 3. Milestones

| # | Milestone | Verantwoordelijk | Acceptatiecriterium |
|---|-----------|------------------|--------------------|
| M1 | **Sensoren live** | Backend + Cor | Alle entities uit §03 leveren waarden, naming‑conventie correct, geen "unavailable". |
| M2 | **Template laag werkt** | Backend | `sensor.actieve_warmtebron` schakelt correct tussen Pellet/Gas/Restwarmte/Geen in alle scenario's. |
| M3 | **Dashboard v1** | Frontend | YAML uit §05 rendert zonder errors, alle tegels gevuld met live data. |
| M4 | **Styling match mockup** | Frontend | Card‑mod styling komt visueel overeen met `04-Dashboard-concept.html` (≥90% similar). |
| M5 | **Automations actief** | Backend + Tester | 5 high‑level automations uit §02‑6 draaien, geen valse alerts >24u. |
| M6 | **Energie dashboard** | Backend | Gas + pellet + elektra in HA Energy dashboard, dagelijkse besparing zichtbaar. |
| M7 | **Acceptatie Cor** | Cor | Eindgebruiker keurt af op gebruiksgemak en visuele kwaliteit. |

## 4. Planning (richting indicatief)

```
Week 1   M1  ──[Backend]──▶ sensoren live
Week 2   M2  ──[Backend]──▶ template laag
Week 2-3 M3  ──[Frontend]──▶ dashboard v1
Week 3   M4  ──[Frontend]──▶ styling match
Week 4   M5  ──[Backend+Tester]──▶ automations
Week 4   M6  ──[Backend]──▶ energie dashboard
Week 5   M7  ──[Cor]──▶ acceptatie
```

## 5. Definition of Done — per item

- **Sensor:** levert min. 1u stabiel data, attributen `device_class` + `state_class` correct, naming conform.
- **Template sensor:** unit‑test in HA Developer Tools voor minstens 3 input‑combinaties.
- **Automation:** trigger getest, conditie getest, action getest, `mode: single` of `restart` bewust gekozen, voorzien van `description`.
- **Dashboard tegel:** rendert in dark mode, responsief op tablet (1024px) en desktop (1440px+), geen "Entity not found" warnings.
- **Documentatie:** wijziging gereflecteerd in betreffende `.md` file.

## 6. Risico's & mitigaties

| Risico | Impact | Mitigatie |
|--------|--------|-----------|
| Toon hack stopt na firmware update | Hoog (referentie ruimtetemp valt weg) | Aanvullende Aqara sensor in woonkamer als fallback |
| Naturela protocol onvolledig gedocumenteerd | Middel | Begin met passieve uitlezing; geen schrijfacties tot bevestigd |
| 1‑Wire kabels te lang → meetfouten | Laag/Middel | Korte buslengte, parasite power vermijden, kalibratie via referentiethermometer |
| Custom cards breken na HA update | Middel | Card‑mod versie pinnen in HACS, releases reviewen vóór updaten |
| Energie‑besparing claim onnauwkeurig | Laag | Markeren als "geschat", werkelijke meting met kalorimeter optioneel |

## 7. Communicatie

- **Stand‑up:** asynchroon — wekelijkse status in `99-Logboek.md` (toe te voegen).
- **Wijzigingen** aan documenten via duidelijke commit‑message als project onder Git komt.
- **Beslissingen** worden in `00-Projectoverzicht.md` onder "Beslissingen tot nu toe" bijgewerkt.
