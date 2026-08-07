# BillDog Android SDK — Eng

![Version](https://img.shields.io/badge/version-1.0.0--beta.2-blue)
![Platform](https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white)
![minSdk](https://img.shields.io/badge/minSdk-21-orange)
![Distribution](https://img.shields.io/badge/Maven-GitHub%20Packages-24292e?logo=github)
![License](https://img.shields.io/badge/license-MIT-green)

The **Eng** edition of the BillDog Android SDK — a single umbrella artifact, **`io.billdog:billdog-eng`**, distributed as a **compiled AAR** (closed source) via **GitHub Packages (Maven)** at version **`1.0.0-beta.2`**.

> `billdog-eng` bundles the Eng feature set transitively: **core, analytics, A/B testing, notifications, in-app messages, session replay, and surveys**. It is **mutually exclusive** with the full `io.billdog:billdog` suite — install one or the other, never both.
>
> For the full (non-Eng) SDK and per-module installs, see **[billdog-android-full](https://github.com/billdogai/billdog-android-full)**.

---

## 🏗️ What's inside

`billdog-eng` is a convenience umbrella that pulls these modules from the **full** registry:

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

### 1. Add **both** registries

`billdog-eng` lives here; its shared dependencies live in `billdog-android-full`, so you must register **both** Maven repositories. GitHub Packages requires a **Personal Access Token** with the **`read:packages`** scope even to download.

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()

        // The Eng umbrella artifact
        maven {
            url = uri("https://maven.pkg.github.com/billdogai/billdog-android-eng")
            credentials {
                username = providers.gradleProperty("gpr.user").orNull ?: System.getenv("GITHUB_ACTOR")
                password = providers.gradleProperty("gpr.key").orNull ?: System.getenv("GITHUB_TOKEN")
            }
        }

        // Shared transitive modules (billdog-core, -analytics, -survey, …)
        maven {
            url = uri("https://maven.pkg.github.com/billdogai/billdog-android-full")
            credentials {
                username = providers.gradleProperty("gpr.user").orNull ?: System.getenv("GITHUB_ACTOR")
                password = providers.gradleProperty("gpr.key").orNull ?: System.getenv("GITHUB_TOKEN")
            }
        }
    }
}
```

Store credentials in `~/.gradle/gradle.properties` (never commit them):

```properties
gpr.user=YOUR_GITHUB_USERNAME
gpr.key=YOUR_GITHUB_PAT_WITH_read_packages
```

### 2. Add the dependency

```kotlin
dependencies {
    implementation("io.billdog:billdog-eng:1.0.0-beta.2")
}
```

That one line brings in the full Eng suite transitively.

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
