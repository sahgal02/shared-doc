# Startup Performance Improvements (TVApplication)

**Scope**: `app/src/main/java/com/telekom/atvretail/TVApplication.kt`  
**Goal**: Reduce startup time by deferring non‑critical WorkManager scheduling and `Lazy.get()` init chains, while keeping thread safety.

---

## WorkManager: What/Why + Before/After

### 1) Defer non‑critical worker scheduling
**Why**: Many workers are enqueued at startup; only a subset are needed for first frame. Deferring non‑critical ones reduces main startup pressure.

**Before**
```kotlin
// onCreate(): all scheduled at startup
initializeCmsWorker()
initialiseRootCategoriesPeriodicWorkRequest()
initialiseAccountCallOneTimeWorkRequest()
initialiseAppStatusPeriodicWorkRequest()
initialiseSecuritySettingsOneTimeWorkRequest()
if (mEnvironment.get().isManagedDevice) {
    initialiseMKLogsDeletionWorkerRequest()
    initializeCmsKeyMapWorker()
    fetchKeyMap()
    initializeKeyMapWorker()
}
```

**After (proposed)**
```kotlin
// startup-critical only
initializeCmsWorker()
initialiseAppStatusPeriodicWorkRequest()
initialiseSecuritySettingsOneTimeWorkRequest()

// defer non-critical
coroutineScope.launch(Dispatchers.IO) {
    delay(200)
    initialiseRootCategoriesPeriodicWorkRequest()
    initialiseAccountCallOneTimeWorkRequest()
    if (environment.isManagedDevice) {
        initialiseMKLogsDeletionWorkerRequest()
        initializeCmsKeyMapWorker()
        fetchKeyMap()
        initializeKeyMapWorker()
    }
}
```
**Impact**: Reduces synchronous startup work; moves non‑critical enqueues off the critical path.

### 2) Prefer `ExistingWorkPolicy.KEEP` for stable periodic work
**Why**: `REPLACE` forces rescheduling on each launch and creates churn.

**Before**
```kotlin
workManager.enqueueUniquePeriodicWork(
    WORK_NAME,
    ExistingPeriodicWorkPolicy.REPLACE,
    request
)
```

**After (proposed)**
```kotlin
workManager.enqueueUniquePeriodicWork(
    WORK_NAME,
    ExistingPeriodicWorkPolicy.KEEP,
    request
)
```
**Impact**: Avoids rescheduling churn; preserves existing periodic cadence.

### 3) Fix unique‑work name collision
**Why**: `initializeCmsKeyMapWorker()` uses `"bookmark-init"` as its unique name, which can replace unrelated work.

**Before**
```kotlin
enqueueUniqueWork("bookmark-init", ...)
```

**After (proposed)**
```kotlin
enqueueUniqueWork("cms-keymap-init", ...)
```
**Impact**: Prevents cross‑worker replacement.

---

## Lazy.get(): What/Why + Before/After

### 4) Cache `mEnvironment.get()` once
**Why**: `mEnvironment.get()` is called 3 times in `onCreate()`. Cache once to avoid repeated lazy init chain calls.

**Before**
```kotlin
LogFramework.init(LoggerConfig(ConsoleLogger(), mEnvironment.get().isDebug))
if (!mEnvironment.get().isManagedDevice) { ... }
if (mEnvironment.get().isManagedDevice) { ... }
```

**After (proposed)**
```kotlin
val environment = mEnvironment.get()
LogFramework.init(LoggerConfig(ConsoleLogger(), environment.isDebug))
if (!environment.isManagedDevice) { ... }
if (environment.isManagedDevice) { ... }
```
**Impact**: Removes 2 extra `Lazy.get()` calls on startup.

### 5) Batch preferences + CMS config access
**Why**: `preferences.get()` called 4 times on main thread + `mCmsDataManager.get()` twice for the same config.

**Before**
```kotlin
preferences.get().putBoolean(... (mCmsDataManager.get().cmsConfig as CmsConfig) ...)
preferences.get().putBoolean(... (mCmsDataManager.get().cmsConfig as CmsConfig) ...)
if (!LanguageMappingsWorker.hasLanguageMappingsDownloadWorkerRan(preferences.get())) {
    LanguageMappingsWorker.runWorker()
}
LanguageMappingsWorker.initializeFindabilityFeature(preferences.get())
```

**After (proposed)**
```kotlin
coroutineScope.launch(Dispatchers.IO) {
    delay(100)
    val prefs = preferences.get()
    val cmsConfig = (mCmsDataManager.get().cmsConfig as CmsConfig)

    prefs.putBoolean(... cmsConfig.settingConfig?.security?.ageRating?.enableVisualIndicator == true)
    prefs.putBoolean(... cmsConfig.settingConfig?.security?.warningIcon?.enableVisualIndicator == true)

    if (!LanguageMappingsWorker.hasLanguageMappingsDownloadWorkerRan(prefs)) {
        LanguageMappingsWorker.runWorker()
    }

    withContext(Dispatchers.Main) {
        LanguageMappingsWorker.setFindabilityViewConfigData()
    }
    LanguageMappingsWorker.setFindabilityCmsConfig(prefs)
}
```
**Impact**: Removes 4 `Lazy.get()` calls from the main thread and prevents repeated CMS config access.

### 6) Defer `setImageCacheClearTime()`
**Why**: `setImageCacheClearTime()` calls `imageCachingRepositoryImpl.get()` synchronously.

**Before**
```kotlin
setImageCacheClearTime()
```

**After (proposed)**
```kotlin
coroutineScope.launch(Dispatchers.IO) {
    delay(50)
    setImageCacheClearTime()
}
```
**Impact**: Removes a lazy init from the critical path.

---

## Safety Review (Why Deferral Is Safe)

- `setImageCacheClearTime()` only sets a SharedPreferences timestamp if unset → safe to defer.
- `LanguageMappingsWorker.runWorker()` only enqueues WorkManager → safe on `Dispatchers.IO`.
- `LanguageMappingsWorker.initializeFindabilityFeature()` mixes IO + resource access:
  - `setFindabilityCmsConfig()` already uses `Dispatchers.IO`.
  - `setFindabilityViewConfigData()` uses `languageService.t()` (Resources) → keep on main thread.
- `preferences.get()` is thread‑safe (SharedPreferences) → safe on `Dispatchers.IO`.
- `mCmsDataManager.get().cmsConfig` is read‑only after init → safe to cache.

---

## Expected Impact

- **Total**: ~50–100ms improvement in `onCreate()` time.
- **Breakdown**:
  - Preferences batching: ~30–60ms
  - `mEnvironment` caching: ~10–20ms
  - Image cache deferral: ~5–15ms

---

## References

- `app/src/main/java/com/telekom/atvretail/TVApplication.kt`
