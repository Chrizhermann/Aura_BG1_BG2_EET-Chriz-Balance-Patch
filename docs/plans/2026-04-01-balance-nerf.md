# Aura NPC Balance Nerf Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Hardcore nerf Aura's items, kit abilities, and XP rewards to bring them in line with vanilla BG power levels.

**Architecture:** All changes are WeiDU-layer patches in .tpa files (SAY DESC text updates + ALTER_ITEM_HEADER / DELETE_EFFECT / ALTER_EFFECT for binary item stats), .baf script edits for bow upgrade triggers, and .d dialogue edits for XP values. Binary .itm/.spl files are NOT edited directly.

**Tech Stack:** WeiDU modding (tp2/tpa/tph), BAF scripting, D dialogue format

---

### Task 1: Nerf Sunshooter Bow descriptions and stats

**Files:**
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:535-1240` (bow section)

**Context:** The Sunshooter currently has 5 tiers x 3 configs (Rapid/Precise/Power) plus Perfect and Opus. We reduce to 3 tiers x 2 configs (Rapid/Precise only).

**Step 1: Update Tier I Rapid (c0aubo1a) description**

In `aura_items.tpa`, replace the SAY DESC for c0aubo1a with:

```
SAY DESC ~The Sunshooter is Aura's personal weapon, a metallic bow of unusual design. Various attachments decorate the arc of the bow, some useful and others purely for accessory.

STATISTICS:

– Configuration: Rapid
  Attacks cannot critically hit
  1 extra attack per round

Charge abilities (1x/day):
– Create Sunshooter Arrows

Combat abilities:
– Magnetism: Each hit reduces target's Armor Class vs. Ranged Weapons by 1 for 5 rounds (max 3 stacks)

THAC0: +1
Damage: -2 (missile)
Speed Factor: 0
Proficiency Type: Shortbow
Type: Two-handed
Requires:
 6 Strength

Weight: 0~
```

Key changes from original:
- Remove "Analytical Archery" (INT-based THAC0) from description
- Remove "19 Intelligence" requirement from description
- Magnetism: add "(max 3 stacks)", remove missile resistance reference
- Remove "almost inhuman speed and accuracy" / "no one besides its creator" flavor (optional, but it oversells)

**Step 2: Update Tier I Precise (c0aubo1b) description**

Same pattern — remove INT THAC0, cap Magnetism, remove INT requirement.

```
SAY DESC ~The Sunshooter is Aura's personal weapon, a metallic bow of unusual design. Various attachments decorate the arc of the bow, some useful and others purely for accessory.

STATISTICS:

– Configuration: Precise
  Attacks never critically miss
  Critical Hit Chance: +5%
  1.5 attacks per round

Charge abilities (1x/day):
– Create Sunshooter Arrows

Combat abilities:
– Magnetism: Each hit reduces target's Armor Class vs. Ranged Weapons by 1 for 5 rounds (max 3 stacks)

THAC0: +2
Speed Factor: 4
Proficiency Type: Shortbow
Type: Two-handed
Requires:
 6 Strength

