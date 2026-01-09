# Startup Performance Analysis (TVApplication)

**Scope**: `app/src/main/java/com/telekom/atvretail/TVApplication.kt`  
**Goal**: Reduce startup time by deferring non‑critical WorkManager scheduling and Lazy.get() init chains with safe thread usage.

---

## Issues + Actions (Merged)

- **Startup WorkManager pressure** → Defer non‑critical workers (keymap, home prefetch, log cleanup) to post‑startup or login; keep CMS/app status/security at startup. Risk: deferring too long delays content availability; choose minimal delay.
- **Work rescheduling churn** → Prefer `ExistingWorkPolicy.KEEP` for periodic work where “latest schedule” isn’t critical; retain `REPLACE` only for CMS‑driven interval changes.
- **Unique‑work name collision** (`initializeCmsKeyMapWorker()` uses `"bookmark-init"`) → Rename to a dedicated keymap name (e.g., `"cms-keymap-init"`).
- **Duplicate scheduling** (startup vs login flows) → Centralize scheduling to one path or enforce unique names + `KEEP`.

- **Repeated `Lazy.get()` calls in `onCreate()`** → Cache once and reuse (e.g., `mEnvironment.get()`).
- **`preferences.get()` + CMS config access on main** → Batch in a single `Dispatchers.IO` coroutine; cache `prefs` and `cmsConfig` locally.
- **`setImageCacheClearTime()` sync call** → Defer on `Dispatchers.IO` with a short delay.

---

## Safety Notes (Lazy + Worker Changes)

- `setImageCacheClearTime()` only sets a SharedPreferences timestamp if unset → safe to defer.
- `LanguageMappingsWorker.runWorker()` only enqueues WorkManager → safe on `Dispatchers.IO`.
- `LanguageMappingsWorker.initializeFindabilityFeature()` mixes IO + resource access:
  - `setFindabilityCmsConfig()` already uses `Dispatchers.IO` internally.
  - `setFindabilityViewConfigData()` uses `languageService.t()` (Resources) → keep on main thread or split into a main‑thread call.
- `preferences.get()` is thread‑safe (SharedPreferences) → safe on `Dispatchers.IO`.
- `mCmsDataManager.get().cmsConfig` is read‑only after init → safe to cache.

---

## References

- `app/src/main/java/com/telekom/atvretail/TVApplication.kt`
