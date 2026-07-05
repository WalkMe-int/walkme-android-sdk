# WalkMe Android SDK (core) — integration guide

Standard WalkMe mobile integration **without** Power Mode editor flows. Artifact: **`walkme-android-sdk`**.

## Requirements

- **Minimum SDK:** API **24+**
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
    implementation "com.github.WalkMe-int:walkme-android-sdk:1.1.1"
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("com.github.WalkMe-int:walkme-android-sdk:1.1.1")
}
```

## 3. Public API — `WalkMeSDK`

**Package:** `com.walkme.sdk`

| API                            | Purpose                                                                                                                                                                                                                                                                             |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `start(activity, options)`     | Initialize and show WalkMe for the current session. Call from a host `Activity`, usually in `onCreate` (or when the screen that should host WalkMe becomes active).                                                                                                                 |
| `start(application, options)`  | Initialize and show WalkMe for the current session. Call from a host `Application`, usually in `onCreate` .                                                                                                                                                                         |
| `stop()`                       | Tear down the SDK and release resources when your app no longer needs WalkMe.                                                                                                                                                                                                       |
| `restart()`                    | Re-initialize the SDK with the same options and host as the last successful `start()`. No-op if the SDK is not running (`start()` was not called, or `stop()` already ran).                                                                                                           |
| `setUserId(userId)`            | Set or clear (`null`) the end-user id for segmentation, analytics, and support.                                                                                                                                                                                                     |
| `setLanguage(language)`        | Set UI language where your WalkMe configuration supports it (requires the relevant admin option when applicable).                                                                                                                                                                   |
| `setVariable(key, value)`      | Set a custom variable used by WalkMe rules and segments; pass `null` for `value` to clear.                                                                                                                                                                                          |
| `setEventUserVars(values)`     | Set keys for WalkMe **event** payloads (`userVars`). Pass a `Map<WalkMeEventUserVarsKey, String>`. Each call **merges** into the stored map (same key overwrites). Use `com.walkme.api.WalkMeEventUserVarsKey` (`NAME`, `ROLE`, `TYPE`, `STATUS`, `INFO`).                          |
| `setTenantId(tenantId)`        | Set or clear (`null`) the tenant ID for the current user (max **50 characters**; longer values are truncated). Attached to analytics events for tenant-based reporting. Call after sign-in when the tenant is known; pass `null` on sign-out. Persisted across app sessions.                                                             |
| `startItemByID(itemId, deepLink?)` | Start a specific **promotion** by WalkMe `itemId`. If another promotion is already playing, it is stopped first. Optional `deepLink` is a URI string; when non-null and your app can resolve `ACTION_VIEW` for that URI (same package), the SDK opens it before playing the promotion. |
| `dismissItem()` | Dismiss the **currently presented** WalkMe promotion (not launchers). Does not stop the SDK. No-op if no promotion is active or the SDK is not started. |
| `sendEvent(name, attributes)`  | Sends a custom tracked event: name identifies the event, attributes is an optional map of key/value data.                                                                                                                                                                           |
| `setItemInfoListener(listener)` | Register a listener for item lifecycle callbacks (`onItemPresented`, `onItemDismissed`, `onItemAction`). Pass `null` to clear. See **Item info callbacks** below.                                                                                                        |
| `setAnalyticsListener(listener)` | Register a listener for successfully posted analytics events (`onSendAnalyticsEvent`). Pass `null` to clear. See **Analytics callbacks** below. No callbacks when `analyticsEnabled` is `false`. |


**Startup options**

`com.walkme.api.WalkMeStartOptions` — same type is used by the Power Mode SDK.

| Option | Type | Default | Purpose |
|--------|------|---------|---------|
| `systemGuid` | `String` | — | WalkMe system GUID (required). |
| `environment` | `String` | `"Production"` | Environment name (e.g. `"Production"`). |
| `dataCenter` | `WalkmeDataCenter` | `prod` | Region — `prod`, `eu`, `us01`, `eu01`, or `Custom("…")`. |
| `analyticsEnabled` | `Boolean` | `true` | When `false`, the SDK does not send analytics/events to WalkMe (including heartbeat). |
| `localLogsEnabled` | `Boolean` | `false` | When `true`, SDK debug logs are written to Logcat (`WMLogger`). Use for troubleshooting only. |

`analyticsEnabled` and `localLogsEnabled` are mutable properties (not constructor parameters). Set them on the options instance before calling `start`:

```kotlin
val options = WalkMeStartOptions(
    systemGuid = "<YOUR_SYSTEM_GUID>",
    environment = "Production",
    dataCenter = WalkmeDataCenter.prod,
)
options.analyticsEnabled = true   // default
options.localLogsEnabled = false  // default; set true for debug builds if needed
```

**Example (Kotlin)**

```kotlin
import com.walkme.api.WalkMeStartOptions
import com.walkme.api.WalkmeDataCenter
import com.walkme.sdk.WalkMeSDK

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val options = WalkMeStartOptions(
            systemGuid = "<YOUR_SYSTEM_GUID>",
            environment = "Production",
            dataCenter = WalkmeDataCenter.prod,
        )
        WalkMeSDK.start(this, options)
    }

    override fun onDestroy() {
        WalkMeSDK.stop()
        super.onDestroy()
    }
}
```

Adjust `environment` and `dataCenter` to match your WalkMe environment (for example `WalkmeDataCenter.eu` for EU, or `WalkmeDataCenter.Custom("…")` for other backend values).

## 4. Item info callbacks

Register **`WMItemInfoListener`** (`com.walkme.api`) to receive item lifecycle events and forward them to your analytics, CRM, or app logic.

| Callback | When it fires |
|----------|----------------|
| `onItemPresented(itemInfo)` | Right before a deployable item is shown. |
| `onItemDismissed(itemInfo)` | After a deployable item is dismissed (close, submit, remind-me-later, etc.). |
| `onItemAction(itemInfo, args)` | When the user performs an action on a deployable (e.g. button click). `args` is an optional map of action parameters. |

**`WMItemInfo`** (`itemId`, `itemActionType`, `userData`) — context for the item and the action that triggered the callback (if any).

**`WMUserData`** — user/device snapshot at interaction time: `userAttributesMap`, `sessionDuration`, `deviceVersion`, `deviceId`, `deviceModel`, `deviceOrientation`, `appVersion`, `appName`, `locale`, `sdkVer`, `sessionId`, `isNewUser` (`"true"` / `"false"`), `timezone`, `network`, `systemName`, `timestamp`.

Callbacks are delivered on the **main thread** in all environments (including when `environment` is `"preview"`). The listener is cleared when you call `stop()`.

**Example (Kotlin)**

```kotlin
import com.walkme.api.WMItemInfo
import com.walkme.api.WMItemInfoListener
import com.walkme.sdk.WalkMeSDK

