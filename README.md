# ParkTrack — PROG2007 Lecture 6: Sensors & Location

ParkTrack is a demo Android app built for the PROG2007 Sensors & Location lecture. It automatically detects when you've parked (accelerometer), lets you track how far you've walked away from the car (step counter), and shows the parked location on a map (GPS + Google Maps). The app follows MVVM: all sensor and location logic lives in `ParkingViewModel`, and the Composable screens only observe `StateFlow`s and call ViewModel functions.

This repo contains three snapshots of the same file, one per stage of the build:

| File | What it adds |
|---|---|
| `Feature-1-Code` | Parking detection via the accelerometer, with a confirmation dialog and adjustable sensitivity |
| `Feature-1-2-Code` | + Step counting and estimated walking distance from the car |
| `Feature-1-2-3-Code` | + Parking location captured via GPS and shown on an embedded Google Map |

Each file is a complete, drop-in replacement for `MainActivity.kt` — they are not meant to be combined, just progressive stages of the same app.

## Prerequisites

- Android Studio (a recent version compatible with AGP 9.3.2 / Kotlin 2.2.10 — Ladybug or newer)
- A physical Android device for testing Feature 2 (step counter has no emulator support) and for the most realistic test of Feature 3
- A Google account with access to Google Cloud Console for Feature 3 (Maps API key)

## 1. Create the base project

1. In Android Studio: **New Project → Empty Activity (Compose)**.
2. Name: `Lect6SensorsLocation`
   Package: `com.example.lect6sensorslocation`
3. Once created, open `app/src/main/java/com/example/lect6sensorslocation/MainActivity.kt` and replace its entire contents with whichever feature file from this repo you want to run.

Which sections below you need depends on which file you copied in — Feature 1 needs the least setup, Feature 3 needs the most.

## 2. Feature 1 — Parking Detection

**Dependencies** (add to `gradle/libs.versions.toml` and the app's `build.gradle.kts` if not already present in a fresh Compose project):

```toml
# libs.versions.toml
[versions]
navigationCompose = "2.9.5"
kotlinxSerializationJson = "1.7.3"

[libraries]
navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinxSerializationJson" }
```

```kotlin
// app/build.gradle.kts
plugins {
    kotlin("plugin.serialization") version "2.2.10"
}

dependencies {
    implementation(libs.navigation.compose)
    implementation(libs.kotlinx.serialization.json)
}
```

**Permissions:** none needed — `SensorManager` and the accelerometer are SDK-native.

**Testing:** works fully in the emulator. Open **Extended Controls → Virtual sensors → Accelerometer** and drag the movement sliders to simulate driving, then let them settle to simulate parking.

## 3. Feature 2 — Step Counter & Distance

Builds on everything in Feature 1.

**Dependencies:** none new — the permission-request UI uses `rememberLauncherForActivityResult`, which comes from `androidx.activity:activity-compose`, already included in a default Compose project template.

**Manifest:** add above `<application>`:

```xml
<uses-permission android:name="android.permission.ACTIVITY_RECOGNITION" />
```

Without this line, Android silently auto-denies the runtime permission request with no dialog at all — if the "Start Step & Distance Tracking" button never prompts for permission, check this first.

**Testing:** requires a **physical device**. The emulator's Extended Controls has no virtual sensor for step count, so `TYPE_STEP_COUNTER` cannot be simulated at all. Enable Developer Options and USB or wireless debugging, then run directly on the phone — wireless (Wi-Fi) debugging works fine if a USB cable isn't convenient.

**Calibration note:** the app estimates distance as `steps × STEP_LENGTH_METERS` (default `0.75f`). This is a rough population-average stride length and will overestimate for shorter strides. To calibrate: walk a known distance, divide by the steps recorded, and update the constant.

## Connecting a Physical Device via USB Debugging

Needed for Feature 2 (step counter has no emulator support) and optional for Feature 3.

1. **Enable Developer Options** on the phone: Settings → About phone → tap "Build number" 7 times until it says "You are now a developer."
2. **Enable USB debugging**: Settings → System → Developer options → turn on "USB debugging."
3. **Connect the phone to your computer with a USB cable.**
4. A prompt appears on the phone: **"Allow USB debugging?"** with an RSA key fingerprint — tap **Allow** (optionally check "Always allow from this computer" to skip this next time).
5. In Android Studio, the device should now appear in the device dropdown at the top of the toolbar (next to the Run button). If it doesn't show up:
   - Make sure the USB cable supports data transfer, not just charging.
   - On the phone's USB notification, set the connection mode to "File Transfer" (MTP) rather than "Charging only."
   - Run `adb devices` in a terminal — the phone should show up as `device`, not `unauthorized` (if `unauthorized`, check the phone screen for the allow prompt) or `offline` (try reconnecting the cable).
6. Select the device from the dropdown and click **Run** — the app installs and launches directly on the phone.

If USB isn't convenient, wireless debugging over Wi-Fi works the same way once paired once via USB or a QR code (Settings → Developer options → Wireless debugging) — see the Feature 2 testing note above.

The exact menu names and steps above can vary by phone manufacturer and Android version (Samsung, Pixel, Xiaomi, etc. each organize Settings a bit differently), so search online for your specific device model if a step doesn't match what you see.

## 4. Feature 3 — Parking Location on a Map

Builds on everything in Features 1 and 2.

**Dependencies:**

```toml
# libs.versions.toml
[versions]
playServicesLocation = "21.3.0"
mapsCompose = "6.4.4"

[libraries]
play-services-location = { group = "com.google.android.gms", name = "play-services-location", version.ref = "playServicesLocation" }
maps-compose = { group = "com.google.maps.android", name = "maps-compose", version.ref = "mapsCompose" }
```

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(libs.play.services.location)
    implementation(libs.maps.compose)
}
```

**Manifest:** add the location permissions and your Maps API key:

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<application ...>
    <meta-data
        android:name="com.google.android.geo.API_KEY"
        android:value="YOUR_API_KEY_HERE" />
    ...
</application>
```

