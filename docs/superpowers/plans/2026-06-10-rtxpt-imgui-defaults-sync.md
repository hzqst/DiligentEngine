# RTXPT ImGui Defaults Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring six default values in the ported Diligent RTXPT UI state structs into alignment with the upstream RTXPT-fork `SampleUI` post-construction + Balanced-preset state.

**Architecture:** Pure default-literal edits to two C++ header structs — no runtime wiring, no new fields, no shader/render-target changes. Correctness is verified statically by cross-checking each changed literal against the upstream source-of-truth file/line, plus a `git diff` review. No unit-test target is added (see "Testing Approach" below).

**Tech Stack:** C++ headers in the `DiligentSamples` git submodule. Verification via `rg` (ripgrep) and `git diff`. Optional compile via the existing CMake/Visual Studio build (user-initiated only).

---

## Source of Truth

All target values come from the upstream RTXPT-fork "state after construction + `ApplyPreset(Balanced)`":

| # | Field | File:struct | Current | Target | Upstream reference |
|---|---|---|---|---|---|
| 1 | `NEEType` | `RTXPTSample.hpp::RTXPTReferenceUIState` | `1` | `2` | `D:/RTXPT-fork/Rtxpt/SampleCommon/CommandLine.h:42` (`NEEType = 2`; ctor copies cmdline, preset doesn't touch it) |
| 2 | `NEEMISType` | `RTXPTSample.hpp::RTXPTReferenceUIState` | `0` | `1` | `D:/RTXPT-fork/Rtxpt/SampleUI.h:156` + Balanced preset row (`SampleUI.cpp:66`) |
| 3 | `NRDMethod` | `RTXPTRealtimeSettings.hpp::RTXPTRealtimeSettings` | `RTXPTNrdMethod::RELAX` | `RTXPTNrdMethod::REBLUR` | `D:/RTXPT-fork/Rtxpt/SampleUI.h:293` (`NRDMethod = NrdConfig::DenoiserMethod::REBLUR`) |
| 4 | `MaxFastAccumulatedFrameNum` | `RTXPTRealtimeSettings.hpp::RTXPTNrdReblurUiSettings` | `0` | `6` | `D:/RTXPT-fork/External/Nrd/Include/NRDSettings.h:241` (`nrd::ReblurSettings` default; not overridden by `getDefaultREBLURSettings()`) |
| 5 | `HistoryFixFrameNum` (REBLUR) | `RTXPTRealtimeSettings.hpp::RTXPTNrdReblurUiSettings` | `0` | `3` | `D:/RTXPT-fork/External/Nrd/Include/NRDSettings.h:249` |
| 6 | `HistoryFixFrameNum` (RELAX) | `RTXPTRealtimeSettings.hpp::RTXPTNrdRelaxUiSettings` | `0` | `3` | `D:/RTXPT-fork/External/Nrd/Include/NRDSettings.h:346` |

Spec: `docs/superpowers/specs/2026-06-10-rtxpt-imgui-defaults-sync-design.md`.

## File Structure

Two files are modified; both live in the `DiligentSamples` submodule. No files are created.

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` — `RTXPTReferenceUIState` struct (fields #1, #2).
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp` — `RTXPTRealtimeSettings`, `RTXPTNrdReblurUiSettings`, `RTXPTNrdRelaxUiSettings` structs (fields #3–#6).

## Testing Approach (read before starting)

This change is six default-literal edits. There is **no unit-test harness wired for the RTXPT sample** (it is a runtime graphics app in `DiligentSamples`, not part of the GoogleTest build), and adding a CMake test target solely to assert literal defaults would violate YAGNI and the spec's stated scope ("Do not wire new runtime behavior"). Therefore the "test" for each task is **static verification**: `rg` confirms the local literal now equals the upstream source-of-truth value cited above.

Each task follows this rhythm, mirroring red→green→commit:
1. **Red** — `rg` the current (wrong) literal to confirm the change is needed.
2. **Green** — apply the exact edit.
3. **Verify** — `rg` the new literal in the local file, and confirm it matches the cited upstream value.
4. **Commit** — inside the submodule.

Per the user's standing instruction, do **not** run build commands automatically. A compile is offered as an optional, user-initiated step in Task 3.

The single-digit numeric edits (#1, #2, #4, #5, #6) preserve clang-format alignment (digit→digit, same width). The enum edit (#3) only changes the value to the right of `=`, so alignment is unaffected. No reflow expected.

---

### Task 1: Reference UI defaults (`RTXPTReferenceUIState`)

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp:83` (`NEEType`)
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp:86` (`NEEMISType`)

- [ ] **Step 1: Confirm the current (wrong) values are present (red)**

Run:
```bash
rg -n "NEEType\s+= 1;|NEEMISType\s+= 0;" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
```
Expected: two matches — `NEEType ... = 1;` on line 83 and `NEEMISType ... = 0;` on line 86.

- [ ] **Step 2: Change `NEEType` default `1` → `2`**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, replace the single line:

Old:
```cpp
    int                        NEEType                            = 1; // Phase R3 (G5): 0=Uniform, 1=Power+, 2=NEE-AT.
```
New:
```cpp
    int                        NEEType                            = 2; // Phase R3 (G5): 0=Uniform, 1=Power+, 2=NEE-AT.
```

- [ ] **Step 3: Change `NEEMISType` default `0` → `1`**

In the same file, replace the single line:

Old:
```cpp
    int                        NEEMISType                         = 0; // Phase R3 (G5): 0=Full, 1=ApproxInRealtime, 2=Approximate (deferred).
```
New:
```cpp
    int                        NEEMISType                         = 1; // Phase R3 (G5): 0=Full, 1=ApproxInRealtime, 2=Approximate (deferred).
```

- [ ] **Step 4: Verify the new values (green) and cross-check upstream**

Run:
```bash
rg -n "NEEType\s+= 2;|NEEMISType\s+= 1;" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
```
Expected: `NEEType ... = 2;` (line 83) and `NEEMISType ... = 1;` (line 86).

Confirm against upstream (optional, should print the cited target values):
```bash
rg -n "NEEType\s+= 2" "D:/RTXPT-fork/Rtxpt/SampleCommon/CommandLine.h"
rg -n "NEEMISType\s+= 1" "D:/RTXPT-fork/Rtxpt/SampleUI.h"
```
Expected: `CommandLine.h:42 ... NEEType = 2` and `SampleUI.h:156 ... NEEMISType = 1`.

- [ ] **Step 5: Commit (inside the submodule)**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp
git -C DiligentSamples commit -m "feat(rtxpt): sync reference UI NEEType/NEEMISType defaults with upstream"
```

---

### Task 2: Realtime NRD defaults (`RTXPTRealtimeSettings` + nested NRD structs)

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp:158` (`NRDMethod`)
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp:108` (`RTXPTNrdReblurUiSettings::MaxFastAccumulatedFrameNum`)
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp:109` (`RTXPTNrdReblurUiSettings::HistoryFixFrameNum`)
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp:124` (`RTXPTNrdRelaxUiSettings::HistoryFixFrameNum`)

- [ ] **Step 1: Confirm the current (wrong) values are present (red)**

Run:
```bash
rg -n "NRDMethod\s+= RTXPTNrdMethod::RELAX;|MaxFastAccumulatedFrameNum\s+= 0;|HistoryFixFrameNum\s+= 0;" DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp
```
Expected: four matches — `NRDMethod = RTXPTNrdMethod::RELAX;` (line 158), `MaxFastAccumulatedFrameNum = 0;` (line 108), and `HistoryFixFrameNum = 0;` on both line 109 (REBLUR) and line 124 (RELAX).

- [ ] **Step 2: Change `NRDMethod` default `RELAX` → `REBLUR`**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`, replace the single line:

Old:
```cpp
    RTXPTNrdMethod           NRDMethod                               = RTXPTNrdMethod::RELAX;
```
New:
```cpp
    RTXPTNrdMethod           NRDMethod                               = RTXPTNrdMethod::REBLUR;
```

- [ ] **Step 3: Change REBLUR `MaxFastAccumulatedFrameNum` `0` → `6`**

In the `RTXPTNrdReblurUiSettings` struct, replace the single line:

Old:
```cpp
    Uint32                                MaxFastAccumulatedFrameNum    = 0;
```
New:
```cpp
    Uint32                                MaxFastAccumulatedFrameNum    = 6;
```

- [ ] **Step 4: Change REBLUR `HistoryFixFrameNum` `0` → `3`**

In the `RTXPTNrdReblurUiSettings` struct, replace the single line (note the column alignment — this is the REBLUR struct's spacing):

Old:
```cpp
    Uint32                                HistoryFixFrameNum            = 0;
```
New:
```cpp
    Uint32                                HistoryFixFrameNum            = 3;
```

- [ ] **Step 5: Change RELAX `HistoryFixFrameNum` `0` → `3`**

In the `RTXPTNrdRelaxUiSettings` struct, replace the single line (note this is the RELAX struct's wider spacing, distinct from Step 4's line):

Old:
```cpp
    Uint32                                HistoryFixFrameNum                 = 0;
```
New:
```cpp
    Uint32                                HistoryFixFrameNum                 = 3;
```

- [ ] **Step 6: Verify the new values (green) and cross-check upstream**

Run:
```bash
rg -n "NRDMethod\s+= RTXPTNrdMethod::REBLUR;|MaxFastAccumulatedFrameNum\s+= 6;|HistoryFixFrameNum\s+= 3;" DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp
```
Expected: four matches — `NRDMethod = RTXPTNrdMethod::REBLUR;` (line 158), `MaxFastAccumulatedFrameNum = 6;` (line 108), and `HistoryFixFrameNum = 3;` on both line 109 and line 124.

Confirm there are no remaining `= 0;` NRD fall-through fields:
```bash
rg -n "MaxFastAccumulatedFrameNum\s+= 0;|HistoryFixFrameNum\s+= 0;" DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp
```
Expected: no matches.

Confirm against upstream (optional, should print the cited target values):
```bash
rg -n "NRDMethod = NrdConfig::DenoiserMethod::REBLUR" "D:/RTXPT-fork/Rtxpt/SampleUI.h"
rg -n "maxFastAccumulatedFrameNum = 6|historyFixFrameNum = 3" "D:/RTXPT-fork/External/Nrd/Include/NRDSettings.h"
```
Expected: `SampleUI.h:293` REBLUR method, and `NRDSettings.h:241` (`maxFastAccumulatedFrameNum = 6`), `:249` and `:346` (`historyFixFrameNum = 3`).

- [ ] **Step 7: Commit (inside the submodule)**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRealtimeSettings.hpp
git -C DiligentSamples commit -m "feat(rtxpt): sync realtime NRD method and denoiser fall-through defaults with upstream"
```

---

### Task 3: Final verification and submodule pointer bump

**Files:**
- Verify only: both modified headers.
- Modify (parent repo pointer): `DiligentSamples` submodule reference.

- [ ] **Step 1: Review the full submodule diff**

Run:
```bash
git -C DiligentSamples diff HEAD~2 -- Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTRealtimeSettings.hpp
```
Expected: exactly six changed lines — `NEEType 1→2`, `NEEMISType 0→1`, `NRDMethod RELAX→REBLUR`, REBLUR `MaxFastAccumulatedFrameNum 0→6`, REBLUR `HistoryFixFrameNum 0→3`, RELAX `HistoryFixFrameNum 0→3`. No other changes (no whitespace/alignment reflow).

- [ ] **Step 2: Confirm copyright year needs no bump**

Run:
```bash
rg -n "Copyright 2026" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp
```
Expected: both files already show `Copyright 2026 Diligent Graphics LLC`. Current year is 2026, so no copyright-date edit is required (per project convention in `CLAUDE.md`). If either file shows an older year, update only that header's year and amend the relevant commit.

- [ ] **Step 3: (Optional, user-initiated) Lightweight compile check**

Do not run automatically — the user's standing instruction is not to run build commands unprompted. If the user asks for a compile, build just the RTXPT sample target with the existing configured build, e.g.:
```bash
cmake --build build/x64/Debug --config Debug --target RTXPT
```
Expected: compiles clean (headers are include-only default changes). If no build is configured or it is too costly, report that the compile was **not run** and that verification rests on the static `rg`/`git diff` checks above.

- [ ] **Step 4: Bump the submodule pointer in the parent repo**

```bash
git add DiligentSamples
git commit -m "chore(samples): update RTXPT submodule"
```
Expected: parent repo records the new `DiligentSamples` commit (matching the existing `chore(samples): update RTXPT submodule` history convention).

---

## Self-Review

- **Spec coverage:** Every field in the spec's "Expected Default Alignment" that requires a change is covered — reference UI `NEEType` (Task 1), `NEEMISType` (Task 1); realtime `NRDMethod` (Task 2) and the three NRD fall-through fields `ReblurSettings.MaxFastAccumulatedFrameNum`, `ReblurSettings.HistoryFixFrameNum`, `RelaxSettings.HistoryFixFrameNum` (Task 2). Fields the spec says to leave unchanged (DLSS-RR/`RealtimeAA` guard, `EnvironmentMapEnabled`, `DenoisingGuideDebugView`, all already-aligned fields) are intentionally untouched. NEE-AT safety (spec Locked Decision 5) is honored — Task 1 sets `NEEType = 2`, which the spec confirms is an implemented path.
- **Placeholder scan:** No TBD/TODO/"handle edge cases"/vague steps — every edit step shows exact old/new lines and every verify step shows exact commands with expected output.
- **Type/name consistency:** Field names, struct names, and enum value (`RTXPTNrdMethod::REBLUR`) match the structs as defined in `RTXPTRealtimeSettings.hpp` and `RTXPTSample.hpp`; submodule commit/path usage (`git -C DiligentSamples`, `Samples/RTXPT/src/...`) is consistent across all tasks.