Weight: 0~
```

Note: THAC0 changed from +3 to +2 for Tier I Precise.

**Step 3: Repurpose Tier II items (c0aubo2a, c0aubo2b)**

- c0aubo2a becomes new Tier II Rapid: THAC0 +2, Damage -1, 1 extra APR
- c0aubo2b becomes new Tier II Precise: THAC0 +3, +5% crit, Speed 4

Update SAY NAME2 and SAY DESC accordingly. Same description template with updated stats.

**Step 4: Repurpose Tier III items (c0aubo3a, c0aubo3b)**

Since scripts transform 2a→3a and 2b→3b at the next level threshold, we repurpose the existing tier 3 items as the final tier:

- c0aubo3a becomes final Tier III Rapid: THAC0 +3, 1 extra APR (no damage penalty)
- c0aubo3b becomes final Tier III Precise: THAC0 +4, +10% crit, Speed 4

Update SAY NAME2 and SAY DESC accordingly.

**Step 5: Gut remaining tier items (c0aubo2c, c0aubo3c, c0aubo4a-c, c0aubo5a-c, c0aubo6, c0aubo7)**

These items will no longer be reachable via scripts, but should still have sensible descriptions in case they exist in a save game. Update their descriptions to say "(Deprecated - this item is no longer obtainable)" or simply make them identical to Tier III equivalents. The simplest approach: leave their descriptions but they'll never be created/transformed to.

**Step 6: Add WeiDU patching for binary stat changes**

After each COPY_EXISTING block for the bows, add ALTER_ITEM_HEADER and effect manipulation. For example, for c0aubo1b (Precise), if the binary has THAC0 +3 but we want +2:

```weidu
COPY_EXISTING ~c0aubo1b.itm~ ~override~
  LPF ALTER_ITEM_HEADER
    INT_VAR
      header_type = 2  // ranged header
      thac0_bonus = 2
  END
  // Remove INT-based THAC0 effect (opcode 282 or custom) if present
  LPF DELETE_EFFECT INT_VAR match_opcode = 282 END
  // Remove missile resistance reduction from Magnetism if baked in
```

Exact opcodes need verification by reading each binary file. The key changes per bow:
- All bows: Remove any opcode that grants INT-based THAC0
- All bows: Remove any missile resistance reduction effect from Magnetism
- Adjust thac0_bonus via ALTER_ITEM_HEADER where binary value differs from target

**Step 7: Remove slot-lock on Sunshooter**

In the Magatama section of aura_items.tpa (around line 68-85), there are `opcode = 180` effects that prevent Aura from equipping other weapons while Sunshooter is equipped. Remove these ADD_ITEM_EQEFFECT blocks. The opcode 319 restriction (Aura-exclusive) stays.

**Step 8: Commit**

```bash
git add Aura_BG1_2_EET/lib/aura_items.tpa
git commit -m "nerf: reduce Sunshooter to 3 tiers, remove INT THAC0, cap Magnetism"
```

---

### Task 2: Update Sunshooter upgrade scripts

**Files:**
- Modify: `Aura_BG1_2_EET/scripts/BGEE/c0aura.baf:812-870`
- Modify: `Aura_BG1_2_EET/scripts/SoD/c02aura.baf:589-642`
- Modify: `Aura_BG1_2_EET/scripts/BG2EE/c0aura2.baf:1127-1200`
- Modify: `Aura_BG1_2_EET/scripts/BG2EE/c0aura25.baf:14-80`

**Step 1: Remap BGEE script (c0aura.baf)**

Current chain: I→II (lvl 6), II→III (lvl 9), III→IV (lvl 12), IV→V (lvl 17)

New chain: I→II (lvl 9), II→III (lvl 17). Remove all other transform blocks.

Replace the SUNSHOOTER section:
```baf
// SUNSHOOTER

IF
	LevelGT(Myself,8)
	OR(2)
	HasItemEquiped("c0aubo1a",Myself)
	HasItemEquiped("c0aubo1b",Myself)
THEN
RESPONSE #100
	DisplayStringHead(Myself,~(Aura has upgraded her bow to Sunshooter II.)~ [GAM_11])
	TransformItem("c0aubo1a","c0aubo2a")
	TransformItem("c0aubo1b","c0aubo2b")
	EquipRanged()
END

IF
	LevelGT(Myself,16)
	OR(2)
	HasItemEquiped("c0aubo2a",Myself)
	HasItemEquiped("c0aubo2b",Myself)
THEN
RESPONSE #100
	DisplayStringHead(Myself,~(Aura has upgraded her bow to Sunshooter III.)~ [GAM_11])
	TransformItem("c0aubo2a","c0aubo3a")
	TransformItem("c0aubo2b","c0aubo3b")
	EquipRanged()
