# Pantrix Android SDK — Integration Guide

A complete, **self-contained** guide to integrating the **Pantrix Android SDK** — a self-hosted
observability SDK (analytics + performance + crash/ANR + HTTP) that ships as a single fused AAR
plus optional add-ons. Events are uploaded **only to the backend URL you configure** — never to
Pantrix or any third party. This single file is everything you need to integrate.

- **Current version:** `1.0.0-beta.8`
- **Group id:** `com.pantrix.analytics`
- **Min SDK:** 24 · **compile/target SDK:** 36 · **JVM:** 17

> **Important — add-ons need peer dependencies.** The add-on modules (Compose, Navigation, Ktor)
> and the built-in OkHttp tracking declare their third-party libraries as **`compileOnly`** so the
> AAR never bundles a second copy. That means **your app must already provide those libraries**
> (Compose, Navigation-Compose, Ktor, OkHttp, …). Each section below lists exactly what to add —
> if a peer is missing, the feature won't compile or simply won't activate.

---

## Contents

1. [Artifacts](#1-artifacts)
2. [Step 1 — Add the repository](#2-step-1--add-the-repository)
3. [Step 2 — Add the SDK](#3-step-2--add-the-sdk)
4. [Step 3 — Initialize](#4-step-3--initialize)
5. [`PantrixConfig` reference](#5-pantrixconfig-reference)
6. [API reference (`Pantrix`)](#6-api-reference-pantrix)
7. [Automatic data collection](#7-automatic-data-collection)
8. [HTTP tracking — OkHttp](#8-http-tracking--okhttp)
9. [HTTP tracking — Ktor](#9-http-tracking--ktor)
10. [HTTP tracking — other clients](#10-http-tracking--other-clients)
11. [Compose interaction tracking](#11-compose-interaction-tracking)
12. [Navigation-Compose screen tracking](#12-navigation-compose-screen-tracking)
13. [Navigation 3 screen tracking](#13-navigation-3-screen-tracking)
14. [Debug add-on — Inspector](#14-debug-add-on--inspector)
15. [Debug/QA add-on — Feedback](#15-debugqa-add-on--feedback)
16. [Debug add-on — Widget](#16-debug-add-on--widget)
17. [Storage encryption](#17-storage-encryption)
18. [R8 / ProGuard](#18-r8--proguard)
19. [Privacy & collected data](#19-privacy--collected-data)
20. [Recommended build-variant setup](#20-recommended-build-variant-setup)
21. [Troubleshooting](#21-troubleshooting)

---

## 1. Artifacts

Everything shares one version (`1.0.0-beta.8`). The **fused SDK** is the only required
dependency; the rest are opt-in add-ons. The right-hand column lists the **peer libraries you must
also have** (they're `compileOnly` in the SDK).

| Artifact | What it is | Typical wiring | You must also provide |
| --- | --- | --- | --- |
| `pantrix-sdk` | **Fused AAR** — the whole SDK (analytics + performance + crash/ANR + OkHttp tracking + native bridge). | `implementation` | — |
| `pantrix-inspector` | Debug-only on-device QA overlay. | `debugImplementation` | — |
| `pantrix-inspector-noop` | No-op twin of the inspector for release. | `releaseImplementation` | — |
| `pantrix-feedback` | Debug/QA bug-report UI (screenshot + annotate + email). | `debugImplementation` | — |
| `pantrix-feedback-noop` | No-op twin of feedback for release. | `releaseImplementation` | — |
| `pantrix-widget` | Debug-only home-screen App Widget (Glance) — glanceable counts, tap opens the inspector. | `debugImplementation` | Jetpack Glance |
| `pantrix-widget-noop` | No-op twin of the widget for release. | `releaseImplementation` | — |
| `pantrix-ktor` | Ktor `HttpClient` plugin → `Pantrix.trackHttp`. | `implementation` | Ktor client + engine |
| `pantrix-compose` | Compose interaction helpers (`Modifier.trackClick`, …). | `implementation` | Jetpack Compose |
| `pantrix-compose-navigation` | Auto screen tracking for Navigation-Compose. | `implementation` | Compose + Navigation-Compose |
| `pantrix-compose-navigation3` | Auto screen tracking for Navigation 3. | `implementation` | Compose + Navigation 3 |
| OkHttp tracking | **Built into `pantrix-sdk`** (no separate artifact). | — | OkHttp |

---

## 2. Step 1 — Add the repository

The SDK is published to an **auth-free** Maven repository served over `raw.githubusercontent.com`.
Add it in `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        // Pantrix SDK — auth-free, no credentials required.
        maven {
            url = uri("https://raw.githubusercontent.com/developersancho/pantrix-sdk-android-aar/maven-repo/")
        }
    }
}
```

---

## 3. Step 2 — Add the SDK

The fused SDK is the only thing required to start collecting:

```kotlin
// build.gradle.kts (app)
dependencies {
    val pantrix = "1.0.0-beta.8"

    // Required.
    implementation("com.pantrix.analytics:pantrix-sdk:$pantrix")

    // Recommended — debug-only QA overlay; release links the no-op twin.
    debugImplementation("com.pantrix.analytics:pantrix-inspector:$pantrix")
    releaseImplementation("com.pantrix.analytics:pantrix-inspector-noop:$pantrix")
}
```

Add-ons (HTTP / Compose / feedback) are wired in their own sections below, **each with the peer
dependencies you must add**. There's no version catalog assumption here — substitute your own
versions for the peer libraries (Compose, Navigation, Ktor, OkHttp).

---

## 4. Step 3 — Initialize

Initialize once in `Application.onCreate()`. It's cheap — storage and export run on background
threads; nothing heavy runs on the main thread.

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()

        Pantrix.init(
            context = this,
            config = PantrixConfig(
                token = "px_your_project_token",
                url = "https://ingest.your-backend.example.com",
            ) {
                enableLogging(BuildConfig.DEBUG)
            }
        )

        // Debug-only QA overlay (no-op in release via the noop twin).
        PantrixInspector.init(this, InspectorConfig(showFloatingButton = true))
    }
}
```

Identify the user once known:

```kotlin
Pantrix.setUser("user-123", mapOf("plan" to "pro", "email" to "user@example.com"))
```

> `autoStart` defaults to **true**, so collection begins at `init`. Set `autoStart(false)` to gate
> on consent, then call `Pantrix.start()` when the user opts in; `Pantrix.stop()` pauses collection.

---

## 5. `PantrixConfig` reference

`PantrixConfig` is built with the Kotlin DSL or the Java-friendly `PantrixConfig.Builder` —
`token` and `url` are required, every other option is optional:

```kotlin
// Kotlin DSL
val config = PantrixConfig(token = "px_…", url = "https://…") {
    trackHttpBody(true)
}

// Java-friendly Builder (equivalent)
val config = PantrixConfig.Builder("px_…", "https://…")
    .trackHttpBody(true)
    .build()
```

| Field | Type | Default | Purpose |
| --- | --- | --- | --- |
| `token` | `String` | — | Your project/ingest token. **Required** (Builder arg). |
| `url` | `String` | — | Your self-hosted ingestion endpoint. **Required** (Builder arg). |
| `enableLogging` | `Boolean` | `false` | Verbose SDK logcat output. |
| `autoStart` | `Boolean` | `true` | Begin collecting at `init`; `false` waits for `start()`. |
| `screenBlocklist` | `List<String>` | `[]` | Screen names to never track. |
| `httpUrlBlocklist` | `List<String>` | `[]` | URL substrings to never track. |
| `httpHeadersBlocklist` | `List<String>` | `[]` | Header names to drop (on top of sensible defaults). |
| `trackHttpHeaders` | `Boolean` | `false` | Capture request/response headers. |
| `trackHttpBody` | `Boolean` | `false` | Capture request/response bodies (review privacy first). |
| `storageEncryption` | `StorageEncryption` | `NONE` | Encryption-at-rest mode — see [§17](#17-storage-encryption). |
| `collectDeviceId` | `Boolean` | `true` | Read `ANDROID_ID` as `device.deviceId`; `false` uses `installId` only. |
| `retentionDays` | `Int` | `0` (off) | Delete data older than N days. |
| `maxStoredEvents` | `Int` | `0` (unbounded) | Cap stored events (sent ones pruned first). |
| `keepSentEvents` | `Boolean` | `true` | `false` drops events right after export. |
| `redactionKeys` | `List<String>` | `[]` | Extra body/URL value keys to mask. |
| `eventTypeBlocklist` | `List<String>` | `[]` | Event **types** to drop (e.g. `"interaction"`). |
| `eventNameBlocklist` | `List<String>` | `[]` | Event **names** to drop (e.g. `"cpu_usage"`). |

Built-in redaction already masks sensitive header names (`Authorization`, `Cookie`, …) and common
secret value keys (`password`, `token`, `api_key`, `cvv`, …) in bodies and URL query strings.

---

## 6. API reference (`Pantrix`)

All methods are static (`@JvmStatic`) and safe to call before `init` (they no-op until initialized).

```kotlin
// Lifecycle
Pantrix.init(context, config)
Pantrix.start()
Pantrix.stop()

// Custom events
Pantrix.trackEvent("checkout_started")
Pantrix.trackEvent("checkout_started", mapOf("cart_size" to 3, "currency" to "USD"))

// UI interactions — a first-class `interaction` category (filter via eventTypeBlocklist = ["interaction"])
Pantrix.trackInteraction(InteractionType.CLICK, mapOf("target" to "buy_button"))
// InteractionType: CLICK, LONG_CLICK, SCROLL, HOVER, FOCUS, DRAG

// Screens — automatic for Activities/Fragments; manual for custom flows
Pantrix.trackScreenView("Checkout")
Pantrix.trackComposeScreenView("Checkout")   // tag as a Compose destination

// Users
Pantrix.setUser("user-123", mapOf("plan" to "pro"))
Pantrix.setUserProperty("plan", "enterprise")
Pantrix.setUserProperties(mapOf("plan" to "enterprise", "region" to "eu"))
Pantrix.unsetUserProperty("plan")

// Handled exceptions (unhandled crashes/ANRs are captured automatically)
runCatching { riskyOperation() }
    .onFailure { Pantrix.trackException(it, mapOf("module" to "checkout")) }

// Helpers
Pantrix.isHttpBodyTrackingEnabled()   // true when trackHttpBody is on
Pantrix.getOkHttpEventCollector()     // for OkHttp wiring (§8)
```

**Manual HTTP** (for clients other than OkHttp/Ktor):

```kotlin
Pantrix.trackHttp(
    url = "https://api.example.com/v1/users",
    path = "/v1/users",
    method = "GET",
    startTime = SystemClock.elapsedRealtime(),   // use elapsedRealtime to avoid clock skew
    endTime = SystemClock.elapsedRealtime(),
    statusCode = 200,
    error = null,
    // requestHeaders / responseHeaders / requestBody / responseBody only when needed
    client = "my-http-client",
)
```

---

## 7. Automatic data collection

Once initialized, the SDK collects with **no extra code**: sessions (foreground/background),
Activity & Fragment screen views, cold/warm/hot launch timings, CPU & memory usage, crashes +
native ANRs, and device/build/network context attached to every event. The exact fields are listed
in [§19](#19-privacy--collected-data).

---

## 8. HTTP tracking — OkHttp

OkHttp tracking is **built into `pantrix-sdk`** — no extra artifact. You only need **OkHttp itself**
on your classpath (you almost certainly already have it):

```kotlin
dependencies {
    implementation("com.squareup.okhttp3:okhttp:<your-version>")
}
```

Wire the interceptor + event listener:

```kotlin
val client = OkHttpClient.Builder()
    .apply {
        // Returns null if OkHttp tracking isn't available — safe no-op.
        Pantrix.getOkHttpEventCollector()?.let { collector ->
            addInterceptor(PantrixOkHttpApplicationInterceptor(collector))
            eventListenerFactory(PantrixEventListenerFactory(collector = collector, delegate = null))
        }
    }
    .build()
```

Each call emits one HTTP event (method, url, status, duration, sizes). Headers/bodies are captured
only when `trackHttpHeaders` / `trackHttpBody` are on; sensitive headers and secret body/URL values
are redacted.

---

## 9. HTTP tracking — Ktor

Add the `pantrix-ktor` add-on **plus your own Ktor client core and an engine** (the add-on declares
Ktor `compileOnly`):

```kotlin
dependencies {
    implementation("com.pantrix.analytics:pantrix-ktor:1.0.0-beta.8")

    // Peer dependencies you must provide:
    implementation("io.ktor:ktor-client-core:<your-ktor-version>")
    implementation("io.ktor:ktor-client-okhttp:<your-ktor-version>")  // or any other Ktor engine
}
```

Install the plugin on your `HttpClient`:

```kotlin
val client = HttpClient(OkHttp) {
    install(PantrixKtor)   // forwards each call to Pantrix.trackHttp
}
```

Body capture follows `PantrixConfig.trackHttpBody` automatically — the plugin checks
`Pantrix.isHttpBodyTrackingEnabled()` and never buffers a body the SDK would drop.

---

## 10. HTTP tracking — other clients

For any other client, call `Pantrix.trackHttp(...)` from your interceptor/listener — signature in
[§6](#6-api-reference-pantrix). No add-on or peer dependency needed.

---

## 11. Compose interaction tracking

Add `pantrix-compose` **plus Jetpack Compose** (the module declares Compose `compileOnly`):

```kotlin
dependencies {
    implementation("com.pantrix.analytics:pantrix-compose:1.0.0-beta.8")

    // Peer dependencies you must provide (use a Compose BOM to align versions):
    implementation(platform("androidx.compose:compose-bom:<your-bom>"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.foundation:foundation")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:<your-version>")
}
```

Each helper emits an `interaction` event:

```kotlin
// A clickable modifier that also tracks the click (label + optional attributes + onClick):
Box(modifier = Modifier.trackClick("buy_button") { onBuy() }) {
    Text("Buy")
}

// Or wrap an existing click handler:
Button(onClick = trackedClick("buy_button") { onBuy() }) { Text("Buy") }

// Track a scroll/list state, or an interaction source, by name:
TrackScroll("feed", scrollState)             // ScrollState or LazyListState
TrackInteractions("buy_button", interactionSource)
```

---

## 12. Navigation-Compose screen tracking

Add `pantrix-compose-navigation` **plus Compose and Navigation-Compose** (declared `compileOnly`):

```kotlin
dependencies {
    implementation("com.pantrix.analytics:pantrix-compose-navigation:1.0.0-beta.8")

    // Peer dependencies you must provide:
    implementation(platform("androidx.compose:compose-bom:<your-bom>"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.navigation:navigation-compose:<your-version>")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:<your-version>")
}
```

Attach the tracker to your `NavController` so every destination is reported as a screen view —
both helpers are `@Composable`, call them inside your composition:

```kotlin
// Option A — observe an existing controller:
val navController = rememberNavController()
PantrixScreenTracking(navController)

// Option B — wrap at creation (returns the same controller with the listener attached):
val navController = rememberNavController().withPantrixNavigationListener()
```

> Without `androidx.navigation:navigation-compose` on your classpath this module **will not
> compile** — it's the peer the add-on builds on.

---

## 13. Navigation 3 screen tracking

Add `pantrix-compose-navigation3` **plus Compose and Navigation 3** (declared `compileOnly`):

```kotlin
dependencies {
    implementation("com.pantrix.analytics:pantrix-compose-navigation3:1.0.0-beta.8")

    // Peer dependencies you must provide:
    implementation(platform("androidx.compose:compose-bom:<your-bom>"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.navigation3:navigation3-runtime:<your-version>")
    implementation("androidx.navigation3:navigation3-ui:<your-version>")
}
```

Hook the tracker into your Nav3 back stack:

```kotlin
PantrixScreenNavTracking(backStack)   // reports each top destination as a screen view
```

---

## 14. Debug add-on — Inspector

An on-device QA overlay. Wire `debugImplementation(pantrix-inspector)` +
`releaseImplementation(pantrix-inspector-noop)` (no extra peer dependencies — it brings its own
Compose UI), then:

```kotlin
PantrixInspector.init(this, InspectorConfig(showFloatingButton = true))
// open programmatically:  PantrixInspector.show()
```

`InspectorConfig` (all optional):

| Field | Default | Purpose |
| --- | --- | --- |
| `enableShakeGesture` | `true` | Shake the device to open the inspector. |
| `shakeThreshold` | `2.5f` | Shake sensitivity (lower = more sensitive). |
| `showFloatingButton` | `false` | Draggable floating bubble + quick panel. |
| `showNotification` | `true` | Ongoing notification that opens the inspector (Android 13+ asks `POST_NOTIFICATIONS`). |
| `maxEventsInMemory` | `10000` | In-memory event cap. |
| `debugOnly` | `true` | Stay inert in non-debuggable builds even if wired in by mistake. |

The inspector reads the SDK's on-device DB (read-only) to show events, network, crashes, device
info, performance charts, a logcat viewer, a read-only SQL browser, a session timeline, and a
SharedPreferences browser.

---

## 15. Debug/QA add-on — Feedback

A standalone bug-report flow (screenshot → annotate → email). Wire
`debugImplementation(pantrix-feedback)` + `releaseImplementation(pantrix-feedback-noop)` (brings its
own Compose UI — no extra peers):

```kotlin
// Application.onCreate
PantrixFeedback.init(this, FeedbackConfig(recipientEmail = "qa@example.com"))

// From a screen — captures the current screen, then opens the form:
PantrixFeedback.show(activity)
```

`FeedbackConfig`: `debugOnly` (default `true`), `recipientEmail`, `subjectPrefix`
(default `"[App Feedback]"`), `enableShakeGesture` (default `false` — shake opens the feedback form
with a screenshot), `shakeThreshold` (default `2.5f`).

> If you enable shake on **both** the inspector and feedback, one shake triggers both — turn one off
> (e.g. `InspectorConfig(enableShakeGesture = false)`).

---

## 16. Debug add-on — Widget

A debug-only **home-screen App Widget** (Jetpack Glance) showing glanceable Pantrix counts —
total events, crash + ANR, network, pending-to-send, sessions, and the last event — that **opens
the Inspector when tapped**. It reads the SDK's on-device DB directly (read-only).

Wire `debugImplementation(pantrix-widget)` + `releaseImplementation(pantrix-widget-noop)` **plus
Jetpack Glance** (the module declares Glance `compileOnly`):

```kotlin
dependencies {
    debugImplementation("com.pantrix.analytics:pantrix-widget:1.0.0-beta.8")
    releaseImplementation("com.pantrix.analytics:pantrix-widget-noop:1.0.0-beta.8")

    // Peer dependency you must provide (debug only — release links the no-op twin):
    debugImplementation("androidx.glance:glance-appwidget:<your-version>")
}
```

There's **no init call** — the widget is declared in the merged manifest, so once the debug build is
installed it appears in the launcher's widget picker. Add it to the home screen: the tiles populate
from a running session, the **Yenile** button refreshes, and tapping the body opens the inspector.
Force a refresh from code with `PantrixWidget.refresh(context)`.

> Tapping opens the inspector, so keep `pantrix-inspector` wired ([§14](#14-debug-add-on--inspector)).
> If the inspector isn't initialized the widget still shows counts — the tap just won't open
> anything. An encrypted DB (`storageEncryption != NONE`) is shown as a degraded "—" state.

---

## 17. Storage encryption

Data is stored app-private (already OS-sandboxed); encryption is **defense-in-depth**. The mode is
fixed at first init.

| Mode | Encrypts | You must also provide |
| --- | --- | --- |
| `NONE` (default) | nothing | — |
| `DATABASE` | the SQLite DB (events, stack traces, captured bodies) via SQLCipher | `net.zetetic:sqlcipher-android` |
| `FULL` | `DATABASE` + sensitive preference values via an Android Keystore AES-GCM key | `net.zetetic:sqlcipher-android` |

```kotlin
PantrixConfig(token = "px_…", url = "https://…") {
    storageEncryption(StorageEncryption.DATABASE)
}
// + implementation("net.zetetic:sqlcipher-android:<your-version>")
```

SQLCipher is `compileOnly` in the SDK; the SDK **fails fast at init** if the flag is set but the
library is missing.

---

## 18. R8 / ProGuard

- The SDK **ships its own consumer keep rules** inside each AAR (`proguard.txt`) — JNI bridge,
  serialization models, capability detection survive minification with **no setup on your side**.
- **Readable screen names under R8.** Automatic screen tracking reads `javaClass.simpleName`; R8
  renames Fragments (`HomeFragment` → `c`). Activities are safe (kept by the manifest). To keep
  Fragment screen names, add to your `proguard-rules.pro`:

  ```proguard
  -keepnames class * extends androidx.fragment.app.Fragment
  ```

  Or pass explicit, obfuscation-proof names: `Pantrix.trackScreenView("Checkout")`.
- **Inspector in a minified build** (e.g. a `qaTest` variant that wires the real inspector with
  `isMinifyEnabled = true`) works out of the box — its consumer rules keep the type-safe navigation
  routes + serializers, so it won't crash with *"Serializer for class … not found"*. Nothing to add.
- **Release crash/ANR deobfuscation** — a minified release obfuscates stack traces (`a.b.c(:1)`).
  Apply the **`pantrix-gradle-plugin`** to upload the R8 `mapping.txt` and stamp its content UUID
  into the app, so the dashboard shows real class/method/line. Add the same Pantrix repo to
  `pluginManagement` (see [§2](#2-step-1--add-the-repository)), then:

  ```kotlin
  // app/build.gradle.kts
  plugins { id("com.pantrix.gradle") }   // resolves from the Pantrix maven repo
  pantrix {
      apiKey.set(providers.environmentVariable("PANTRIX_CI_KEY"))   // CI key (key_type=CI); never commit it
      variantFilter { apiUrl = "https://<your-host>/api" }          // where this variant's crashes go
  }
  ```

  Full reference (config, the build-time guard, verification):
  [R8 Mapping Upload plugin](collectors/en/gradle-plugin/r8-mapping-upload.md).

---

## 19. Privacy & collected data

Pantrix is **self-hosted** — data goes only to your `PantrixConfig.url`. You must disclose the
collection in your own privacy policy. In particular the SDK reads **`Settings.Secure.ANDROID_ID`**
(sent as `device.deviceId`) unless you set `collectDeviceId = false`.

| Group | Fields (examples) |
| --- | --- |
| **Identifiers** | `deviceId` (ANDROID_ID), `installId` (UUID, resets on reinstall), `cdId` (custom, optional) |
| **User-supplied** | `userId`, `userProperties` (exactly what you pass to `setUser`) |
| **Device** | brand, model, manufacturer, OS version & SDK, locale, time zone, screen size/density, carrier, `isPhysical`, `isRooted`, `isFoldable`, NFC |
| **App / build** | applicationId, app name, versionName/Code, `isDebuggable`, buildId |
| **Network** | connection type, up/down kbps, signal strength, carrier (no IP address) |
| **Activity** | screen views & names, activity/fragment lifecycle, launch timings, foreground/background |
| **Performance** | CPU & memory usage, trim-memory level |
| **Stability** | crashes & ANRs — exception messages, stack traces, thread dumps |
| **HTTP** (opt-in) | method, url, status, timings, sizes; headers/bodies only when enabled (redacted) |
| **Custom** | event name + attributes you pass to `trackEvent` |

**Controls:** `screenBlocklist`, `httpUrlBlocklist`, `httpHeadersBlocklist` (+ defaults), body/URL
value masking (built-ins + your `redactionKeys`), `trackHttpBody` off by default,
`storageEncryption`, and device-id opt-out (`collectDeviceId = false`).

---

## 20. Recommended build-variant setup

```kotlin
Pantrix.init(
    context = this,
    config = PantrixConfig(
        token = "px_your_project_token",
        url = "https://ingest.your-backend.example.com",
    ) {
        enableLogging(BuildConfig.DEBUG)
        // Keep everything in debug (for the inspector); prune in release.
        retentionDays(if (BuildConfig.DEBUG) 0 else 30)
        maxStoredEvents(if (BuildConfig.DEBUG) 0 else 50_000)
        keepSentEvents(BuildConfig.DEBUG)
    }
)
```

| Variant | SDK | Inspector / Feedback / Widget |
| --- | --- | --- |
| `debug` | real, keep-everything | real (`debugImplementation`) |
| `release` | real, pruned | **noop** (`releaseImplementation`) |
| `qaTest` (optional) | real, pruned, minified | real (`qaTestImplementation`) |

---

## 21. Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| Nothing is collected | `Pantrix.init` not called, or `autoStart = false` without `start()`. Set `enableLogging = true` to see SDK logs. |
| A Compose/Navigation/Ktor/Widget feature won't compile | Missing peer dependency — add the libraries listed in that add-on's section (they're `compileOnly` in the SDK; e.g. the widget needs `androidx.glance:glance-appwidget`). |
| `Pantrix.getOkHttpEventCollector()` returns null | OkHttp isn't on the classpath — add `com.squareup.okhttp3:okhttp`. |
| Screen names are single letters in release | R8 renamed Fragments — add `-keepnames … Fragment`, or pass explicit names. |
| Init fails with an encryption error | `storageEncryption != NONE` but SQLCipher missing — add `net.zetetic:sqlcipher-android`. |
| HTTP bodies not captured | `trackHttpBody` defaults to `false`; bodies are only captured for `Content-Type: application/json`. |
| Release build pulls in the debug inspector UI | You used `implementation` instead of `debugImplementation`; pair it with `releaseImplementation(pantrix-inspector-noop)`. |
