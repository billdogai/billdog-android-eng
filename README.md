# BillDog Android SDK — Eng

![Version](https://img.shields.io/badge/version-1.0.0--beta.2-blue)
![Platform](https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white)
![minSdk](https://img.shields.io/badge/minSdk-21-orange)
![Maven Central](https://img.shields.io/badge/Maven%20Central-io.billdog-blue?logo=apachemaven)
![License](https://img.shields.io/badge/license-MIT-green)

The **Eng** edition of the BillDog Android SDK — a single umbrella artifact, **`io.billdog:billdog-eng`**, published as a **compiled AAR** (closed source) to **Maven Central** at version **`1.0.0-beta.2`** — no account or token required.

> `billdog-eng` bundles the Eng feature set transitively: **core, analytics, A/B testing, notifications, in-app messages, session replay, and surveys**. It is **mutually exclusive** with the full `io.billdog:billdog` suite — install one or the other, never both.
>
> For the full (non-Eng) SDK and per-module installs, see **[billdog-android-full](https://github.com/billdogai/billdog-android-full)**.

---

## 🏗️ What's inside

`billdog-eng` is a convenience umbrella that pulls these modules transitively (all published to Maven Central under `io.billdog`):

```text
billdog-eng  →  billdog-core            (identity + billing foundation)
             +  billdog-analytics       (events + user properties)
             +  billdog-ab-test         (A/B testing)
             +  billdog-notifications   (push + deep links)
             +  billdog-inappmessages   (in-app messaging)
             +  billdog-replay          (session replay)
             +  billdog-survey          (survey engine)
             +  billdog-survey-compose  (survey UI)
```

---

## 🚀 Installation

### 1. Add the registry

`billdog-eng` and all its transitive dependencies are on **Maven Central** — no account, credentials, or token required. Ensure `mavenCentral()` is in your `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}
```

### 2. Add the dependency

```kotlin
dependencies {
    implementation("io.billdog:billdog-eng:1.0.0-beta.2")
}
```

That one line brings in the full Eng suite transitively.

<details>
<summary>Alternative: GitHub Packages</summary>
<br>

The artifacts are also mirrored to GitHub Packages, but the Eng umbrella and its shared modules live in **two** repos (`billdog-android-eng` + `billdog-android-full`), and GitHub Packages requires a PAT with `read:packages` even to download — so **Maven Central above is the recommended path**. If you specifically need the mirror, register both Maven URLs (`.../billdogai/billdog-android-eng` and `.../billdogai/billdog-android-full`) with `gpr.user`/`gpr.key` credentials.

</details>

---

## ⚡ Quick start

Initialize once, in your `Application`:

```kotlin
import android.app.Application
import io.billdog.core.BillDog

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        BillDog.configure(this, "YOUR_API_KEY")
    }
}
```

Then, anywhere in your app:

```kotlin
// Track a product / analytics event
BillDog.trackEvent("checkout_started", mapOf("plan" to "pro"))

// Read entitlements for a product
val entitlements = BillDog.entitlementsForProduct("premium_monthly")

// End the session / clear identity
BillDog.logOut()
BillDog.reset()
```

---

## ✅ Requirements

- **minSdk 21** (Android 5.0+)
- **Java 17** toolchain

---

## 📄 License

MIT — see the published POM metadata. Artifact is a compiled AAR; **source is not distributed**.

<sub>© BillDog.io</sub>
