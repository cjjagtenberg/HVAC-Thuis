# HVAC Thuis — Projectoverzicht

**Doel:** complete Home Assistant integratie en dashboard voor het verwarmingssysteem (CV‑ketel, restwarmte‑boiler, pellet‑CV‑kachel, vloerverwarming, Toon thermostaat) inclusief moderne grafische vormgeving.

**Teamleider:** Cor
**Datum start:** 2026-04-26

---

## Documenten in dit project

| # | Document | Doel |
|---|----------|------|
| 00 | `00-Projectoverzicht.md` | Dit bestand — index en navigatie |
| 01 | `01-Analyse-systeem.md` | Fysieke installatie, waterstromen, regellogica |
| 02 | `02-Architectuur-HA.md` | Integratielagen, datastromen, beslissingen |
| 03 | `03-Entities-en-sensoren.md` | Volledige lijst entities, helpers, automations |
| 04 | `04-Dashboard-concept.html` | Visuele mockup (dark glassmorphism) |
| 05 | `05-Lovelace-config.yaml` | Werkende basis YAML voor het HA dashboard |
| 06 | `06-Team-en-rollen.md` | Rolverdeling, milestones, planning |

---

## Beslissingen tot nu toe

- **Stijl:** Dark glassmorphism (zwarte basis, glaskaarten, neon‑accenten op temperatuur en flow).
- **Layout:** Volledige dashboard view — hoofdtegel met schema + KPI's, daaronder detail‑tegels per component.
- **Deliverable:** Compleet pakket — analyse, mockup, en Lovelace YAML.
- **Hoofdtegel‑inhoud:** schematische flow + KPI strip onderin.

## Volgende stappen na review

1. Cor reviewt mockup en analyse.
2. Backend zet ontbrekende sensoren / integraties live (Toon, Naturela, boiler temps).
3. Frontend implementeert YAML in HA en finetuned card‑mod.
4. Tester valideert per scenario (zomer/winter, pellet aan/uit, tapwater piek).
