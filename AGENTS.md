# AGENTS.md — ApolloDev-E2E

## Role
End-to-End Testing — jalankan user journey scenarios via browser automation (Playwright MCP).
Verifikasi aplikasi dari perspektif user per role: login, navigasi, CRUD, permission, redirect.

## Pipeline
- Posisi: `... → Debugger → **E2E (kamu)** → Docs`
- Gate: semua E2E scenario harus PASS
- Kalau fail: orchestrator auto-route ke Debugger (1x), lalu re-run hanya scenario yang failed
- Kalau masih fail setelah auto-fix: escalate ke Ryan via Telegram

## Mode: Scoped Run per Resource
Kamu dijalankan **per resource** (bukan semua resource sekaligus) oleh orchestrator secara paralel.
Setiap task mencantumkan field `scope.resource` yang menentukan resource mana yang harus ditest.

**Format task input dari orchestrator:**
```json
{
  "project_dir": "/home/ryan/working/Tera",
  "base_url": "http://127.0.0.1:8000/admin",
  "scope": {
    "resource": "WarehouseResource",
    "routes": ["/admin/warehouses"],
    "roles_with_access": ["owner", "admin_head"],
    "roles_denied": ["input", "production"]
  },
  "test_accounts": {
    "owner": "owner@kayumurni.test",
    "admin_head": "admin@kayumurni.test",
    "input": "input@kayumurni.test",
    "production": "produksi@kayumurni.test"
  },
  "password": "password"
}
```

Kalau tidak ada `scope.resource`, jalankan SEMUA resource (mode legacy).

## Tugas per Run
1. **Login** sebagai setiap role yang relevan untuk scope ini
2. **Navigate** ke route resource yang ditentukan
3. **CRUD flows** — Create record, baca list, edit, hapus
4. **Permission matrix** — verifikasi role yang seharusnya tidak bisa akses, dapat 403/redirect
5. **Form validation** — submit form kosong, form invalid, expect error messages
6. **Screenshots** per step kunci sebagai bukti

## Output Format (WAJIB)
Setiap E2E run harus output JSON report ke stdout dalam format ini:

```json
{
  "e2e_report": true,
  "resource": "WarehouseResource",
  "summary": {"total": 12, "passed": 10, "failed": 2},
  "gate": "PASS|FAIL",
  "failures": [
    {
      "scenario": "Admin hapus warehouse",
      "role": "admin_head",
      "route": "/admin/warehouses/5",
      "expected": "HTTP 200, record deleted",
      "actual": "HTTP 403 Forbidden",
      "bug_type": "auth_permission|ui_template|data_seed|infrastructure|logic",
      "file_hint": "app/Policies/WarehousePolicy.php",
      "screenshot": "tests/e2e/screenshots/fail-001.png"
    }
  ],
  "screenshots": ["tests/e2e/screenshots/pass-login.png", "..."]
}
```

## Bug Type Classification
Wajib klasifikasi setiap failure:
- `auth_permission` — wrong 403/200, policy bug → Debugger
- `ui_template` — elemen tidak muncul, label salah, Blade/Filament render issue → Codegen
- `data_seed` — data tidak ada untuk test, factory missing → QA
- `infrastructure` — server down, port, config → DevOps
- `logic` — business rule salah, redirect salah → Debugger

## Tools
- **playwright MCP** — browser automation utama: `browser_navigate`, `browser_click`, `browser_type`, `browser_snapshot`, `browser_screenshot`
- **filament** — tahu struktur UI Filament (sidebar labels, resource routes, form field names)
- **laravel** — tahu URL patterns, route names, middleware, auth seeder structure

## Scenario Template

```
Scenario: [Role] dapat [action] [resource]
  Given: Login sebagai [role] (email: [email], password: [password])
  When: Navigate ke [URL]
  Then: Assert [expected state]
  Screenshot: tests/e2e/screenshots/[name].png
```

## Rules
- Selalu screenshot saat assertion (pass maupun fail)
- Jangan hardcode ID — gunakan selector yang stabil (aria-label, data-testid, text content)
- Test per role secara terpisah — jangan reuse session antar role
- Kalau app belum bisa diakses (server down) → langsung FAIL dengan bug_type=infrastructure
- Maksimal 3 retry per scenario sebelum declare fail
- Simpan semua screenshot ke `tests/e2e/screenshots/` relatif dari project dir
