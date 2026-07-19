# R8 Mapping Upload Plugin

|  |  |
|---|---|
| **Module** | `pantrix-gradle-plugin` |
| **Type** | Build-time Gradle plugin (not a runtime collector) |
| **Plugin id** | `com.pantrix.gradle` |
| **Applies to** | `com.android.application`, **minified variants only** (`isMinifyEnabled = true`) |
| **Requires** | AGP 8.x / R8 8.0+ (reads R8's `# pg_map_hash` header) |
| **Companion to** | [`pantrix-crash`](../crash/uncaught-exception-collector.md) · [ANR](../crash/anr-collector.md) — makes release crash & ANR stacks readable |

## What it does

Release builds are shrunk and **obfuscated** by R8: class and method names become `a.b.c`. So a
crash captured by [`pantrix-crash`](../crash/uncaught-exception-collector.md) in a release build
arrives as an unreadable stack like `a.b.c(:1)` — you can see *that* it crashed, not *where*.

This plugin closes that gap **at build time**. For every minified variant it:

1. derives a stable **content UUID** (`debug_id`) from the R8 `mapping.txt`,
2. stamps that UUID into the app so every crash the SDK sends carries it, and
3. uploads the `mapping.txt` to the Pantrix backend.

When a release crash lands, the backend looks up the mapping by that UUID and turns
`a.b.c(:1)` back into `CheckoutViewModel.submit(CheckoutViewModel.kt:42)`.

## Why you need it

Without the plugin, `pantrix-crash` still records crashes and ANRs in release — but every frame
stays obfuscated, so you can't tell which class or line failed and the crash grouping is built
on meaningless names. Applying the plugin is what makes release crash reports **actionable**.
Debug builds aren't minified, so they're already readable and the plugin skips them.

## How it works

Three tasks are registered per minified variant (`<Variant>` = `Release`, `QaTest`, …):

| Task | What it does |
|---|---|
| `pantrixGenerateMappingUuid<Variant>` | Derives `debug_id = UUID.nameUUIDFromBytes(pg_map_hash)` from R8's `# pg_map_hash: SHA-256 …` header. **Content-addressed** — every distinct build produces a distinct id, so a mapping can never be confused with another build's. |
| `pantrixInjectDebugMeta<Variant>` | Stamps that id into the APK/AAB as `assets/pantrix-debug-meta.properties` (`com.pantrix.mapping.uuid=<debug_id>`). The SDK reads it at startup and tags every crash/ANR with it. |
| `pantrixUploadMapping<Variant>` | Uploads `mapping.txt` to `…/v1/ci/debug-files` (manifest → presigned PUT → commit). Runs after `assemble`/`bundle`, and only when a CI key is configured. |

> **Built-in safety guard.** Before uploading, `pantrixUploadMapping<Variant>` reads the
> `pantrix-debug-meta.properties` actually packaged into the app and **fails the build** if its
> UUID doesn't match the mapping being uploaded. A wrong or stale mapping can never ship
> silently — the failure names both UUIDs and tells you to do a clean build.

## Setup

**1. Make the plugin resolvable** — add the Pantrix Maven repo to `pluginManagement` in
`settings.gradle.kts` (the same auth-free repo the SDK uses; see
[SDK-INTEGRATION](../../../SDK-INTEGRATION.md)):

```kotlin
// settings.gradle.kts
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
        maven { url = uri("https://raw.githubusercontent.com/developersancho/pantrix-sdk-android-aar/maven-repo/") }
    }
}
```

**2. Apply it** in the app module:

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("com.pantrix.gradle") version "<version>"
}
```

**3. Configure `pantrix { }`** (see below). Then a normal `./gradlew assembleRelease` uploads
the mapping automatically.

## Configuration (`pantrix { }`)

| Option | Type | Default | Meaning |
|---|---|---|---|
| `apiUrl` | `String` | — | Backend base URL for the upload (`…/api`). **Required to upload**; injection works without it. |
| `apiKey` | `String` | — | **CI** API key (`px_…`, `key_type=CI`). Not the app's ingest token — that won't authorize uploads. |
| `autoUpload` | `Boolean` | `true` if `apiKey` is set | When `false`, the UUID is still injected but the mapping is **not** uploaded (so a laptop build never talks to prod). |
| `variantFilter { }` | block | — | Per-variant `enabled` / `apiUrl` / `apiKey` / `autoUpload` overrides. Unset fields fall back to the globals above. |

```kotlin
// app/build.gradle.kts
pantrix {
    // CI key: from the environment on CI; never commit the real key.
    val ciKey = providers.environmentVariable("PANTRIX_CI_KEY")
        .orElse(providers.gradleProperty("pantrixCiKey"))
    apiKey.set(ciKey)
    autoUpload.set(ciKey.map { it.isNotBlank() })

    // Send each variant's mapping to the SAME backend that variant's crashes go to.
    variantFilter {
        apiUrl = if (name == "release") {
            "https://api.pantrixapp.com/api"   // prod
        } else {
            "http://localhost:8099/api"        // test / qa
        }
    }
}
```

> ⚠️ **Match the mapping to the crashes.** A variant's mapping must be uploaded to the backend
> its crashes are reported to. If a release ships crashes to prod but its mapping goes to test,
> deobfuscation **breaks silently** — the crash is there, but unreadable. Change the URL in both
> places (this block **and** the SDK's runtime URL) together.

> 🔑 **Never commit the CI key.** Put `pantrixCiKey` in `~/.gradle/gradle.properties` or a CI
> secret (`PANTRIX_CI_KEY`). Mint CI keys from the dashboard (they are `key_type=CI`).

## What it produces

- **In the app:** `assets/pantrix-debug-meta.properties` containing `com.pantrix.mapping.uuid=<debug_id>`.
  (Escape hatch: if you disable the assets transform, the SDK also reads a
  `com.pantrix.mapping-uuid` manifest `<meta-data>`.)
- **In the backend:** a `PROGUARD` debug file stored under the project, keyed by `debug_id`.

## Verifying it worked

Watch the build log for these lines:

```
Pantrix: mapping debug_id = 4c80f720-…
Pantrix: injected pantrix-debug-meta.properties (com.pantrix.mapping.uuid=4c80f720-…)
Pantrix: verified shipped asset UUID == uploaded debug_id (4c80f720-…)
Pantrix: mapping committed — debug_id=4c80f720-…
```

- **Check the shipped APK:** `unzip -p app-release.apk assets/pantrix-debug-meta.properties`.
- **If the guard fails** (`SHIPPED asset UUID (…) != mapping being uploaded (…)`): the packaged
  app and the mapping disagree — do a **clean build** (`./gradlew clean assembleRelease`).
- **If nothing uploads:** no CI key is set (`autoUpload` is off) — the UUID is still injected,
  but you must upload the mapping for that build before its crashes can be deobfuscated.

## Privacy

Uploads **only** the R8 `mapping.txt` — a table that maps your obfuscated names back to the
original class / method / line names. It contains **no user data, no device data, and no source
code**. It is the same artifact you would upload to the Play Console for deobfuscation.
