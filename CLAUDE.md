# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Deadly Boss Mods (DBM) is a World of Warcraft addon that warns players about boss abilities during combat. It has two layers:

- **DBM-Core**: The framework — timer bars, sound/visual alerts, event dispatching, UI, localization, network sync.
- **Boss mods**: Thin Lua files that register for combat log events and call framework APIs to fire alerts. One file per boss encounter.

## Code Quality Checks

There is no build step. Linting runs in CI via two tools:

```bash
# Luacheck (syntax and style)
luacheck .

# LuaLS type checking (uses .luarc.json config)
# Requires the LuaLS language server — run via the CI action locally or VSCode extension
```

Luacheck config is in `.luacheckrc`. Globals are validated by LuaLS rather than luacheck (luacheck has `ignore = {"1.."}` for all global warnings).

## Tests

Tests are **characterization tests** that replay recorded combat logs in-game and compare output to a stored "golden" report. They require a running WoW client with DBM-Test loaded.

```
# In-game slash commands:
/dbm test <test name>       -- run a single test
/dbm test                   -- run all tests
```

**Adding a test:**
1. Record a boss fight with [Transcriptor](https://www.curseforge.com/wow/addons/transcriptor)
2. Convert the log: `lua DBM-Test/Tools/CreateTest.lua /path/to/log`
3. Run `/dbm test <name>` in-game, inspect the report
4. Store the golden: `lua DBM-Test/Tools/ImportTestResults.lua --prefix <raid>/<boss> /path/to/SavedVariables/DBM-Test.lua /path/to/reports/`
5. Commit the test data and golden file together; include the behavior diff in the PR

Tests live under `DBM-Test/`. The runner (`Runner.lua`) mocks WoW APIs (`Mocks.lua`) and warps time (`TimeWarper.lua`) to replay events.

## Repository Structure

```
DBM-Core/               Framework: core engine, UI components, module system
  DBM-Core.lua          Main ~11k-line framework file
  DBM-Arrow/Flash/RangeCheck/InfoFrame/HudMap/Nameplate.lua  UI components
  modules/              Internal framework modules
    objects/            Prototype classes (BossMod, Timer, Announce, SpecialWarning, Yell, PrivateAura, …)
    gui/                In-combat info displays (latency, durability, keystones)
    *.lua               Feature modules (Commands, Scheduler, Icons, TargetScanning, SpecRole, …)
  localization.*.lua    Core UI strings (10 languages)
  commonlocal.*.lua     Shared combat terminology (10 languages)
DBM-GUI/                Options UI — configuration panels
DBM-StatusBarTimers/    Timer bar library (dependency of DBM-Core)
DBM-Raids-WarWithin/    Raid boss mods for War Within expansion
DBM-Raids-Midnight/     Raid boss mods for Midnight expansion
DBM-KhazAlgar/          World boss mods for Khaz Algar
DBM-Midnight/           Dungeon boss mods for Midnight
DBM-Brawlers/           Brawlers Guild mods
DBM-Test/               Test harness (characterization tests)
```

## Multi-Expansion TOC Files

One codebase targets six WoW versions. Each addon has a separate `.toc` per version:

- `AddonName_Mainline.toc` — retail/current
- `AddonName_Wrath.toc`, `_Cata.toc`, `_Mists.toc`, `_TBC.toc`, `_Vanilla.toc` — classic

The `.pkgmeta` file drives packaging: the BigWigs packager pulls external library repos, then `move-folders` splits subdirectories into separate addon packages for distribution. In the repo, all boss mods live as siblings to `DBM-Core/`; during release they are packaged independently.

## Boss Mod Architecture

Every boss file follows the same pattern:

```lua
local mod = DBM:NewMod(journalID, "DBM-PackageName", subTab, instanceId)
local L   = mod:GetLocalizedStrings()  -- pulls locale table for current client language

mod:SetRevision("@file-date-integer@")  -- replaced by packager
mod:SetCreatureID(npcId)
mod:SetEncounterID(encounterId)

mod:RegisterCombat("combat")  -- or "yell", "cast", etc.
mod:RegisterKill("yell", L.Win)

mod:RegisterEventsInCombat(
    "SPELL_CAST_START 12345 67890",
    "SPELL_AURA_APPLIED 11111"
)

-- Alert/timer objects are created once at file scope:
local specWarnFoo = mod:NewSpecialWarningDodge(spellId, ...)
local timerFooCD  = mod:NewCDCountTimer(duration, spellId, ...)

-- State goes in mod.vb (variable bucket), reset in OnCombatStart:
mod.vb.count = 0

function mod:OnCombatStart(delay)
    self.vb.count = 0
    timerFooCD:Start(firstCastDelay - delay, 1)
end

-- Event handlers are methods named after the registered event:
function mod:SPELL_CAST_START(args)
    if args.spellId == 12345 then
        specWarnFoo:Show()
        specWarnFoo:Play("watchstep")
        timerFooCD:Start()
    end
end
```

**Key conventions:**
- `mod.vb` (variable bucket) holds all per-fight mutable state. Always reset every field in `OnCombatStart`.
- Timer and alert objects are module-scoped locals, never created inside event handlers.
- Difficulty branching uses `self:IsMythic()`, `self:IsHeroic()`, `self:IsNormal()`.
- Tank mechanics use `self:IsTanking(unit)`.
- `@file-date-integer@` is a packager placeholder for the revision timestamp.

## Alert Object Types

| Constructor | Purpose |
|---|---|
| `mod:NewAnnounce(spellId, importance)` | Chat/raid warning message |
| `mod:NewSpecialWarningDodge/Run/Move/...` | Large on-screen warning + sound |
| `mod:NewSpecialWarningDefensive/Taunt/...` | Role-specific tank/healer warnings |
| `mod:NewSpecialWarningGTFO(spellId)` | Stand-in-bad detection |
| `mod:NewCountAnnounce(spellId, ...)` | Announces with stack/count |
| `mod:NewIncomingCountAnnounce(spellId, ...)` | Incoming debuff targeting you |
| `mod:NewYell(spellId)` | Auto-yell in chat |
| `mod:NewTimer(duration, spellId)` | Basic countdown timer |
| `mod:NewCDTimer/CDCountTimer(...)` | Cooldown timer with count tracking |
| `mod:NewAITimer(duration, spellId)` | Auto-learned timer (PTR/new content) |
| `mod:NewBuffFadesTimer(duration, spellId)` | Timer for buff/debuff expiry |
| `mod:AddPrivateAuraSoundOption(spellId)` | Private aura sound alert |

## Localization

Each addon has `localization.*.lua` for 10 languages (en, de, fr, es, br, ru, cn, tw, kr, it). The English file defines all strings; other locales override keys they translate. Boss mods access strings via `mod:GetLocalizedStrings()` which returns the appropriate `L` table with English fallback.

String keys in boss mods follow the alert object name (e.g., `L.specWarnFoo` for the text shown by `specWarnFoo:Show()`). Core-wide strings live in `DBM_CORE_L`; shared combat terms (Tank, Healer, etc.) live in `DBM_COMMON_L` (from `commonlocal.*.lua`).

## Internal Module System

DBM-Core uses a private namespace (`select(2, ...)`) passed between files. Modules register themselves as prototypes or named modules:

```lua
-- Access in any file loaded after registration:
local DBM       = private:GetPrototype("DBM")
local scheduler = private:GetModule("DBMScheduler")
local timer     = private:GetPrototype("Timer")
```

This avoids global pollution while allowing cross-file communication without `require`.

## CI / Release

CI runs on every push/PR to `master`:
1. **Luacheck** — static analysis
2. **LuaLS** — type checking via `DeadlyBossMods/LuaLS-config` action

On successful merge to `master` (or a tag push), the **BigWigs packager** creates release zips and publishes to CurseForge, GitHub Releases, Wago, and WoWInterface simultaneously. The `@project-version@` and `@file-date-integer@` tokens in TOC/Lua files are substituted during packaging.
