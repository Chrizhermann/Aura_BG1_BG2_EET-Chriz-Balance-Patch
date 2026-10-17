# Aura NPC Mod for BG1/BG2/EET

## Project Overview

This is the **Aura NPC mod** for Baldur's Gate: Enhanced Edition, BG2:EE, and EET (Enhanced Edition Trilogy). Aura is a gnome Artificer companion who can be found:
- **BG1**: Beregost
- **SoD**: West of Coast Way Crossing
- **BG2**: Jansen home in the Slums, then Waukeen's Promenade

Original author: **The Artisan**. This repo is a balance patch fork by Chriz.

## Mod Structure

```
Aura_BG1_2_EET/                    # Main mod folder (%MOD_FOLDER%)
  Setup-Aura_BG1_2_EET.tp2         # WeiDU installer
  lib/                             # WeiDU include libraries
    aura_bg1.tpa                   # BG1 installation logic
    aura_bg2.tpa                   # BG2 installation logic
    aura_kit.tpa                   # Artificer kit definition
    aura_items.tpa                 # Item creation
    aura_resources.tpa             # Shared resources
    functions.tph                  # Utility functions
    crossmod.tpa / crossmod_bg1.tpa # Cross-mod compatibility
    bgforge/                       # BGForge library (third-party)
  dialogue/
    BGEE/                          # BG1 dialogues (.d files)
    BG2EE/                         # BG2 dialogues (.d files)
    SoD/                           # Siege of Dragonspear dialogues
    BG2EE/crossmod/                # Cross-mod banter/interjections (many NPC mods)
  scripts/
    BGEE/                          # BG1 area/creature scripts (.baf)
    BG2EE/                         # BG2 area/creature/cutscene scripts
    SoD/                           # SoD scripts
  tra/english/                     # Translation strings (.tra files)
  creatures/                       # .cre files (BG2, SoD)
  items/                           # .itm files
  spells/                          # .spl files
  artificer/                       # Artificer kit resources (2da, items, spells, scripts)
    eeex/                          # EEex-dependent Artificer (Bard) variant
  animations/                      # Custom creature animations (.bam, .ini)
  areas/                           # Area files
  portraits/                       # NPC portrait .bmp files
  alt-portrait/                    # Alternate portrait options
  2da/                             # 2DA tables (kit, class, runes)
```

## Install Components (tp2)

| # | Component | Requirement |
|---|-----------|-------------|
| 0 | Aura NPC for BG:EE | BGEE |
| 1 | Aura NPC for BG2:EE | BG2EE |
| 2 | Aura NPC for EET | EET |
| (subcomponent) | Choose class for Aura | Requires 0/1/2 |
| - Artificer (Thief) | auraclass=1 |
| - Illusionist/Artificer (Mage/Thief) | auraclass=2 |
| - Priestess/Artificer (Cleric/Thief) | auraclass=3 |
| - Artificer (Bard, EEex) | auraclass=4, requires EEex |
| (portraits) | NPC Portraits for BG2 | Requires 1/2 |
| (subcomponent) | Alternate portraits | Multiple options |
| (last) | Import BG1 upgrades into BG2 | Requires 1/2 |

## Key Variables

- `%MOD_FOLDER%` = `Aura_BG1_2_EET`
- `auraclass` = 1-4 (determines class variant)
- `bg2_chapter` = offset by 12 for EET
- `bg1_eet_symbol` = `AR` (BGEE) or `BG` (EET) for area prefix
- `include_saves` = 1 when class component is installed

## File Naming Conventions

- Prefix: `C0A` or `c0a` for Aura-specific resources
- `C0AU` prefix for Aura creature/script files
- `C0ACUT` prefix for cutscene scripts
- Dialog files: `c0aura` (BG1), `c0aura2` (BG2), `c0aura25` (ToB)
- Banter: `bc0aura` (BG1), `bc0aura2` (BG2), `bc0aur25` (ToB)

## Working With This Mod

- **Dialogues**: Written in WeiDU .d format with `AUTO_TRA` for string references
- **Scripts**: Written in .baf (BAF script format), compiled by WeiDU
- **EET compatibility**: Handled via `lib/eet.tph` and cpmvars; chapter numbers offset by 12
- **Cross-mod content**: Each cross-mod has its own subfolder under `dialogue/BG2EE/crossmod/`
- **bgforge library**: Third-party lib in `lib/bgforge/` - do not modify

## Conventions

- Use `~tilde~` delimiters for WeiDU strings, not quotes
- String references use `@nnn` format with AUTO_TRA
- Game detection: `GAME_IS ~BGEE~`, `GAME_IS ~BG2EE~`, `GAME_IS ~EET~`
- SoD detection: `GAME_INCLUDES ~sod~`
- Resource names are limited to 8 characters (IE engine limitation)
