# Critical HP Config Guide

This guide explains where Critical HP stores its config, what each section does, and how to write valid entries.

## Where the config lives

Critical HP uses two config scopes:

- Client config: `config/criticalhp-client.toml`
- Active world or server config: `<world>/serverconfig/criticalhp-server.toml`
- Default template for new worlds: `defaultconfigs/criticalhp-server.toml`

Use the client file for HUD and local display options.

Use the server file for gameplay rules such as health scaling, damage typing, multipliers, and effects. In singleplayer, this is still the world-scoped server config inside the save.

## How to edit safely

1. Close the game or return to the main menu before editing by hand.
2. Edit the correct TOML file.
3. For server config changes, reload with `/ch reload` after re-entering the world or on a running server.
4. For client HUD changes, restart the client if the HUD does not refresh immediately.

Notes:

- `/ch reload` requires operator-level permission.
- `/ch debug <target>` remains available even in runtimes where Forge's `EntityArgument` is incompatible; in those environments it falls back to string matching by entity id, entity name, or `@s`.
- Critical HP does not currently register its own custom config menu. If you use another mod that exposes Forge config screens, that UI may still work, but the TOML files are the authoritative source.
- On load and reload, Critical HP logs which server config file is active. For new worlds, edit `defaultconfigs/criticalhp-server.toml`. For an existing world, edit that world's `serverconfig/criticalhp-server.toml`.

## Damage types

Critical HP uses three damage classes:

- `slash`
- `pierce`
- `blunt`

Use those exact values in all combat mappings.

## Health section

Server file section:

```toml
[health]
playerHealthScale = 1.0
defaultMobHealthScale = 1.0
entityHealthOverrides = ["minecraft:ender_dragon=400", "minecraft:wither=350"]
```

What each setting does:

- `playerHealthScale`: Multiplies player max health. `1.0` keeps vanilla. `5.0` turns a 20 HP player into 100 HP.
- `defaultMobHealthScale`: Multiplies base max health for living non-player entities unless an override exists.
- `entityHealthOverrides`: Sets an exact max health value for a specific entity.

Format for `entityHealthOverrides`:

- `namespace:entity=value`

Examples:

- `minecraft:ender_dragon=400`
- `minecraft:wither=350`
- `untamedwilds:boar=120`

Use overrides when a specific mob should ignore the global mob scale.

## Combat section

Server file section:

```toml
[combat]
entityDamageMultipliers = ["minecraft:skeleton:pierce=0.75"]
weaponDamageTypes = ["#minecraft:swords=slash", "minecraft:trident=pierce"]
nonWeaponDamageTypes = ["#minecraft:is_projectile=pierce", "minecraft:stalagmite=pierce"]
armorChanceReductionPerPoint = 0.05
```

### Weapon damage types

