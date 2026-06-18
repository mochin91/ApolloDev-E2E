---
name: Filament
slug: filament
version: 1.0.0
description: Build Filament 4 and Filament 5 admin panels for Laravel. Covers Resources, Forms, Tables, Actions, Widgets, Panels, and breaking changes between versions.
metadata: {"clawdbot":{"emoji":"🎨","requires":{"bins":["php","composer"]},"os":["linux","darwin","win32"]}}
---

## Quick Reference

| Topic | File |
|-------|------|
| Filament 4 — Resources, Forms, Tables, Actions, Widgets, Auth | `filament4.md` |
| Filament 5 — Breaking changes, new API, migration guide | `filament5.md` |

## Version Detection

Check `composer.json` for `filament/filament` version:
- `^4.x` → refer to `filament4.md`
- `^5.x` → refer to `filament5.md`

```bash
composer show filament/filament | grep versions
```

## Critical Rules (Both Versions)

- Always use `->relationship()` on Select fields for BelongsTo — not `->options()`
- `->required()` on form fields ≠ required at DB level — set both separately
- Gate/Policy authorization via `->can()` or `canCreate()/canEdit()/canDelete()` on Resource
- Never put Eloquent queries inside form field closures that run on every render — use `->options(fn() => ...)` with eager loading
- `Tables\Columns\TextColumn::make('relation.column')` auto-loads relation — still causes N+1 without `->with()` on the query
- Widgets are Livewire components — any state that changes must trigger `$this->js(...)` or use wire:model
