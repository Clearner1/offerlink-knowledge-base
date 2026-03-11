# OfferLink Field Mapping Shadowing Bug Pattern

## Problem Summary

`FormFiller.matchResumeField()` has two layers of field matching:
1. **Server-side `fieldMappings`** (cached from backend API) — checked FIRST
2. **Local hardcoded maps** (`basicInfoMap`, `educationMap`, etc.) — checked SECOND

When a server-side mapping matches a label but points to a **non-existent field** in the resume, the field is "skipped" because `getResumeValue()` returns `undefined`. The local maps (which have the correct mapping) are never reached.

## Root Cause

```javascript
// Server mappings — NO value existence check (BUG)
if (aliases.some(alias => areConceptuallySimilar(labelText, alias))) {
    return `${section}.${fieldName}`;  // Returns immediately, even if value is undefined!
}

// Local maps — HAS value existence check (CORRECT)
const value = this.getResumeValue(field);
if (value !== null && value !== undefined && value !== '') {
    return field;  // Only returns if value exists
}
```

## Concrete Example

| Form Label | Server Mapping | Value | Local Mapping | Value |
|---|---|---|---|---|
| 最高学历 | `education.labels` | `undefined` ❌ | `basicInfo.highestEducation` | `"硕士"` ✅ |
| 最高学位 | `basicInfo.highestEducation` | `"硕士"` (wrong field!) | `basicInfo.highestDegree` | `"硕士"` |

Result: "最高学历" is skipped (undefined), "最高学位" gets the wrong value source (highestEducation instead of highestDegree).

## Fix

Added value existence check to server `fieldMappings` matching, consistent with local maps:

```javascript
if (matched) {
    const fieldPath = `${section}.${fieldName}`;
    const value = this.getResumeValue(fieldPath);
    if (value !== null && value !== undefined && value !== '') {
        return fieldPath;
    }
    // Fall through to local maps
}
```

## Debugging Methodology

When a dropdown field is "skipped" during autofill:

1. **Add debug logging** to `fillAllFields` to print `labelText`, `fieldType`, `resumeField`, and `value` for every field
2. **Check the mapping** — if `resumeField` points to an unexpected path (like `education.labels` instead of `basicInfo.highestEducation`), the issue is in `matchResumeField`
3. **Check if server fieldMappings are interfering** — `this.fieldMappings` (from `chrome.storage.local.get(['fieldMappings'])`) is checked BEFORE local maps. Stale or incorrect server mappings will shadow correct local ones
4. **Always ensure value existence checks** — any mapping layer that returns a field path should verify the path resolves to an actual value in the resume

## Related Shadowing Bug (nativePlace → birthOrigin)

A similar shadowing pattern was fixed earlier for 籍贯 (native place):
- `basicInfo.nativePlace` matched first but had no value (resume only had `birthOrigin`)
- Fix: the local map loop now continues searching when matched field has no value

## Key Files

- `src/content/formFiller.js` — `matchResumeField()` (lines ~478-495 for server mappings, ~617-631 for local maps)
- `src/content/index.js` — loads `fieldMappings` from `chrome.storage.local` (line ~157)
- `src/content/utils/textSimilarity.js` — `areConceptuallySimilar()` used for fuzzy matching