This forces an item or item tag to resolve to a damage class.
(Note: If enableTfcCompat is enabled, any weapons using TFC's damage classification (slashing/piercing/crushing) will not need to be entered here).

Format:

- Exact item: `namespace:item=damage_type`
- Item tag: `#namespace:tag=damage_type`

Examples:

- `minecraft:trident=pierce`
- `#minecraft:swords=slash`
- `#minecraft:shovels=blunt`
- `#tfc:javelins=pierce`
- `#tfc:maces=blunt`

Use this when you want a weapon to override fallback heuristics or a mod's built-in classification.

Practical rule:

- Prefer an exact item rule for a single problem weapon.
- Prefer a tag rule when an entire family of weapons should behave the same way.

### Non-weapon damage types

This classifies damage that does not come from the held weapon.

Preferred formats:

- Damage type id: `namespace:path=damage_type`
- Damage type tag: `#namespace:tag=damage_type`

Advanced formats also supported:

- `msg_id:id=damage_type`
- `direct_entity:namespace:entity=damage_type`
- `direct_entity_tag:namespace:tag=damage_type`
- `attacker_entity:namespace:entity=damage_type`
- `attacker_entity_tag:namespace:tag=damage_type`

Examples:

- `#minecraft:is_projectile=pierce`
- `minecraft:stalagmite=pierce`
- `minecraft:sonic_boom=blunt`
- `msg_id:cactus=pierce`
- `attacker_entity:minecraft:iron_golem=blunt`

Compatibility note:

- Older configs may use `damage_type:minecraft:arrow=pierce` or `damage_type_tag:minecraft:is_projectile=pierce`. Those legacy forms are still understood, but the simplified forms above are preferred for new edits.

### Entity damage multipliers

This scales incoming damage to the target after the damage class has been resolved.

Format:

- `namespace:entity:damage_type=value`

Examples:

- `minecraft:skeleton:pierce=0.75`
- `minecraft:skeleton:blunt=1.35`
- `minecraft:zombie:slash=1.15`
- `minecraft:iron_golem:blunt=0.85`

How to read the value:

- `1.0` = unchanged
- `< 1.0` = resistance
- `> 1.0` = vulnerability

### Armor-based effect chance reduction

Critical HP can lower effect proc chance based on the target's armor.

Setting:

- `armorChanceReductionPerPoint`

How it works:

- Uses the target's normal vanilla armor value.
- Each 1 point of armor reduces configured proc chance by this amount.
- Default is `0.05`, which means 5 percentage points less proc chance per armor point.
- Set it to `0.0` to disable vanilla armor-based chance reduction.

Example:

- Base effect chance = `0.35`
- Target armor = `8`
- `armorChanceReductionPerPoint = 0.05`
- Armor multiplier = `1.0 - (8 * 0.05) = 0.60`
- Final pre-TFC chance = `0.35 * 0.60 = 0.21`

Important:

- This reduction applies to both players and mobs.
- If TerraFirmaCraft armor resistance reduction is also enabled, both reductions apply together.
- The combined multiplier is clamped into a safe `0.0` to `1.0` range.

## Effect settings

Critical HP has separate effect settings for player targets and mob targets.

- `playerDamageEffectSettings`: effects applied when the damaged target is a player
- `mobDamageEffectSettings`: effects applied when the damaged target is a non-player living mob

Exact nested path format:

- `combat.playerDamageEffectSettings.<damage_type>.<effect_key>`
- `combat.mobDamageEffectSettings.<damage_type>.<effect_key>`

Where:

- `<damage_type>` is one of `slash`, `pierce`, or `blunt`
- `<effect_key>` is the effect id normalized into a config key

Effect key normalization rule:

- start from the full effect id, such as `minecraft:instant_damage`
- replace `:` with `_`
- replace `/` with `_`
- replace `-` with `_`
- lowercase the result

Examples:

- `minecraft:weakness` -> `minecraft_weakness`
- `minecraft:instant_damage` -> `minecraft_instant_damage`
- `minecraft:instant_health` -> `minecraft_instant_health`

Each effect entry has:

- `chance`: decimal from `0.0` to `1.0`
- `duration`: ticks
- `amplifier`: `0` to `2`, where `0 = I`, `1 = II`, `2 = III`

Example structure:

```toml
[combat.playerDamageEffectSettings.pierce.minecraft_weakness]
chance = 0.35
duration = 120
amplifier = 1

[combat.mobDamageEffectSettings.pierce.minecraft_instant_damage]
chance = 0.35
duration = 1
amplifier = 0

[combat.mobDamageEffectSettings.slash.minecraft_wither]
chance = 0.35
duration = 120
amplifier = 1
```

Default behavior summary:

- `pierce` tends to apply debuffs such as weakness and mining fatigue
- `blunt` tends to apply blindness or nausea
- `slash` tends to apply wither, slowness, or weakness

Important behavior:

- Player-target and mob-target effects are configured separately.
- Negative or oversized values are clamped into safe ranges.
- For undead mobs, a configured pierce `minecraft:instant_damage` effect is automatically swapped to `minecraft:instant_health` so the effect still harms undead instead of healing them.

## TerraFirmaCraft section

Server file section:

```toml
[tfc]
enableTfcCompat = true
enableTfcArmorResistanceChanceReduction = true
tfcArmorResistanceChanceReductionPerPoint = 0.05
```

What these do:

- `enableTfcCompat`: enables TFC-specific hooks such as weapon classification and health-display compatibility.
- `enableTfcArmorResistanceChanceReduction`: lowers effect proc chance based on worn TFC armor resistances.
- `tfcArmorResistanceChanceReductionPerPoint`: amount removed from effect chance for each resistance point.

How this interacts with normal armor reduction:

- `armorChanceReductionPerPoint` uses the target's normal vanilla armor value.
- `tfcArmorResistanceChanceReductionPerPoint` uses TFC's per-damage-type resistance values.
- When both are enabled, Critical HP multiplies both reductions together.
- The debug log reports both contributions separately.

Example:

- `0.05` means each resistance point removes 5 percentage points from the configured proc chance.

Recommendation:

- Leave TFC compat enabled in TFC packs unless you are isolating a bug.

## Misc section

Server file section:

```toml
[misc]
debugLogging = false
```

What it does:

- `debugLogging`: writes verbose classification, multiplier, effect, and reload information to `latest.log`.

Recent debug output includes:

- active server config path on load and reload
- weapon and non-weapon damage classification details
- mob scaling and spawn stabilization details
- undead effect-swap detection details
- effect proc rolls with `baseChance`, `adjustedChance`, and `blockedByArmor`
- combined armor reduction details including vanilla armor points and TFC resistance contributions

Enable this when testing new weapon rules, non-weapon rules, or mob multipliers.

## HUD section

Client file section:

```toml
[hud]
enableHudOverlayIntegration = true
showHudOutsideSurvival = true
replaceVanillaHearts = false
showCustomHealthBar = false
showCustomHealthText = false
```

What each setting does:

- `enableHudOverlayIntegration`: enables Critical HP HUD handling and overlay interception.
- `showHudOutsideSurvival`: allows the Critical HP HUD to render outside normal survival HUD contexts such as creative mode. Disable it if you want the HUD limited to survival-style gameplay only.
- `replaceVanillaHearts`: replaces the standard player health display when the layout is compatible.
- `showCustomHealthBar`: draws the custom Critical HP bar.
- `showCustomHealthText`: shows numeric HP text.

Current behavior note:

- In large modpacks, some UI mods can clear Forge overlay registrations after startup.
- Critical HP now also renders inline during intercepted health overlay events when heart replacement is active, which makes the HUD more resilient to overlay-registry resets.
- If the HUD still breaks, it is more likely a live HUD conflict or render-state conflict than a simple startup timing issue.

Recommended combinations:

- Vanilla-like replacement: `enableHudOverlayIntegration=true`, `replaceVanillaHearts=true`, `showCustomHealthBar=true`
- Survival-only HUD: also set `showHudOutsideSurvival=false`
- Text-focused debugging: also set `showCustomHealthText=true`
- HUD conflict isolation: set `enableHudOverlayIntegration=false`

If another HUD mod is also drawing hearts or a health widget, disable overlay integration first to confirm whether the issue is a layout conflict.

## Testing workflow

When changing gameplay config:

1. Edit the world's `criticalhp-server.toml`.
2. Run `/ch reload`.
3. Attack the target with a known weapon.
4. Check `latest.log` if the result is not what you expected.

Useful test commands:

1. `/ch debug @s`
2. `/ch debug minecraft:zombie`
3. `/ch reload`

When changing HUD config:

1. Edit `criticalhp-client.toml`.
2. Restart the client if needed.
3. Check for overlap with other HUD mods.

## Troubleshooting

### My config edit did not seem to apply

- Make sure you edited the active world's `serverconfig/criticalhp-server.toml`, not just `defaultconfigs/criticalhp-server.toml`.
- Use `/ch reload` after editing server-side settings.
- Turn on `debugLogging` and watch `latest.log` for the active config path, effect-roll diagnostics, armor chance multipliers, and active classification details.

### A weapon still uses the wrong damage type

- Add an exact item rule first.
- If many related items are wrong, switch to a tag rule.
- In TFC packs, prefer rules like `#tfc:javelins=pierce` or `#tfc:maces=blunt`.

### Effects are working on players but not mobs, or the reverse

- Check the correct effect section. Player targets and mob targets do not share the same runtime table.

### The HUD is duplicated or misplaced

- Set `enableHudOverlayIntegration=false` to isolate HUD conflicts.
- Then re-enable it and test with `showCustomHealthBar` and `showCustomHealthText` one at a time.
- If the log shows other mods repeatedly rebuilding or removing overlays, treat the issue as a live HUD conflict rather than a bad config value.

## Quick examples

```toml
[health]
playerHealthScale = 5.0
defaultMobHealthScale = 2.0
entityHealthOverrides = ["minecraft:wither=500", "untamedwilds:boar=120"]

[combat]
entityDamageMultipliers = ["minecraft:skeleton:blunt=1.25", "minecraft:zombie:pierce=0.80"]
weaponDamageTypes = ["#tfc:javelins=pierce", "#tfc:maces=blunt", "minecraft:trident=pierce"]
nonWeaponDamageTypes = ["#minecraft:is_projectile=pierce", "minecraft:sonic_boom=blunt"]
armorChanceReductionPerPoint = 0.05

[tfc]
enableTfcCompat = true
enableTfcArmorResistanceChanceReduction = true
tfcArmorResistanceChanceReductionPerPoint = 0.05

[misc]
debugLogging = true
```

That setup would:

- give players 100 max HP
- double most mob health
- make skeletons weaker to blunt damage
- make zombies resist pierce damage
- force TFC javelins to pierce and TFC maces to blunt
- reduce effect proc chance by normal armor value and, in TFC packs, by TFC armor resistances as well
- log extra detail while you test