END
```

Delete all remaining transform blocks (III→IV, IV→V, V→Perfect).

**Step 2: Same changes for SoD script (c02aura.baf)**

Same pattern — two transform blocks only, same level thresholds.

**Step 3: Same changes for BG2 script (c0aura2.baf)**

Same pattern. Also remove the Perfect Sunshooter dialogue trigger block (lines ~1202-1218).

**Step 4: Same changes for ToB script (c0aura25.baf)**

Same pattern. Remove all transform blocks beyond II→III. Remove the c0aubo6 check block.

**Step 5: Commit**

```bash
git add Aura_BG1_2_EET/scripts/
git commit -m "nerf: remap Sunshooter upgrades to 3 tiers at level 9/17"
```

---

### Task 3: Nerf arrow types

**Files:**
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1343-1517` (arrow section)

**Step 1: Update Lightning Arrow (c0auaro2)**

Update SAY DESC:
```
THAC0: +3
Damage: 1d6+2 electrical
- Magnetism: Target suffers -1 penalty to AC vs. Ranged Weapons for 5 rounds
```

Add after SAY DESC:
```weidu
LPF ALTER_ITEM_HEADER INT_VAR header_type = 2 thac0_bonus = 3 END
```

Remove chain lightning effect (DELETE_EFFECT for any AoE/chain opcodes). Remove missile resistance component of Magnetism.

**Step 2: Update Static Arrow (c0auaro3)**

Update SAY DESC:
```
– Target suffers -15% penalty to electrical resistance for 3 rounds

THAC0: +2
Damage: 1d6+2 electrical
```

Remove paralyze effect (DELETE_EFFECT match_opcode = 109). Reduce electrical resistance penalty duration.

**Step 3: Update Disruptor Arrow (c0auaro5)**

Update SAY DESC:
```
– Target loses one 8th- or lower level spell protection

THAC0: +3
Damage: 1d6+2 magic
```

Patch THAC0 from +10 to +3. Remove magic resistance reduction effect.

**Step 4: Update Warp Arrow (c0auaro6)**

Update SAY DESC:
```
– Target is Warped into another dimension for one turn. Enemies may make a Saving Throw vs. Spells to avoid the effects.

THAC0: +3
```

Patch THAC0 from +10 to +3. Change save penalty from -4 to 0.

**Step 5: Update Detonator Arrow (c0auaro7)**

Update SAY DESC:
```
Damage: 3d6 explosion upon impact (Save vs. Spell for half)
Area of Effect: 10' radius
```

Patch damage, save penalty, AoE radius.

**Step 6: Update Sniper Shot, Blazing Shot, Arrow Rain**

Change SAY NAME2 from "+5" to "+3" for all variants (c0auboa1-3, c0aubob1-3, c0auboc). Remove stun from Blazing Shot.

**Step 7: Commit**

```bash
git add Aura_BG1_2_EET/lib/aura_items.tpa
git commit -m "nerf: reduce arrow power levels across all types"
```

---

### Task 4: Nerf Magatama, Goggles, and Containers

**Files:**
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:56-155` (Magatama section)
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:140-250` (Goggles section, find exact offset)
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa` (container sections)

**Step 1: Nerf Aura's Magatama (c0amaga1)**

Update SAY DESC:
```
Equipped abilities:
– Saving Throws: +1
– Tamanoya: The wearer is protected by Protection from Evil

Weight: 0
```

Remove the opcode 180 blocks (lines 68-85) that prevent equipping other weapons — this makes the Sunshooter slot-lockable.

Add WeiDU patching:
```weidu
// Reduce saves from +2 to +1
LPF ALTER_EFFECT INT_VAR match_opcode = 33 parameter1 = 1 END
// Remove Death Ward effect
LPF DELETE_EFFECT INT_VAR match_opcode = 210 END  // or whatever opcode Death Ward uses
// Remove Protection from Chaos
// Remove Yasakani (cheat-death) - likely a complex spell reference, delete the casting opcode
```

**Step 2: Nerf Charname's Magatama (c0amaga2)**

Update SAY DESC:
```
Equipped abilities:
– Saving Throws: +1
– Tamanoya: The wearer is protected by Protection from Evil

