# Pokémon Showdown Tournament Tracker

A personal data pipeline built to track Pokémon usage and performance statistics across a custom competitive tournament series. The system covers data collection, processing, and public reporting across four tools.

---

## Pipeline Overview

```
Google Sheets  →  Python (Colab)  →  R Studio  →  Google Sheets (public)
  (raw data)       (live input)      (analysis)     (final report)
```

---

## 1. Google Sheets — Data Structure

The base dataset was built manually. Each row represents a **Pokémon**, and columns track cumulative statistics across all tournaments.

| Column | Description |
|---|---|
| `Pokémon` | Pokémon name (Showdown format) |
| `Rank` | Competitive ranking estimate by the tournaments organizers. |
| `tipo1` / `tipo2` | Primary and secondary type |
| `Set` | Full Showdown moveset block (used for a randomizer) |
| `TourGanado` | +1 if the Pokémon was on the winning team of a tournament |
| `Usos` | Unique appearances per tournament — counted once per team |
| `Partidas` | Total battles the Pokémon participated in |
| `Wins` | Total victories accumulated across all battles |
| `Mirror` | +1 per battle where both teams shared this Pokémon |
| `Unique` | +1 per battle where only one of the two teams used this Pokémon |

> `Partidas`, `Wins`, `Mirror`, and `Unique` are tracked at the **battle level**. 
> `Usos` and `TourGanado` are tracked at the **tournament level**.

The columns left blank at start (`Partidas`, `Wins`, `Mirror`, `Unique`, `Usos`, `TourGanado`) were populated by Python during the tournament.

---

## 2. Python (Google Colab) — Live Data Entry

At the start of each battle, players share their **team preview** from Pokémon Showdown, which always follows the format:

```
Pokémon / Pokémon / Pokémon / Pokémon / Pokémon / Pokémon
```

This string was pasted directly into a Colab `input()` cell. Python used **set operations** to automatically resolve which Pokémon were shared between teams (Mirror) and which were exclusive to one team (Unique) — no manual if/else logic needed.

### First battle of each player per tournament
*(counts `Usos` — unique appearance per event)*

```python
ganador  = input("Team ganador:  ")
perdedor = input("Team perdedor: ")

ganadores = set(ganador.split(' / '))
perdedores = set(perdedor.split(' / '))
dups = ganadores.intersection(perdedores)   # Mirror: in both teams
ganadores.difference_update(dups)           # Unique to winner
perdedores.difference_update(dups)          # Unique to loser

df.loc[df['Pokémon'].isin(ganadores), 'Unique']  += 1
df.loc[df['Pokémon'].isin(ganadores), 'Partidas'] += 1
df.loc[df['Pokémon'].isin(ganadores), 'Wins']     += 1
df.loc[df['Pokémon'].isin(ganadores), 'Usos']     += 1
df.loc[df['Pokémon'].isin(perdedores),'Unique']   += 1
df.loc[df['Pokémon'].isin(perdedores),'Partidas'] += 1
df.loc[df['Pokémon'].isin(perdedores),'Usos']     += 1
df.loc[df['Pokémon'].isin(dups),      'Mirror']   += 1
df.loc[df['Pokémon'].isin(dups),      'Usos']     += 2
```

### Subsequent battles (semifinals, swiss rounds)
*(same logic, `Usos` not updated — appearance already counted)*

### End of tournament — Champion tracking

```python
campeon   = input("Digite pokes campeones: ")
campeones = set(campeon.split(' / '))
df.loc[df['Pokémon'].isin(campeones), 'TourGanado'] += 1
```

After all rounds, the DataFrame was exported as `.xlsx` for R.

---

## 3. R Studio — Analysis & Tiering

```r
library(readxl)
library(dplyr)
library(tibble)

df <- read_excel("SSB9.xlsx")

# Recalculate Partidas from battle-level columns
df$Partidas <- df$Unique + df$Mirror

# Win Rate
df <- df %>% mutate(WinRate = Wins / Partidas)

# Smogon-style tier classification based on usage
# Replicates the real Smogon tiering logic: usage determines tier
df <- df %>%
  arrange(desc(Usos)) %>%
  mutate(Smogon = case_when(
    ntile(desc(Usos), 3) == 1 ~ "OU",  # Overused  — top third by usage
    ntile(desc(Usos), 3) == 2 ~ "UU",  # Underused — middle third
    ntile(desc(Usos), 3) == 3 ~ "RU"   # Rarely Used — bottom third
  ))
```

Some ranking tables were generated and exported manually to the public Google Sheet:

- **Tabla de Uso** — ranked by total appearances
- **Tabla de WinRate** — ranked by win rate (min. usage threshold applied)
- **Tabla de Campeones** — ranked by tournament wins, with full stats

---

## 4. Google Sheets (Public) — Final Report

The three tables were pasted into a color-coded public Google Sheet accessible to all tournament participants. Tier labels (OU / UU / RU) represent one tertil, with OU being the highest 33%, UU the half 33% and RU the lowest 33%.

> [Link to public Google Sheet](#) ← PENDIENTE: agregar link real

---

## Key Insights (Season 9 sample)

- **Zapdos** led usage with 57 appearances, but ranked outside top 10 in win rate
- **Slaking** held the highest win rate at ~69% despite being classified UU by usage
- **Meloetta-Pirouette** topped the Champions table with 8 tournament wins across 84 battles
- High-Mirror Pokémon (like Zapdos with 28 mirrors) suggest strong meta centrality

---

## Stack

| Tool | Purpose |
|---|---|
| Google Sheets | Raw data entry and public reporting |
| Python / pandas | Live battle input processing |
| R / dplyr | Statistical analysis and tiering |
| Pokémon Showdown | Data source |
