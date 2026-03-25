# BetterBusStopPosition — Integration vs. Standalone Source

## Major Architectural Differences

The IPT3 integration of BBSP differs fundamentally from the original standalone mod.

### 1. Patch Type: Transpiler → Postfix

| Aspect | Original BBSP | IPT3 Integration |
|--------|---------------|-----------------|
| Patch Type | Transpiler on `BusAI.CalculateSegmentPosition` | Postfix on `BusAI.CalculateSegmentPosition` |
| Patch Point | Rewrites IL to replace `NetLane.CalculateStopPositionAndDirection` call | Modifies output parameters `pos`, `dir` after vanilla method completes |
| Method Replaced | `NetLane.CalculateStopPositionAndDirection` call | (none; vanilla call proceeds unchanged) |

**Consequence**: Original prevented vanilla SmootherStep tapering by bypassing the call entirely. IPT runs vanilla code first, then recalculates output — functionally equivalent for IPT integration, but execution differs.

### 2. Logic Modes: Single → Three Configurable Modes

**Original BBSP**: Always applies the offset calculation and calls vanilla method with modified offset.

**IPT3 Integration**: Runtime-selectable modes via `ImprovedPublicTransport.Settings.Settings.BbspLogic`:
- **Disabled** — Vanilla behavior (new in IPT3)
- **Enabled** — Fixed BBSP behavior (offset calculation + vanilla method call)

### 3. Vehicle Flags Condition

| Version | Condition |
|---------|-----------|
| Original | `(vehicleData.m_flags & Vehicle.Flags.Leaving) == 0` |
| IPT3 | `(vehicleData.m_flags & (Vehicle.Flags.Leaving \| Vehicle.Flags.Arriving)) == 0` (in postfix gate) |
| IPT3 BBSPMode | `(vehicleData.m_flags & Vehicle.Flags.Leaving) == 0` (in inner logic) |

**Original** blocks positioning changes only when vehicle has Leaving flag.  
**IPT3 Postfix Gate** blocks the entire postfix if vehicle is neither Leaving nor Arriving (stricter gate).  
**IPT3 BBSPMode** replicates original Leaving check inside the mode.

**Consequence**: IPT3 skips all processing if vehicle is not actively boarding/alighting. Original processes if not leaving. This is a **functional difference** for idle vehicles.

### 4. Null Safety

| Version | m_generatedInfo Check |
|---------|----------------------|
| Original | None (assumes non-null) |
| IPT3 | `vehicleData.Info.m_generatedInfo != null ? vehicleData.Info.m_generatedInfo.m_size.z : 0f` |

**Consequence**: IPT3 is safer; original could crash on vehicles with null Info.m_generatedInfo.

### 5. Options Integration

| Aspect | Original | IPT3 |
|--------|----------|------|
| Master Enable | Not present | `BbspLogic` option (0=Disabled, 1=BBSPMode, default) |
| Options Framework | Standalone mod none | Integrated into IPT Options (OptionsWrapper) |
| UI Configuration | None (always on) | Settable from IPT Options panel |

### 6. TrolleybusAI Support

**Both versions**: Patch `TrolleybusAI.CalculateSegmentPosition` identically.

**Original**: Uses same transpiler as BusAI.  
**IPT3**: Uses same postfix pattern, calls shared `BusAI_Patch.CalculateModifiedStopPosition`.

---

## Functional Equivalence Assessment

### When IPT3 is in "BBSPMode" mode:
- ✅ Offset calculation is **identical** to original
- ✅ Final method call (vanilla `CalculateStopPositionAndDirection`) is **identical**
- ⚠️ Execution path differs (postfix recalc vs. transpiler bypass), but **result is mathematically equivalent**
- ✅ TrolleybusAI behavior **identical**

### When IPT3 is in "Disabled" mode:
- ✅ **Identical to vanilla behavior** (no positioning changes)

### Where they differ:
1. **Postfix Gate** — IPT3's `(Leaving | Arriving)` check is stricter than original's `!Leaving` check
   - **Test case**: Idle vehicle at stop → Original would apply offset, IPT3 would skip
   - **Real-world impact**: Minimal (idle vehicles rarely need stop positioning adjustments)

2. **Null Safety** — IPT3 handles null m_generatedInfo, original assumes non-null
   - **Test case**: Vehicle with uninitialized Info → Original might crash, IPT3 uses fallback (0.0f)

3. **UpdatedLogic Mode** — Disabled in IPT3 (commented code), would use direct Bezier calculation
   - **Not yet enabled** pending testing/validation

---

## Recommended Next Steps

1. **Test idle vehicle positioning** — Verify original vs. IPT3 behavior for stationary vehicles at stops
2. **Enable UpdatedLogic mode** — The Bezier-direct method (commented) may improve smoothing once validated
3. **Simplify Postfix Gate** — Consider reverting to original `!Leaving` check to match more closely (if idle vehicle testing shows no issue)