Weight: 0
```

Same effect deletions as Aura's version.

**Step 3: Nerf Goggles (c0auiden)**

Update SAY DESC to reflect: Identify 1x/day, +10% Find Traps, blindness immunity only.

Remove effects for: Reveal Magic, Detect Illusions (+20%), gaze protection. Reduce Find Traps from +20% to +10%. Reduce Identify charges from 3 to 1.

**Step 4: Nerf Containers**

For c0muffy.sto, c0aubook.sto, c0bandol.sto — these are .sto (store) files that define capacity. Update SAY DESC in aura_items.tpa to say "Capacity: 20" for all three. The actual capacity is in the binary .sto file and may need WRITE_LONG patching or the store files themselves may need editing.

**Step 5: Commit**

```bash
git add Aura_BG1_2_EET/lib/aura_items.tpa
git commit -m "nerf: strip Magatama, Goggles, and container capacity"
```

---

### Task 5: Nerf melee weapons

**Files:**
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1658-1860` (weapons section)

**Step 1: Nerf Blade of Howling Souls +2 (c0ausw01)**

Update SAY DESC to show: 1d10+2, THAC0 +2, Speed 4. Remove all extra abilities from description (boomerang, disarm, crit bonus, extra APR).

Add WeiDU patching:
```weidu
COPY_EXISTING ~c0ausw01.itm~ ~override~
  // Remove extra APR (opcode 1)
  LPF DELETE_EFFECT INT_VAR match_opcode = 1 END
  // Remove boomerang/ranged header if present
  // Adjust damage bonus, speed factor via ALTER_ITEM_HEADER
  LPF ALTER_ITEM_HEADER INT_VAR header_type = 1 thac0_bonus = 2 speed = 4 damage_bonus = 2 END
```

**Step 2: Nerf Jin'u +1 (c0ausw02)**

Update SAY DESC: 1d8+1, THAC0 +1, Speed 3, no extras.

```weidu
COPY_EXISTING ~c0ausw02.itm~ ~override~
  LPF DELETE_EFFECT INT_VAR match_opcode = 15 END  // Remove DEX bonus
  LPF DELETE_EFFECT INT_VAR match_opcode = 176 END // Remove movement bonus
  LPF ALTER_ITEM_HEADER INT_VAR header_type = 1 damage_bonus = 1 speed = 3 END
```

**Step 3: Nerf Death's Razor +2 (c0ausw03)**

Update SAY DESC: 1d8+2, THAC0 +2, Speed 3, +5% crit, no bleed.

Patch THAC0 from +4 to +2, speed from 1 to 3. Remove bleed effect. Keep crit at +5% (reduce from +10% if needed).

**Step 4: Nerf Sword of Wrath (c0auim01)**

Update SAY DESC: 1d10+3 dmg, THAC0 +2, AC -2, Speed 7.

Patch THAC0 from +3 to +2. Enchantment from +3 to +2.

**Step 5: Nerf Biting Thorns (c0auim02)**

Update SAY DESC: 1d6+2, THAC0 +2, Speed 3, minor bleed (1/round 2 rounds, save negates). Enchantment from +3 to +2.

**Step 6: Nerf Onibi-no-kozuchi +2 (c0aclub1)**

Update SAY DESC: 1d6+2, THAC0 +2, Speed 4, 10% fire proc (1/round 2 rounds).

Remove 1d4 fire on-hit damage. Remove low-HP conditional bonus.

**Step 7: Nerf Kiel's Morning Star (c0auim09)**

Update SAY DESC: 2d4+2, THAC0 +2, +1 saves, Speed 4.

Patch THAC0 from +3 to +2, damage from +3 to +2, saves from +2 to +1.

**Step 8: Commit**

```bash
git add Aura_BG1_2_EET/lib/aura_items.tpa
git commit -m "nerf: reduce all melee weapon stats to appropriate levels"
```

---

### Task 6: Nerf equipment items