**Getting a Maps API key:**

1. Go to [Google Cloud Console](https://console.cloud.google.com), create or select a project.
2. Enable billing on the project (Maps SDK for Android requires it, though usage stays within the free tier for a class project).
3. **APIs & Services → Library** → search "Maps SDK for Android" → **Enable**.
4. **APIs & Services → Credentials → Create Credentials → API key** — copy the generated key.
5. Restrict the key: **Application restrictions → Android apps** (add package name `com.example.lect6sensorslocation` and its SHA-1, from `./gradlew signingReport`), and **API restrictions → Maps SDK for Android** only.
6. Paste the key into the manifest `meta-data` value above.

**Testing:** unlike the step counter, this **works in the emulator**. Open **Extended Controls → Location**, set a latitude/longitude (or play back a route), and it's reported exactly as if from real GPS. Set a location *before* tapping "Yes, parked," since `lastLocation` returns null if no fix has ever been set. A physical device is only needed if you want to test with real GPS movement.

## Known gotchas (all features)

- **`Cannot access 'val RowColumnParentData?.weight...'`** — if you see this compile error, remove any explicit `import androidx.compose.foundation.layout.weight`. It's a scoped `RowScope`/`ColumnScope` member, not a top-level import, and Kotlin's K2 compiler rejects the explicit import even though K1 tolerated it.
- **`MissingPermission` lint warning** on location/step calls — the code already checks permissions before calling, but lint can't trace custom helper functions, so the calls are wrapped in `try { } catch (e: SecurityException) { }` to satisfy it.
- **Permission dialog never appears** — almost always a missing manifest `<uses-permission>` line for the permission being requested (see the Feature 2 and Feature 3 sections above).

## Architecture

All three files follow MVVM:

- `ParkingViewModel` owns all sensor/location state and logic (`SensorManager`, `FusedLocationProviderClient`, debounce timers, permission-gated flows), exposed as `StateFlow`s.
- Composable screens (`StartScreen`, `MainScreen`, `HistoryScreen`) only observe those flows via `collectAsState()` and call ViewModel functions from `onClick` / `DisposableEffect` — no sensor or location code lives in the UI layer.
- Navigation uses type-safe Navigation Compose routes (`@Serializable` route objects + `composable<T>`).
