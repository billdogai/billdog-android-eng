# BillDog Android SDK — Eng

The **Eng** edition of the BillDog Android SDK — `io.billdog:billdog-eng`, distributed as a **compiled AAR** (no source) via **GitHub Packages (Maven)**.

> **Version:** `1.0.0-beta.2`  ·  **Group:** `io.billdog`

`billdog-eng` is a single umbrella artifact that bundles the Eng feature set (core, analytics, A/B testing, notifications, in-app messages, session replay, and surveys). It is **mutually exclusive** with the full `io.billdog:billdog` suite — install one or the other, not both.

---

## 1. Repository setup

`billdog-eng` pulls its shared dependencies from the **full** registry, so add **both** Maven repositories. GitHub Packages requires a **Personal Access Token** with `read:packages` even to download.

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        // Eng artifact
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

Credentials go in `~/.gradle/gradle.properties` (never commit them):

```properties
gpr.user=YOUR_GITHUB_USERNAME
gpr.key=YOUR_GITHUB_PAT_WITH_read:packages
```

## 2. Add the dependency

```kotlin
dependencies {
    implementation("io.billdog:billdog-eng:1.0.0-beta.2")
}
```

That single line brings in the full Eng suite transitively.

---

_Artifact is a compiled AAR; source is not distributed. For the full (non-Eng) SDK see [`billdog-android-full`](https://github.com/billdogai/billdog-android-full). © BillDog._