**Files:**
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1658-1665` (Oni Mask)
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1883-1894` (Bracers)
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1896-1915` (Idol)
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1917-1930` (Gauntlets)
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1932-1964` (Rings)
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:14-49` (Armors)
- Modify: `Aura_BG1_2_EET/lib/aura_bg1.tpa` (Armor patches for BG1)
- Modify: `Aura_BG1_2_EET/lib/aura_bg2.tpa` (Armor patches for BG2)

**Step 1: Nerf Oni Mask (c0amask1)**

Update SAY DESC: AC +1, Charisma -2 only. Remove WIS bonus effect and AoE debuff aura.

**Step 2: Nerf Bracers of Precision (c0auim03)**

Update SAY DESC: +5% crit melee only. Remove THAC0 +2 effect.

**Step 3: Nerf Idol of Kozah (c0auim04)**

Update SAY DESC: +20% electrical resistance, -2 save vs spells penalty. Remove all other effects (lightning immunity, proc, electrical damage bonus, Storm Shield).

**Step 4: Nerf Gauntlets of Security (c0auim05)**

Update SAY DESC: +15% Pick Pockets, +15% Open Locks. Remove THAC0 bonus.

**Step 5: Nerf Ring of Discipline (c0auim06)**

Update SAY DESC: +1 Intelligence only. Remove casting speed bonus.

**Step 6: Nerf Ring of Agility (c0auim07)**

Update SAY DESC: +1 Dexterity only. Remove AC vs missiles bonus.

**Step 7: Nerf Cherry Leather Armors**

Basic (c0aule01): Update SAY DESC to AC 8, no bonuses. Remove save vs breath effect. Patch in all locations (aura_items.tpa, aura_bg1.tpa, aura_bg2.tpa).

Enhanced (c0aule02): Update SAY DESC to AC 6, +1 save vs breath, no elemental resist. Change enchantment from +3 to +2. Remove elemental resistance effects. Patch in all locations.

**Step 8: Commit**

```bash
git add Aura_BG1_2_EET/lib/aura_items.tpa Aura_BG1_2_EET/lib/aura_bg1.tpa Aura_BG1_2_EET/lib/aura_bg2.tpa
git commit -m "nerf: strip equipment items to single-purpose effects"
```

---

### Task 7: Nerf grenades, bombs, and misc items

**Files:**
- Modify: `Aura_BG1_2_EET/lib/aura_items.tpa:1519-1641` (misc items)
- Modify: Various bomb .itm binary patching sections

**Step 1: Nerf Burst Orb (c0autt01)**

Update SAY DESC: 2d6 electrical, 10' AoE. No knockback, no stun.

**Step 2: Nerf Cryonics Rod (c0autt02)**

Update SAY DESC: 3d6 cold, freeze 3 rounds on failed save vs wand. No double damage while frozen, no shatter effect.

**Step 3: Nerf Wildsurge Amplifier (c0autt03)**

Update SAY DESC: Rapid Surge (forces wild surges, no casting speed reduction), +10% positive surge chance. Remove +2 bonus spell slots.

**Step 4: Nerf Invisibility Device (c0autt04)**

Update SAY DESC: Self only, 5 rounds duration. Remove party-wide AoE.

**Step 5: Nerf Screaming Void (c0abomba)**

This item scales via multiple ability headers in the binary. Update SAY DESC to flat 3d6 magic damage, -1 STR 3 rounds. Need to modify or remove level-scaling headers.

**Step 6: Nerf Anti-Mechanism Bomb (c0abombb)**

Update SAY DESC: 2d6 electrical + 2d6 vs constructs, save for half. Remove disable/control.

**Step 7: Nerf Phaser Bomb (c0abombc)**

Update SAY DESC: 2d6 magic + 2d6 vs extraplanar, save for half. Remove stun.

**Step 8: Nerf Firebomb (c0abombd)**

Update SAY DESC: 3d6 fire, save for half. Keep troll regen removal.

**Step 9: Nerf Arcane Disjunctor (c0auimb)**

Update SAY DESC: 1d6 magic, 50% miscast 2 rounds on failed save. No effect on successful save, no casting speed penalty.

