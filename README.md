# WalkMe Android SDK (core) — integration guide

Standard WalkMe mobile integration **without** Power Mode editor flows. Artifact: **`walkme-android-sdk`**.

## Requirements

- **Minimum SDK:** API **21+**
- Use Android Gradle Plugin and Kotlin versions compatible with your chosen SDK release (follow release notes if provided).

## 1. Add the JitPack repository

In your **root** `settings.gradle` / `settings.gradle.kts` (Gradle 7+):

**Groovy (`settings.gradle`)**

```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url "https://jitpack.io" }
    }
}
```

**Kotlin (`settings.gradle.kts`)**

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven(url = "https://jitpack.io")
    }
}
```

If repositories are declared only in the project `build.gradle`, add the same `maven { url "https://jitpack.io" }` there.

## 2. Add the dependency

Replace the version with any tag or commit published on JitPack.

**Groovy**

```gradle
dependencies {
    implementation "com.github.WalkMe-int:walkme-android-sdk:0.0.5alpha"
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("com.github.WalkMe-int:walkme-android-sdk:0.0.5alpha")
}
```

## 3. Public API — `WalkMeSDK`

**Package:** `com.walkme.sdk`

| API | Purpose |
|-----|--------|
| `start(activity, systemGuid, options, env, dataCenter)` | Initialize and show WalkMe for the current session. Call from a host `Activity`, usually in `onCreate` (or when the screen that should host WalkMe becomes active). |
| `stop()` | Tear down the SDK and release resources when your app no longer needs WalkMe. |
| `setUserId(userId)` | Set or clear (`null`) the end-user id for segmentation, analytics, and support. |
| `setLanguage(language)` | Set UI language where your WalkMe configuration supports it (requires the relevant admin option when applicable). |
| `setUserAttribute(key, value)` | Set a custom user attribute; pass `null` for `value` to clear. |

**Startup options**

- `WalkMeStartOptions(analyticsEnabled = true)` — toggle analytics (default `true`).

**Example (Kotlin)**

```kotlin
import com.walkme.sdk.WalkMeSDK
import com.walkme.sdk.WalkMeStartOptions

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        WalkMeSDK.start(
            activity = this,
            systemGuid = "<YOUR_SYSTEM_GUID>",
            options = WalkMeStartOptions(analyticsEnabled = true),
            env = "Production",
            dataCenter = "prod"
        )
    }

    override fun onDestroy() {
        WalkMeSDK.stop()
        super.onDestroy()
    }
}
```

Adjust `env` and `dataCenter` to match your WalkMe environment (for example `"prod-eu"` if your project uses EU data residency).

## 4. Integration checklist

1. Add **JitPack** to repositories.
2. Add **`walkme-android-sdk`** with latest version.
3. Obtain **`systemGuid`**, **`env`**, and **`dataCenter`** from your WalkMe project / onboarding.
4. Call **`start`** from an `Activity` when the host screen is ready; call **`stop`** when tearing down (for example in `onDestroy` or your logout flow—confirm with your WalkMe contact).
5. Wire **`setUserId`** / **`setUserAttribute`** after login and clear on logout if your policy requires it.

---

**Related:** For Power Mode (editor) features, see [Walkme-Android-Sdk-Editor](https://github.com/WalkMe-int/walkme-android-sdk-editor). Do not add both artifacts at the same time.