WalkMeSDK.setItemInfoListener(object : WMItemInfoListener {
    override fun onItemPresented(itemInfo: WMItemInfo) {
        // Item about to show — itemInfo.itemId, itemInfo.userData, …
    }

    override fun onItemDismissed(itemInfo: WMItemInfo) {
        // Item dismissed — itemInfo.itemActionType describes the dismiss action
    }

    override fun onItemAction(itemInfo: WMItemInfo, args: Map<String, String>?) {
        // User interacted with a deployable action
    }
})

// On teardown:
WalkMeSDK.setItemInfoListener(null)
```

## 5. Analytics callbacks

Register **`WMAnalyticsListener`** (`com.walkme.api.analytics`) to receive analytics payloads after the SDK successfully posts them to WalkMe (`event/postEvent`).

| Callback | When it fires |
|----------|----------------|
| `onSendAnalyticsEvent(eventName, params)` | After an analytics event POST succeeds. Not called on network failure or when events are not sent. |

- **`eventName`** — WalkMe event type string (same as the `"type"` field in the POST body, e.g. `"play"`, `"click"`, `"activity"`).
- **`params`** — Full JSON body posted to WalkMe (`time`, `type`, `data`, `env`, `version`, `wm`, `ctx`, `sId`). Treat as read-only.

Callbacks are delivered on the **main thread**. The listener is cleared when you call `stop()`. No callbacks are delivered when **`analyticsEnabled`** is `false` on your [WalkMeStartOptions](#startup-options) (the SDK does not send events in that case).

**Example (Kotlin)**

```kotlin
import com.walkme.api.analytics.WMAnalyticsListener
import com.walkme.sdk.WalkMeSDK
import org.json.JSONObject

WalkMeSDK.setAnalyticsListener(object : WMAnalyticsListener {
    override fun onSendAnalyticsEvent(eventName: String, params: JSONObject) {
        // Forward to your analytics — eventName, params.optJSONObject("data"), …
    }
})

// On teardown:
WalkMeSDK.setAnalyticsListener(null)
```

## 6. Integration checklist

1. Add **JitPack** to repositories.
2. Add **`walkme-android-sdk`** with latest version.
3. Obtain **`systemGuid`**, **`environment`**, and **`dataCenter`** from your WalkMe project / onboarding.
4. Call **`start`** from an `Activity` when the host screen is ready; call **`stop`** when tearing down (for example in `onDestroy` or your logout flow—confirm with your WalkMe contact).
5. Wire **`setUserId`** / **`setVariable`** / **`setTenantId`** after login and clear on logout if your policy requires it.
6. Optionally register **`setItemInfoListener`** after `start()` if you need item lifecycle hooks.
7. Optionally register **`setAnalyticsListener`** after `start()` if you need successfully posted analytics payloads.

---

**Related:** For Power Mode (editor) features, see [Walkme-Android-Sdk-Editor](https://github.com/WalkMe-int/walkme-android-sdk-editor). Do not add both artifacts at the same time.