**Step 10: Remove permanent stat potions**

For c0apotn1-4: Either remove the GiveItemCreate lines from dialogue, or change the items to temporary buffs. Simplest: change the SAY DESC to show temporary effect and patch timing from permanent to duration-based.

**Step 11: Commit**

```bash
git add Aura_BG1_2_EET/lib/aura_items.tpa
git commit -m "nerf: reduce grenade/bomb/misc item power levels"
```

---

### Task 8: Nerf kit abilities and descriptions

**Files:**
- Modify: `Aura_BG1_2_EET/lib/aura_kit.tpa:47-104` (kit description for auraclass 1)
- Modify: `Aura_BG1_2_EET/lib/aura_kit.tpa` (descriptions for auraclass 2, 3, 4)
- Modify: `Aura_BG1_2_EET/2da/c0auracl.2da` (CLAB table)

**Step 1: Update kit description text (auraclass 1)**

In aura_kit.tpa, update the `help = RESOLVE_STR_REF(...)` string. Key changes:
- Lore: "+10 bonus to Lore" and "+2 additional Lore per level"
- Analyze: "May use Analyze once per day. Gains one additional use at 8th and 16th level."
- Infusion damage: "5% bonus to damage" (flat, no scaling text)
- Infusion enchantment: caps at +3 at 18th level, remove +4/+5/+6 lines
- Elemental Infusion: "5% bonus to damage with and 10% resistance to that element" (no scaling)
- Soul of Artifice: "+1 bonus to Saving Throws" (remove the +3 under Infusion text)
- Remove 24th/32nd/40th level enchantment lines

**Step 2: Update kit descriptions for auraclass 2, 3, 4**

Find and update the same description blocks for the Mage/Thief, Cleric/Thief, and Bard variants. Same text changes.

**Step 3: Update CLAB table (c0auracl.2da)**

The CLAB controls when abilities are granted. Key changes needed:
- Analyze (GA_c0a_a01): Currently at level 5. Keep at level 5 but the spell itself needs charge limits.
- Infusion enchantment thresholds: The enchantment scaling is likely handled by the spell itself (c0auin2 etc.), not the CLAB. The CLAB just grants the ability. The actual spell files (.spl) in the artificer folder would need their effect values changed.

**Step 4: Modify Infusion spell effects**

The Infusion spell files need their damage bonus changed from 10%+scaling to flat 5%, and enchantment caps adjusted. These are binary .spl files patched during installation.

**Step 5: Modify Analyze spell**

Change from at-will/unlimited to 1/day with additional uses. This may require creating a new spell or modifying the existing one to use charges.

**Step 6: Modify Lore bonuses**

The Lore bonus is likely applied via opcode 21 (Lore bonus) in a passive effect spell (AP_c0aukt3 or similar). Change the per-level value from +5 to +2, and base from +20 to +10.

**Step 7: Modify Soul of Artifice**

Remove the +3 saves under Infusion conditional. Keep only the flat +1.

**Step 8: EEex abilities (auraclass 4 only)**

If the EEex Lua files are in scope:
- Elemental Mastery (c0auhl1.spl): Change 30% to 10% resistance
- Ontological Inertia (c0auhl2.spl): Change +50% to +25% duration
- Thaumaturgical Matrix (c0auhl3.spl): Change 3 spells to 2, remove duration modifier

**Step 9: Commit**

```bash
git add Aura_BG1_2_EET/lib/aura_kit.tpa Aura_BG1_2_EET/2da/
git commit -m "nerf: cap Infusion at +3, reduce Lore/Analyze/Soul of Artifice"
```

---

### Task 9: Reduce XP rewards

**Files:**
- Modify: `Aura_BG1_2_EET/dialogue/BGEE/c0auraj.d`
- Modify: `Aura_BG1_2_EET/scripts/SoD/bdwaters.baf`
- Modify: `Aura_BG1_2_EET/scripts/SoD/bdfoods.baf`
- Modify: `Aura_BG1_2_EET/dialogue/BG2EE/dialog.d`
- Modify: `Aura_BG1_2_EET/dialogue/BG2EE/c0aura2j.d`
- Modify: `Aura_BG1_2_EET/scripts/BG2EE/c0aice01.baf`
- Modify: `Aura_BG1_2_EET/scripts/BG2EE/c0aurb.baf`

