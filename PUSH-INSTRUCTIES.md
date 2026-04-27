# Push status — 2026-04-27

## ✅ HA push: gedeployed (live)

Future Dash → tabblad **HVAC** (mdi:radiator, 8e view).

**Wat erin zit:**
- Hero status met 4 tile-cards (Toon, Pellet, Boiler boven/onder)
- Energie-flow Sankey-diagram (gas + pellet → CV/pellet → buffer/schuur)
- CV ketel detail (Toon-attributes: kamer, setpoint, modulatie, brander, hvac_action)
- Restwarmte boiler met 24u stratificatie-grafiek
- **Pellet CV-kachel** met `naturela-pellet-card` custom card + alle detail-entiteiten
- Schuur (Tado) met 24u temp-trend
- Verbruik & tarieven (Toon gas, Frank Energie prijzen, pellet) + 7-dag bar charts
- 24u trends (5 lijnen: pellet kW + boiler-temps + pellet-circuit)
- Live insights via markdown-templates

**Helpers aangemaakt** (8 totaal):
| ID | Type | Waarde |
|----|------|--------|
| `input_number.pellet_prijs_per_ton` | input_number | 425 €/ton |
| `input_number.pellet_laatste_bestelling_kg` | input_number | 0 (gereset) |
| `input_number.pellet_gem_dagverbruik` | input_number | 0 (gereset) |
| `input_datetime.pellet_laatste_bestelling_datum` | input_datetime | 2026-04-27 (vandaag) |
| `input_datetime.cv_ketel_volgende_onderhoud` | input_datetime | (in te vullen) |
| `input_datetime.pellet_volgende_onderhoud` | input_datetime | (in te vullen) |
| `counter.cv_ketel_brander_uren` | counter | 0 (baseline) |
| `counter.pellet_brander_uren` | counter | 0 (baseline) |

**Reset 2026-04-27**: alle counters/voorraad op 0 als baseline. Naturela-pelletverbruik moet jij nog handmatig resetten op de controller.

---

## ⏸ GitHub push: bundle klaar, jij doet de remote-koppeling

Git-init via de Cowork-mount lukte niet (Windows-permissies blokkeren git's lockfiles). Daarom is alles gebundeld in een **`hvac-thuis.bundle`** bestand. Dit is een single-file git repo met de volledige history.

### Bundle-bestand
- Locatie: `C:\Users\info\Claude\HVAC Thuis\hvac-thuis.bundle` (76 KB)
- Bevat: 1 commit met 16 project-bestanden + .gitignore

### Push naar GitHub (jij doet dit lokaal op Windows)

**Stap 1 — Clone de bundle naar een nieuwe folder:**
```powershell
cd C:\Users\info
git clone "C:\Users\info\Claude\HVAC Thuis\hvac-thuis.bundle" hvac-thuis-git
cd hvac-thuis-git
```

**Stap 2 — Maak een nieuwe repo op GitHub:**
- Ga naar https://github.com/new
- Repo naam: bijv. `hvac-thuis` of `home-assistant-hvac`
- **NIET** initializeren met README/license (je pusht zo de bundle erop)
- Klik **Create repository**

**Stap 3 — Vervang de remote en push:**

`git clone <bundle>` zet `origin` automatisch op het bundle-bestand (read-only). Je moet die vervangen door GitHub:

```powershell
# Verifieer huidige origin (wijst nu naar de bundle)
git remote -v

# Vervang origin van bundle naar GitHub
git remote set-url origin https://github.com/USERNAME/REPO.git

# Optioneel: branch hernoemen naar 'main' als hij anders heet
git branch -M main

# Push
git push -u origin main
```

⚠️ **Niet** `git remote add origin ...` gebruiken — dat geeft een fout omdat origin al bestaat. Gebruik `git remote set-url`.

GitHub vraagt om login — gebruik een **personal access token** (geen wachtwoord meer):
- https://github.com/settings/tokens/new (klassiek) of fine-grained tokens
- Scope: `repo` (full control)
- Plak token als password bij de prompt

### Nieuw commit en push (later)
Als ik (of jij) de project-bestanden updates, kun je via dezelfde flow:
```powershell
cd C:\Users\info\hvac-thuis-git
# Kopieer bijgewerkte files vanuit "Claude\HVAC Thuis"
copy "C:\Users\info\Claude\HVAC Thuis\*.md" .
copy "C:\Users\info\Claude\HVAC Thuis\*.html" .
copy "C:\Users\info\Claude\HVAC Thuis\*.yaml" .

git add -A
git commit -m "Update project files"
git push
```

Of nog netter: maak van de Cowork-folder zelf een git-repo door de bundle daarin uit te pakken. Maar het mount-probleem (Windows-permissies) blijft.

---

## Cleanup

In de workspace staat een afgebroken `.git`-folder die ik niet kon opruimen vanwege permissies. **Verwijder hem handmatig** in Verkenner:
- `C:\Users\info\Claude\HVAC Thuis\.git\` → verwijder

Het bundle-bestand `hvac-thuis.bundle` mag je houden — handig voor de eerste push naar GitHub.

---

## Status alle componenten

| Component | Status |
|-----------|--------|
| Topologie + analyse-docs | ✅ compleet |
| Visualisatie (04c) | ✅ canonical |
| HA dashboard (Future Dash → HVAC) | ✅ live |
| HA helpers + counters | ✅ aangemaakt + reset 0 |
| Sankey energie-flow | ✅ in dashboard |
| Naturela custom card | ✅ in dashboard |
| GitHub repo | ⏸ bundle klaar — jij doet de remote-koppeling |
| utility_meter (configuration.yaml) | ⏸ moet handmatig (geen API-mogelijkheid) |
| template-sensoren (kosten/besparing) | ⏸ inline in dashboard markdown — fysieke entities optioneel later |
| DS18B20 sensoren (VLW G1/G2/G3 + tap-warm) | ⏸ hardware-installatie wachtend |

*Toegangs-token van deze sessie kan je nu intrekken: HA → profiel → Beveiliging → Long-lived access tokens.*