---

## Code Summaries

### Original BBSP Logic
```csharp
// Transpiler replaces CalculateStopPositionAndDirection call
// CalculateSegmentPosition_Hook(lane, laneOffset, vehicleID, vehicleData, laneInfo, flags):
if (vehicle not leaving AND laneLength ≥ 1):
    margin = laneLength / 6
    vehicleLength = vehicleData.Info.m_generatedInfo.m_size.z
    newStopOffset = 1 - (margin + vehicleLength/2) / laneLength
    if newStopOffset ≥ 0.5:
        account for lane/segment invert flags
        laneOffset *= 2 * newStopOffset
return laneOffset (modified or original)
// Passed to vanilla CalculateStopPositionAndDirection(laneOffset, stopOffset, ...)
```

### IPT3 BBSPMode (functionally equivalent)
```csharp
// Postfix modifies pos/dir after vanilla method
if (vehicle not Leaving/Arriving):  // stricter gate!
    modifiedOffset = laneOffset
    if (vehicle not leaving AND laneLength ≥ 1):
        margin = laneLength / 6
        vehicleLength = vehicleData.Info?.m_generatedInfo?.m_size.z ?? 0f  // null-safe
        newStopOffset = 1 - (margin + vehicleLength/2) / laneLength
        if newStopOffset ≥ 0.5:
            account for lane/segment invert flags
            modifiedOffset *= 2 * newStopOffset
    lane.CalculateStopPositionAndDirection(modifiedOffset, stopOffset, out pos, out dir)
// pos/dir now contain result with modified offset
```

Root Cause: SmootherStep Mismatch

Vanilla `NetLane.CalculateStopPositionAndDirection` uses a `SmootherStep(0.5, 0, |laneOffset - 0.5|)` curve to apply the lateral curb offset (`stopOffset`). This curve peaks at exactly 1.0 when `laneOffset = 0.5` (the vanilla lane-centre stop) and tapers to 0 at both ends.

BBSP shifts the stop forward — `targetOffset` now ranges from ~0.6 to ~0.8 depending on vehicle length. At those positions, `SmootherStep` yields only ~0.5–0.7, so the vanilla call applies only a partial curb pull. The result is a visible arc/swoop: buses steer toward the curb as they cross the 0.5 mark, then drift away as the lerp value falls back toward 0 at the actual stop point.

## Solution: Replace the Call Entirely

Instead of patching around the SmootherStep value, `CalculateStopPositionAndDirection` is **replaced entirely** by a transpiler that calls `CalculateModifiedStopPosition`.

The replacement:
1. Computes `pos` and `dir` directly from the Bezier at `targetOffset` (skipping SmootherStep).
2. Applies the full `stopOffset` lateral displacement unconditionally at the actual stop point via `Vector3.Cross(Vector3.up, dir).normalized * stopOffset`.

Because the steering physics already produces a smooth physical approach, no mathematical smoothing of the curb pull is needed — the bus drives smoothly to its curbside position naturally.

## Transpiler Changes (`BusAI_Patch.CalculateSegmentPosition`)

The transpiler makes three insertions into the IL of `BusAI.CalculateSegmentPosition`:

1. **Capture `NetSegment.Flags`** — inserts `Dup` + `Stloc_S` immediately after the `Ldfld m_flags` load so the flags value is available later.

2. **Inject extra arguments** — after the `laneOffset * 0.003921569f` multiply (which leaves `[ref NetLane] [laneOffset]` on the stack), inserts:
   - `Ldarg_1` — vehicleId (`ushort`)
   - `Ldarg_2` — vehicleData (`ref Vehicle`)
   - `Ldloc_2` — lane (`NetInfo.Lane`, pre-verified as local #2)
   - `Ldloc_S localFlags` — the captured flags

3. **Replace the call** — rewrites the `Call CalculateStopPositionAndDirection` instruction to `Call CalculateModifiedStopPosition`.

`TrolleybusAI_Patch` re-uses the same transpiler unchanged.

## `CalculateModifiedStopPosition` Logic

```
targetOffset = laneOffset  // fallback: vanilla position

if laneLength ≥ 1 AND vehicle is not Leaving:
    margin = laneLength / 6
    vehicleLength = vehicleData.Info.m_generatedInfo.m_size.z
    newStopOffset = 1 - (margin + vehicleLength/2) / laneLength

    if newStopOffset ≥ 0.5:
        account for lane/segment invert flags
        targetOffset = adjusted position along Bezier

pos = bezier.Position(targetOffset)
dir = bezier.Tangent(targetOffset)

if stopOffset ≠ 0:
    pos += Cross(up, dir).normalized * stopOffset   // full lateral displacement
```

## Files Changed

| File | Change |
|------|--------|
| `Integration/BetterBusStopPosition/BusAI.cs` | Transpiler rewritten (+62 −74 net); `CalculateModifiedStopPosition` replaced (+11 −19 net) |