**Step 1: BG1 XP (c0auraj.d)**

- Line 1109: `AddExperienceParty(5000)` → `AddExperienceParty(1500)`
- Line 1340: `AddExperienceParty(10000)` → `AddExperienceParty(3000)`
- All `AddXPObject("C0Aura",500)` lines (crafting XP): change to `AddXPObject("C0Aura",0)` or remove the call entirely
- Line 1289: `AddXPObject("C0Aura",1000)` → remove or set to 0
- Line 2021: `AddXPObject("C0Aura",1500)` → remove or set to 0

**Step 2: SoD XP**

bdwaters.baf: Change all `AddXPObject(PlayerN,4000)` to `AddXPObject(PlayerN,1500)`
bdfoods.baf: Same change, both occurrences

**Step 3: BG2 XP**

dialog.d:
- Line 1914: `AddExperienceParty(50000)` → `AddExperienceParty(20000)`
- Line 2448: `AddExperienceParty(10000)` → `AddExperienceParty(5000)`
- Line 4467: `AddExperienceParty(60000)` → `AddExperienceParty(25000)`

c0aura2j.d:
- Lines 449, 453: `AddExperienceParty(3500)` → `AddExperienceParty(2000)`

c0aice01.baf:
- Line 11: `AddExperienceParty(50000)` → `AddExperienceParty(20000)`

c0aurb.baf:
- Line 84: `AddExperienceParty(30000)` → `AddExperienceParty(15000)`

**Step 4: Commit**

```bash
git add Aura_BG1_2_EET/dialogue/ Aura_BG1_2_EET/scripts/
git commit -m "nerf: reduce all XP rewards by 50-70%"
```

---

### Task 10: Update Invisibility Device quantity in dialogue

**Files:**
- Modify: `Aura_BG1_2_EET/dialogue/BGEE/c0auraj.d:571`

**Step 1: Reduce quantity**

Change: `GiveItemCreate("c0autt04",Player1,5,0,0)`
To: `GiveItemCreate("c0autt04",Player1,1,0,0)`

**Step 2: Commit**

```bash
git add Aura_BG1_2_EET/dialogue/BGEE/c0auraj.d
git commit -m "nerf: reduce Invisibility Device gift from 5 to 1"
```

---

## Implementation Notes

### WeiDU Patching Reference

Common opcodes for DELETE_EFFECT / ALTER_EFFECT:
- 0 = AC bonus
- 1 = Modify attacks per round
- 6 = Charisma
- 10 = Constitution
- 15 = Dexterity
- 17 = HP per round (regeneration)
- 19 = Intelligence
- 21 = Lore bonus
- 27-32 = Individual save bonuses
- 33 = Save vs all
- 54 = Max HP
- 109 = Paralyze
- 142 = Portrait icon
- 166 = Movement rate modifier
- 176 = Movement rate bonus
- 180 = Cannot use item
- 210 = Protection from spell (Death Ward etc.)
- 282 = Script variable (used for INT THAC0?)
- 319 = Restrict item (usability)

### Binary Item Header Offsets

For items where we need to change core stats not exposed via equipped effects:
- Use `LPF ALTER_ITEM_HEADER` for attack header fields (thac0_bonus, damage_bonus, speed, etc.)
- Use `WRITE_SHORT 0x42 value` for overall enchantment level
- Use `WRITE_SHORT 0x68 value` for number of charges (identify uses etc.)

### Testing

This mod cannot be unit-tested. Verification requires:
1. WeiDU syntax check: `weidu --parse-check Setup-Aura_BG1_2_EET.tp2`
2. Install on a test game and verify items in-game via Near Infinity or EE Keeper
3. Spot-check key items: Sunshooter stats, Magatama effects, XP rewards
