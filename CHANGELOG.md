# Changelog

All notable changes to the BillDog Android SDK will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.1.1-beta.5] - 2026-11-03

### Fixed — Superwall 2.9: paywall purchases reach BillDog

- `billdog-integration-superwall` now implements the `purchase(…, ProductReplacement)` overload Superwall 2.9 added to `PurchaseController` and calls for every paywall purchase. Compiled against Superwall 2.0, the controller lacked it: on Superwall 2.9 every tap on a paywall's buy button was an `AbstractMethodError` that Superwall swallowed, and nothing was bought. A Superwall plan change (`ProductReplacement`) becomes BillDog's (`oldProductId` + `replacementMode`). The module now compiles against Superwall 2.9.0; Superwall 2.0–2.8 keep working.

### Fixed — OAuth sign-in opens in a browser, never in another app

- The sign-in tab is pinned to a browser (the default one if it supports Custom Tabs, else any that does). Unpinned, Android handed `https://www.facebook.com/...` to the installed Facebook app, which drops a web OAuth request it did not start, so Facebook sign-in came back "cancelled by the user" on every phone with the Facebook app. AppAuth pins its tab the same way. The manifest now also declares the Custom Tabs service and `http` browsers under `<queries>` so Android 11+ can see them.

### Fixed — analytics batches name the SDK that sent them

- Each ingest batch carries `lib` / `lib_version` (PostHog's `$lib` / `$lib_version`): `billdog-android` at the SDK version, or the wrapper that runs it (`billdog-flutter`, `billdog-react-native`, `billdog-expo`, …, from `BillDogPlatformInfo.current`) at the wrapper's version. Native rows were stored with no lib, so their `event_source` was 'other' instead of 'mobile'.

### Fixed — analytics flushes when the app goes to the background

- The queue is flushed when the process goes to the background (server `flush_on_background`, default on) and the WorkManager sync is scheduled, whether or not session auto-tracking runs. Both lived in `AutoTracker`, which runs only with session tracking on (off by default), so an app on defaults left its last events for a timer tick in a process the OS may freeze, or for the next launch. posthog-android flushes on every process `onStop` the same way.

### Added — session replay for the wrappers

- `SessionReplayConfig.externalFrames` and `BillDogReplay.appendExternalFrame(bitmap)`: a recording whose frames the host draws and masks itself (the Flutter SDK), encoded and uploaded like screenshots.
- `SessionReplayConfig.maskAllImages` (default false). React Native's `testID` is read as a `bd-mask` marker, and Fabric's `PreparedLayoutTextView` is masked as text.

### Changed (round 3 platform parity)

- **`billdog-notifications` declares no exact-alarm permission.** `USE_EXACT_ALARM` (Google Play restricts it to alarm and calendar apps) and `SCHEDULE_EXACT_ALARM` (denied by default from Android 14) are gone from the library manifest, so no host inherits a Play-policy problem. Reminders go through WorkManager, now as unique work per reminder (re-scheduling replaces instead of queueing a duplicate), and with the same fire-time refund recheck as the alarm path. An app that declares `SCHEDULE_EXACT_ALARM` itself and is granted it still gets exact alarms; a revoked grant falls back to WorkManager instead of dropping the reminder.
- **The web-checkout return scheme is app-specific.** `BillDogConfig.customUrlScheme` defaults to empty, meaning `<applicationId>.billdog` (`CheckoutUrlScheme`), instead of the `billdog` every app shared — with two BillDog apps installed a return or redeem link (a bearer token) could open the other one. An app that still registers only `billdog` keeps using it (logged) until it adds its own intent filter; incoming `billdog://` links stay accepted in default mode. A scheme set in the config is the only one claimed.
- **One consent path.** `ConsentManager` tells its listeners after every write (`addChangeListener`), and `BillDog` drops its kept placement answers there, so a per-purpose grant, a module's own consent setter or a wrapper calling `ConsentManager` directly has the same effect as `BillDog.setConsentGiven`. New facade calls `BillDog.setConsent(purpose, granted)` / `hasConsent(purpose)` (iOS parity).

### Added

- **FCM token path** (`io.billdog.notifications.fcm`): with `firebase-messaging` and `google-services.json` in the app (the SDK links Firebase compile-only), the install's FCM token is registered at every configure, and `BillDogFirebaseMessagingService` — declare it in the app manifest — registers new tokens and draws BillDog's pushes. Without Firebase nothing changes.

### Deprecated (could never work; no longer offered by any wrapper)

- `VirtualCurrencyManager.addCurrency`, `deductCurrency`, `grantCurrency` and `exchangeCurrency` throw `BillDogException(ConfigurationError)` at once, without a request. Every wallet write needs the project's secret key and the server answers an app's publishable key 403, so they failed on every device (`/exchange` is retired server-side). Virtual currencies are read-only in the client, as in RevenueCat: credit and spend from your server (`POST /v1/virtual-currency/transactions` or `/adjustments`). `getCurrencyBalance` and `getTransactionHistory(currencyId)` take a currency code as well as its id. iOS: the same.
- `BillDogNotifications.addTag` / `addTags` / `removeTag` / `removeTags` / `getTags`, the alias calls, `addEmail` / `removeEmail` / `getEmails` and `addSms` / `removeSms` / `getSmsNumbers` are no-ops that log one warning; the getters return nothing. They kept their values in memory and never sent them. Tags, email and phone are contact fields only your server may write (`sync-subscriber`, secret key); an alias is `BillDog.logIn(appUserId)`. iOS: the same.

### Fixed

- Session replay lost the chunk, video included, whenever the user scrolled for a few seconds: every touch move the panel reported was recorded (a 400 ms swipe was 381 events), the chunk went over record-session-replay's 500 events and was refused (400). Moves are thinned as iOS and rrweb thin them (one per 50 ms, 10 dp apart) and a chunk's touches stay under 400.
- Session replay recordings were never stored: every chunk named `project_id` = `BillDogConfig.projectId`, the backend's ref, and record-session-replay refused it (403 "API key does not match the project ID"); the uploader drops a 403. Chunks name no project now — the API key decides it.
- Session replay started while a screen was already showing (a button, the dashboard's `session_replay` once sdk-config lands after launch) recorded nothing until the user left the app and came back. It now starts on the Activity in front.
- An active user's session ended at the analytics session timeout unless the app tracked an event, and session replay, which follows the session, split the recording there (each part billed as a recording). A touch is now session activity: new `BillDogAnalytics.touchSession()`, called by the replay touch tracker, as PostHog's `touchSession` does.
- Notification permission observers (`PushSubscriptionManager.addPermissionObserver`) hear a change made in Settings when the app returns to the foreground: `BillDogNotifications` re-reads the permission on each foreground, as iOS does on `didBecomeActive`. They used to hear it only when the app read the status itself. Subscription observers hear it too (`optedIn` follows the permission), as on iOS.

### Added

- `PaywallPresentationHandler.onOAuthSuccess(PaywallSignIn)` (public, also a constructor parameter): the paywall closed after a sign-in made on it that the server verified, with no purchase or restore. Called just before `onDismiss`, from `presentPaywall` and from `register(…, presentationHandler)`, which reported that close as a plain dismissal. Same shape as iOS.
- `PaywallScreen(onOpenUrl: ((String) -> Boolean)?)`: a host that embeds the screen hears an "Open URL" tap and the Terms / Privacy links, and returns true to take the URL (the screen then does not open it) or false to let the screen open it. The screen opened every URL itself with no callback, so the Flutter and React Native inline paywall views could not report it. Deep links, checkout and sign-in tabs are unaffected. iOS hands the same URLs to its `PaywallViewControllerDelegate`.

- `BillDogConfig.showsPurchaseFailureAlert` (default true), as on iOS: turns off the paywall's restore alerts and its purchase-failed Toast for an app that shows its own message.
- `BillDog.canMakePayments(features, callback)` and `awaitCanMakePayments(features)`: connect a BillingClient and answer whether Play can take payments, and whether each requested `BillingClient.FeatureType` is supported (RevenueCat `Purchases.canMakePayments`). The synchronous `canMakePayments()` is deprecated.

### Fixed

- Restore on the native paywall tells the user when nothing was restored ("No Subscription Found / We couldn't find an active subscription for your account.") or the restore failed (the error), in an alert over the paywall; a cancellation shows nothing. It only logged, and the button looked dead. iOS (contract C4) and Superwall Android (`PaywallOptions.RestoreFailed`) show the same alert.
- `BillDog.compose.presentPaywall` runs through core's presentation: the delegate hears `willPresentPaywall`, `onPaywallPresented`, `didPresentPaywall` and `onAnalyticsEvent(paywall_presented)` once the screen has started, and `willDismissPaywall`, `onPaywallDismissed` and `didDismissPaywall` when it closes, as from `BillDog.presentPaywall` and on iOS. It told the delegate nothing, and counted its `paywall_presented` before the screen started, for a launch that might never render.
- An embedded `PaywallScreen` counts one `paywall_presented` across Activity recreations (rotation, dark mode, a locale change in an app that does not declare `configChanges`). Each recreation counted another, deflating conversion rates. iOS guards the same case.
- `setUserAttributes` is refused until consent is given when `setConsentRequired(true)`, like every other identity setter and as on iOS. It sent the attributes regardless.
- `redeemAndNotify` without a `delegate` argument tells `BillDog.delegate` (`onWillRedeemLink` / `onDidRedeemLink`), as on iOS; it told nobody. After a successful `redeemPurchase` or `redeemAndNotify`, customer info and `subscriptionStatus` are read again and published, so a web purchase unlocks at once (RevenueCat `WebPurchaseRedemptionHelper`).
- `getCustomerInfo` sends store purchases still waiting to be recorded (offline, a timed-out recording) before it reads, as `refreshSubscriptionStatus` and iOS do (RevenueCat `CustomerInfoHelper`). A paid purchase was missing from an explicit `getCustomerInfo(forceRefresh = true)`.
- `subscriptionStatus` judges the subscription's end date: an answer, restored from disk or stale, whose subscription has since lapsed no longer reads `ACTIVE` (RevenueCat's date-judged `activeSubscriptions`, iOS `hasActiveSubscription`).
- `canMakePayments()` asked `PackageManager.hasSystemFeature("com.android.vending.billing.IN_APP_BILLING")`, which is not a system feature, so it answered false on every device. React Native's `canMakePayments()` now asks Play through the callback form.
- `logOut()` clears each module's user-scoped state (feature flags, experiment variants and payloads, in-app message triggers and displayed ids), and the A/B test module, when it was evaluating flags for a user, fetches them again for the new guest (PostHog `reset()`). They survived logOut and served the next user on the device. `logIn` still keeps them.

- In-app messages: on a fullscreen message the title and the first line of the body no longer run under the ✕. The ✕ floated over the card 32 dp down; it now has its own row (a 48 dp target, read as "Close" by TalkBack) and the content starts below it. The fullscreen also no longer draws its title under the status bar (the window is translucent-status, and the white status icons vanished on the white background).
- In-app messages: an Open URL (`deeplink`) button opens the URL in an app that never called `BillDogInAppMessaging.registerActivityLifecycleCallbacks` — every React Native and Flutter app, and any native app following the README. It opened nothing while the click result reported `InAppMessageUrlTarget.BROWSER`. It now uses the Activity in front that core tracks, as showing a message already did.
- In-app messages: a message with no image no longer draws an empty 180 dp image box, and a button with no URL reports `url = null` to the click-result handler, not the string `"null"`. The server sends `"image_url": null` / `"url": null`, which Android's `JSONObject.optString(key, null)` reads as the string `"null"`.
- A module's event on the in-process `BillDogEventBus` reaches each subscriber once. The analytics module records a bus event through `track()`, and `track()` broadcast every event back onto the bus — the one it had just taken from it too — so an event emitted once without the recorded-elsewhere marker (`survey_dismissed`, paywall and purchase lifecycle events) was delivered to every other subscriber twice: React Native's `onSurveyDismissed` fired twice for one survey close. An event taken from the bus is no longer broadcast back (iOS: `track(event:fromEventBus:)`); an app's own tracked event is still broadcast once, for event-triggered surveys. Each event is still recorded once.
- A purchase through `billdog-integration-superwall` is attributed to the BillDog offering behind the Superwall paywall on screen, when `SuperwallBillDogPurchaseController(offeringMap = …)` names it (Superwall paywall identifier → BillDog offering identifier); the Play offer Superwall selected still applies. Superwall presents its own paywall, so no BillDog paywall was on screen and a product sold in more than one offering (experiment arms, a seasonal offering) was credited to the first loaded offering selling it. A purchase with no mapped paywall on screen, or whose mapped offering is not loaded or does not sell the product, is attributed as before (the last two log a warning). New: `PurchaseParams.presentedOfferingContext` (`Builder.presentedOfferingContext`) names the offering a `purchase(activity, params)` is attributed to, ahead of the first loaded offering selling the product, as a package's own context does (RevenueCat `PurchaseParams.Builder.presentedOfferingContext`). Code compiled against 1.1.1-beta.3's nine-property `PurchaseParams` constructor and `copy` still links.

### Added (also in this release)

- `BillDog.setProxyURL(url, trustedForProofs)` and `BillDogConfig.setProxyURL(url, trustedForProofs)` (`@JvmOverloads`, default `false`): an opted-in proxy also gets the customer proofs (the BillDog Auth session, the device key), so customer-scoped calls such as in-app account deletion work through a proxy you run. Behind a proxy the SDK withheld them, and those calls answered `PROOF_REQUIRED`. The proofs go only to the proxy's exact scheme, host and port, over https (plain http only to a loopback or private-network host in a debug build of the SDK); otherwise the trust is refused and logged, and the proxy still routes. Setting the proxy again without the flag, or clearing it, withdraws the trust; WorkManager workers follow the app's choice. `setProxyURL` now returns whether the trust was granted, and a refusal made before `configure` is logged once `configure` turns logging on.
- Feature flags record `$feature_flag_called` (`$feature_flag`, `$feature_flag_response`, and `$feature_flag_payload` when the flag is on and has one) once per user, flag and value per process, from the flag getters, as iOS and PostHog do; `getFeatureFlagPayload` does not report, and nothing is recorded without the analytics module. Flag results count only this event, so native Android flag results were empty. Set `sendFeatureFlagEvents = false` on the `BillDogABTest` module to turn it off. New on the module: `isReady` and `waitForFlags(timeoutMs)`.
- `PlacementResult.OAuthSuccess`: `register` reports a wall that closed after a verified sign-in on it (it was a dismissal). `SkipReason.wireValue` gives the skip reason's cross-platform name.
- `BillDogInAppMessages.queueCount`, `showNextMessage()` and `resetFrequencyCaps()`, as on iOS.
- `BillDogNotifications.handleNotificationIntent(intent)`, for a push tap that reaches a running Activity (`onNewIntent`).

### Fixed (also in this release)

#### Identity and auth

- `logOut()` ends a BillDog Auth session held for the signed-in person. The new guest kept sending that session, so the server answered the guest as the signed-out person (their customer info, and any purchase, push token or attribute write), and the session stayed live. It is now signed out locally and revoked on the server.
- `reset()` ends the BillDog Auth session and revokes it on the server. Android kept the session on purpose, and the new guest's requests carried the previous person's session. A feature-flag or customer-info answer that was in flight for the previous user is never applied to the new one, and the A/B module's `logOut` also drops the targeting attributes, as iOS does.
- A BillDog Auth sign-in (a wall's sign-in button included) refreshes customer info at once. The SDK kept the guest's cached customer info until its next refresh, so a returning subscriber who signed in on a wall stayed unentitled and a gated `register` showed them the paywall again.
- An app that configures while a screen is already showing (every React Native and Flutter app, an app that shows a consent screen first) reads the customer's access at launch. `subscriptionStatus` stayed `UNKNOWN`, or the last stored status, for the whole session.
- `setUserAttributes` saves custom attributes on the server. Every write was answered "No attributes to update" and nothing was stored. Values are now kept on disk per user until the server has them, sent together, resent with the next write and on foreground or launch if a write failed (a rejected value is not resent), and merged on the server; `null` removes a key, and `getUserAttributes` returns the stored values.
- Google sign-in: the native flow asks for the ID token with the project's Web client id, as Google requires. With an Android client id saved it failed every time with `ApiException 10` (`DEVELOPER_ERROR`). With no native Google configuration, Google sign-in takes BillDog's hosted redirect, as iOS does, instead of failing with "provider not configured".
- A sign-in on a wall that fails says so on the button ("Couldn't sign in. Try again.", localizable) for a few seconds; the spinner just stopped before. A user cancel shows nothing: the SDK now tells the wall when the user cancelled.
- BillDog Auth reads a JSON `null` as null, not the string `"null"`: a pending TOTP factor no longer reads as verified, and clearing the active organization answers `null`.
- A paywall hosted in an Activity that cannot launch the Google sign-in screen (Flutter's inline paywall) no longer crashes; that sign-in answers an error.

#### Purchases and paywalls

- Buying a product the user already owns, and that was already recorded, fails with `AlreadyOwned`, as RevenueCat answers. It came back as a success with the old order id: the paywall closed as if it had sold something and `purchase_completed` fired again.
- An explicit purchase always records the transaction on the server. A transaction an earlier automatic sync had been refused for was answered from the local record, so the purchase reported success without reaching the server.
- `restore()` returns each `RestoredPurchase` with its Play order id as `transactionId`; it was empty.
- A Purchase action with no product, inside a button or package that names one, buys that product. Tapping an "Annual" button bought Monthly when its label sat in a stack with a product-less Purchase.
- A Purchase with no product, on a wall with several packages and nothing selected, buys nothing and reports a failed purchase (logged), instead of buying the offering's first package. A wall with exactly one package still buys it, and a products component's selection still applies.
- The delegate's `onPaywallSkipped` hears a skipped placement (entitled, holdout, no paywall configured) from the closure `register` too, not only from the presentation-handler variant.
- `SubscriptionStatus.name` is the cross-platform value (`active`, `trial`, `gracePeriod`, `expired`, `notSubscribed`, `unknown`), as on iOS; it was `ACTIVE`, `GRACE_PERIOD`, …. The `subscription_status_changed` event carries the same values. A status saved under the old name still restores.
- `SubscriptionManager.previewUpgrade` (and its server-only upgrade / downgrade siblings) fail at once with a server-only error, as on iOS; use `BillDog.upgradeSubscription` / `downgradeSubscription` to change the Play plan. `PresentationResult.Error` keeps its exception.
- `billdog-core` keeps Room databases' no-argument constructor for apps minified with R8 on AGP 9 (strict full mode). Those apps crashed at start with `NoSuchMethodException: androidx.work.impl.WorkDatabase_Impl.<init>`.

#### Feature flags

- Flags are evaluated for `setUserId`'s id, else the current user, else the install's anonymous id, and every getter (`getFeatureFlags` included) starts this launch's fetch when none was asked for.
- `logIn` reloads flags for the signed-in user. An answer in flight for the guest was dropped and nothing asked again, so flags stayed empty for the launch and `waitForFlags` returned false. Flag requests name the platform, SDK version and app version, so the dashboard's platform and app-version conditions match.
- `ab_variant_assigned` carries `variant_name`, `is_control` and `config`.

#### Analytics

- `BillDogConfig.exceptionAutocapture = true` and `autocaptureEnabled = true` turn those features on; set locally, they did nothing unless the dashboard also enabled them.
- With analytics consent declined, `$revenue` events are no longer uploaded. Events are classed by name, as on iOS and the server; the `$transaction_id` property made them count as essential.

#### Push and journeys

- Push delivery and opens reach the dashboard: the SDK reports each delivery (`notification_delivered`) under the push's BillDog id, and a tap is recorded from the Activity intent it opens (click handler first, then `onNotificationOpened`), with a tap the app also reports counted once.
- Push action buttons work (up to three, each with a title and URL), and a push's action URL, or the tapped button's, opens on tap; `opensActionUrlOnTap = false` turns that off.
- A tap that arrives before a click handler is set is held and given to the first handler (the latest one only); `clearClickHandler()` is new.
- `optedIn` survives a relaunch: it follows the saved opt-out and the OS permission. The permission status is `NOT_DETERMINED` until the prompt was shown. An opted-out install no longer turns its token back on at the next token callback.
- The push token moves to the new customer on `logIn` / `logOut` / `switchUser`.
- Post-purchase reminders are opt-in: `BillDogNotifications.autoSchedulesPurchaseReminders` (default `false`). A purchase used to schedule a local renewal or expiry reminder by itself, in every app that registers the notifications module (every Flutter, React Native and Expo app). Reminders you schedule yourself are unaffected, and a purchase still cancels the reminders of what was bought.
- The notifications module reads and writes the SDK's consent (the `messaging` purpose) instead of its own copy.
- In an app that configures after its first screen (every React Native and Flutter app), journey paywall and survey steps, a push tap that opens a paywall, and event-triggered surveys use the screen in front. A journey step was held for the whole session, an `open_paywall` push tap did nothing, and event-triggered surveys never armed.
- A journey step that arrives with no screen to show it on (a cold start, before the first screen resumes) is kept and shown when one resumes; it was handed back and never delivered again. A journey survey step its gating declines is reported, instead of parking the journey until it expired.

#### In-app messages

- Messages show in an app that configures after its first screen, without `registerActivityLifecycleCallbacks`. They sat in the queue ("No Activity available, cannot display message").
- A bottom-sheet message can be swiped away (past a third of its height, or a downward fling), like tapping outside or Back.
- A button tap reports `closingMessage = true` for every button (the message always closes), not only for a dismiss button.

#### Surveys

- Survey state works on Android. The presenter `BillDog.getSurveyPresenter()` returns was built without the part that keeps it, so event triggers never armed or fired, `show_once` / recurrence / wait periods never applied, drafts were never saved, a failed submit was never queued for retry, and `hasCompletedSurvey` was always false.
- A respondent who answers part of a survey and closes it keeps those answers: partial answers are sent page by page (as on iOS), one at a time, and submit and abandon wait for them.
- A survey closed while its submit is still in flight is completed, not abandoned, and the app hears `onComplete` (or `onError`) before `onDismiss`.
- Event-triggered surveys (`registerEvent`, armed triggers) announce their dismissal and completion, so the wrappers' survey listeners and analytics hear them.
- The survey sheet's Back / Next / Submit sit above the navigation bar (they were drawn under it, and with 3-button navigation could not be tapped), the sheet opens at its full height, and the NPS row fits the screen (9 was cut off and 10 could not be chosen).
- Rating questions draw the number of stars the builder set (`scale`), and text questions show the builder's placeholder and multi-line setting.
- TalkBack reads each rating star as "N of M stars" with its selected state, emoji ratings as "N of M", thumbs as "Thumbs up" / "Thumbs down", and the close button as "Close" (localized). It read "★" and "✕".
- WebView surveys load in apps that target API 30 or later.

## [1.1.1-beta.4] - 2026-10-24

### Fixed

- `downgradeSubscription` (and any plan change with `GoogleReplacementMode.DEFERRED`) returns: under DEFERRED Play answers with the subscription being replaced, and BillDog waited for the new product, so a successful downgrade never resolved and no `DOWNGRADE_SUBSCRIPTION_COMPLETED` fired. Play's answer is now matched to the billing flow BillDog launched (RevenueCat parity), which also keeps that flow's user and paywall attribution from attaching to a later purchase or renewal, and stops a later, unrelated billing error from failing it.
- `restorePurchases()` fails when Play cannot be read (billing unavailable, an error answer) with `BillDogError.StoreUnavailable`; it answered `Restored` with nothing in it. When the device is offline and BillDog's server kept none of the restored purchases, it fails with `BillDogError.NetworkError` to retry; the purchases are saved and still sent when the device is back online. Without `billdog-paywall` linked, BillDog's own restore fails as `StoreUnavailable`.
- `purchase(activity, package)` attributes the purchase to that package's own offering and placement (`Package.presentedOfferingContext`), in the `purchase_*` events and the recorded purchase. When the same product was sold in more than one loaded offering, it was credited to the first package found selling it (RevenueCat and iOS parity).
- A purchase from a paywall (WebView or Compose, including an upgrade or downgrade from it) is attributed to the offering that paywall rendered. A paywall buys by product id, and when that product was sold in more than one loaded offering (experiment arms, a seasonal offering), the purchase was credited to the first offering found selling it; the paywall id was right. `BillDog.claimPaywallAttribution` gains overloads naming the rendered offering (an `Offering`, or its id and placement id) for a custom rendering surface.
- Analytics applies the dashboard's **Session timeout** (default 300 s): an event after that long without one, or after 24 hours, starts a new session, and returning to the app after the timeout records `$session_start` for it. The setting was read and never used; a session lasted as long as the process. The session also outlives the process: Android kills backgrounded apps freely, and a launch within the timeout continues the visit instead of starting another. Each session gets exactly one `$session_start` — however it began (a launch, the timeout passing while the app stayed open, an opt-in) — and `$app_backgrounded` / `$session_end` durations run from the session's start. The timeout (the client override or the dashboard's) is clamped to 60 s – 24 h: 0 made every event its own session.
- A journey action the guest's poll delivered is not delivered again after `logIn` (or `switchUser` / `logOut`): each poller kept its own in-flight set, so the signed-in user's poll — the guest adopted into the account — delivered the same action, and the app ran one journey step twice. The pollers now share one ledger, as on iOS; only `reset()` empties it.
- Journey actions across an identity change: after `logOut` (or `logIn` / `switchUser`), an answer to the previous identity's poll still in flight is no longer delivered — the signed-out user's step reached the next guest.
- Fonts from an older paywall config's family map (`resources.fonts: { "Inter": { weights: [...] } }`) load: only the server's array was read, so such a paywall drew every custom family in the system font. Read as iOS reads it (`tests/parity/font-faces.vectors.json`).
- A font face with an explicit JSON `null` (`"url": null`) is read as absent; it became the URL "null". A face's weight may be a decimal or a numeric string.
- A paying subscriber who is offline at a cold launch gets through a gated placement. Each placement's last-known gate (`gated` + `required_entitlement`) is kept per user on the device; when the server cannot be reached (an `IOException`: no connection, an unknown host, a timeout — never a server answer such as the 403 withheld or a 5xx) and that gate is a gate, the customer info on the device decides it: holding the required entitlement (or any active one when it names none) reports `PlacementResult.Entitled` / `onSkip(USER_HAS_ENTITLEMENT)`, runs `registerFeature`'s closure, `getPresentationResult` says `UserSubscribed`, `getPaywall` throws `PaywallSkippedException(USER_HAS_ENTITLEMENT)` and `hasAccessForPlacement` answers `true`. Anything else still fails as before; an ungated or never-resolved placement is never unlocked offline, and one user's gates never answer for another (Superwall `ConfigState` falls back to its persisted config the same way).
- Background work (analytics sync, receipt sync, the Play acknowledgement deadline) in a process WorkManager starts without your `configure` — always so for React Native and Flutter apps — routes through the proxy you configured; it went straight to BillDog. The WebView survey page's API base follows the proxy and the environment (it was the production URL), and relative paywall asset paths resolve against the proxy's host when one is set.
- A `setProxyURL` URL that does not end in `/v1` gets it: a bare host (`https://proxy.example.com`) was used as given and missed the version every BillDog route is under. A URL ending in `/v1` is unchanged (`BillDogConfig.functionsBase`, the same rule as iOS and React Native).
- `BillDogConfig.localeOverride` takes effect: `BillDog.configure` applies it as `BillDogConfig.setLocaleOverride` does (iOS's config always did). Nothing read it — every paywall, survey and the localization resolver read the runtime override — so a configured locale did nothing. A blank one is no override.
- The background analytics sync (a process WorkManager starts without the app's `configure()` — always the case for React Native and Flutter apps) applies the app's `BillDogConfig` analytics overrides from its last configure. It rebuilt its config from the API key alone, so it sent queued events for an app that had set `analyticsEnabled = false`.
- `survey_presented` was recorded twice per presentation when analytics was registered (once from the event bus, once by the survey presenter); it is recorded once. The bus copy of `survey_completed` (recorded server-side) is no longer sent either. Bus events whose record is made elsewhere carry `BillDogEventBus.recordedElsewhere` / `RECORDED_ELSEWHERE`.
- Customer info belongs to the user it was read for. An answer that lands after its user signed out or switched away (a read already in flight at `logOut`) is dropped: it no longer became the new guest's `getCustomerInfo()`, `hasEntitlement` and placement gate, reached the delegate and `customerInfoStream`, or was written back to disk. A `logIn` whose read fails shows the new user their own stored customer info or none — never the previous user's — and `customerInfoStream` no longer replays the previous user's to a new subscriber.
- A launch restores the current user's stored customer info before anything reads it (RevenueCat parity): a subscriber is entitled at once, `getCustomerInfo()` answers offline from disk instead of failing, and `entitlements` / `subscriptionStatus` name the stored entitlements. The disk copy was written on every read and never read back.
- `getCustomerInfo()` (the default cached-or-fetched policy) refreshes in the background only when its cached answer is five minutes old, as RevenueCat does; it started a server read on every call. `refreshSubscriptionStatus()` always reads the server.
- `switchUser` and `invalidateCustomerInfoCache()` keep every other user's stored customer info (they wiped them all), so switching back answers from disk. `logOut` still wipes every user's.
- `subscriptionStatus` and `entitlements` are always the current user's: `logIn`, `switchUser`, `logOut` and a new guest set them from that user's stored customer info or stored status, or `UNKNOWN` when there is none, instead of keeping the previous user's until a refresh. The last-known status is stored per user (`logOut` wipes all of them); the device-wide status an earlier version stored, whoever it belonged to, is no longer restored at launch.
- A failed `refreshSubscriptionStatus()` keeps the last-known status: it published and stored `UNKNOWN`, so the next launch restored no status at all.

### Added — products naming a Play base plan (`subscriptionId:basePlanId[:offerId]`)
- A BillDog product's Play id may name one base plan of a Play subscription, as RevenueCat (`productId:basePlanId`) and Superwall (`productId:basePlanId:offerId`) write it: `premium:monthly` and `premium:yearly` are two products, each showing and buying its own base plan's price, period, free trial and intro price, and `premium:yearly:intro-50` also pins an offer (the base plan when the user is not eligible). Play is queried with the subscription id; its purchase, which names only the subscription, answers the purchase of the base plan asked for, under that id (callbacks, attribution, `PurchaseResult.productId`). A base plan Play does not list is unavailable, never swapped for another. A bare id keeps choosing among every base plan. Restore reports Play's subscription id (Play's purchase carries no base plan); the server maps each purchase to its base plan's product. New public type: `io.billdog.core.models.PlayProductId`.
- A plan change (`upgradeSubscription`, `PurchaseParams` with `oldProductId`) buys the offer a purchase of the same package would: its base plan, its pinned offer (`playOfferId` or the id's `:offerId`), else the default rule (longest free trial, else cheapest intro, skipping `-ignore-offer`). It bought whichever offer Play listed first — a win-back or another base plan's offer included.

### Changed — a product Play does not sell fails on every key; simulating is opt-in
- A test key used to turn "Play returned no product" into a synthesized success (a
  `sandbox_<uuid>` transaction): the app was told it purchased, its paywall closed as
  purchased, post-purchase actions ran and `purchase_completed` was sent, while nothing was
  charged and no entitlement was ever granted. It also hid real misconfiguration (a SKU
  Play does not sell) exactly while the app was being tested. That fallback is removed: the
  purchase fails, on every key, with an error naming the id Play was asked for and what to
  check.
- New: `BillDogConfig.simulatedPurchases` (default false), RevenueCat Test Store parity. On a
  test key every purchase is simulated: no billing sheet, nothing charged, and BillDog records it
  on the server as a simulated SANDBOX purchase that grants its entitlements, so the app's
  customer info matches what it was told. Results carry a `sim:` transaction id, every
  purchase event carries `simulated: true`, and none carries revenue. Ignored with an error
  on a live key; a `purchaseController` passed to `configure` wins over it. Needs the
  `record-simulated-purchase` edge function.
- A sandbox purchase is no longer also posted to `sdk-create-test-purchase` (the dashboard
  test-store table). Real sandbox purchases are recorded once, in the purchase ledger.

### Fixed — a Paywall Builder purchase button asked Play for the wrong product
- The Builder's Purchase action names its product by the BillDog `product_identifier`
  (`sbx-sub-monthly`), which Play does not sell; the Play SKU is `play_store_product_id`.
  Only a package identifier was mapped to the SKU, so the tap queried Play with the BillDog id:
  product-not-found on a live key, and on a test key the sandbox simulator's synthesized
  success, with no billing sheet and nothing recorded.
- Every purchase path now resolves its product reference to the SKU before Play Billing:
  `purchase(activity, productId)`, `purchase(productId)`, `purchase(activity, params)`,
  `purchase(activity, package)`, both paywall Activities, and the observer-mode
  `observePurchase*` calls. A SKU, a package identifier, a `product_identifier` or a
  `products.id` all resolve, across every loaded offering and the offering a placement wall
  sells; an id no package knows is used as given, as the SKU.
- `purchase(activity, package)` for a package with a Play offer (`playOfferId`) sent the bare
  package identifier when the package had no loaded Play product; it now sends the SKU.
- The purchase's offering attribution and revenue follow the resolved SKU.
- A WebView tap with no product falls back to the wall's first package by its Play SKU, not its
  BillDog id; a tap naming only its `packageId` purchases that package.

### Fixed — test walls and attribution follow the purchased SKU
- Draft (`fetchPaywallDraft`) and preview-link (`fetchPreviewPaywall`) fetches now name the
  platform, so the server serves their purchase actions by the Play SKU, as it does for a live
  wall. Previously only the SDK's own resolver mapped a draft or preview wall's purchase.
- A purchase's offering attribution is resolved through the same `StoreProductReference` as the
  purchase, then credited to the package selling that SKU. The lookup had its own matcher, which
  knew no Builder `product_identifier` or `products.id`, so attribution and the purchased SKU
  could disagree.

## [1.1.1-beta.3] - 2026-10-17

### Fixed — every package of a bound product list is shown
- A product list bound to the offering (`bindToPackages`) drew only its authored cards, so a
  package added to the offering (a new lifetime plan, say) never appeared until someone added a
  card in the builder. A package beyond the authored cards now gets a copy of the last card:
  its own ids (`__pkg<n>`) and selection state, filled from its package like any card. Fewer
  packages than cards still shows the first N, and a copy of the badged card shows no badge.
  The same rule runs on Android, iOS and the web, pinned by a shared conformance suite.

### Fixed — a one-time product's card shows no period
- A lifetime or other one-time product's period is empty; its card showed a bare "/". It now
  shows nothing, as on the web.

### Fixed — shadows and theme colours drawn as designed
- A shadow is drawn once. A second, rectangular pass drew every authored shadow twice.
- Shadow geometry authored as `{value, unit}` (what the builder exports) is read; it was
  dropped.
- An inner shadow is not drawn as an outer one, and a negative-offset hairline (`0 -1px 0`) is
  drawn.
- Text resolves its colour with the theme, so a semantic token draws in its own theme colour
  instead of plain foreground.

### Removed — `ImmediatePaywallHandler`
- **BREAKING**: `ImmediatePaywallHandler` and `ImmediatePlaceholderView` (billdog-webview) are removed.
  Nothing in the SDK called them, and nothing stored the "immediate" paywall ids that
  `shouldShowImmediately` read, so it always answered `false`. `presentImmediately` replaced the
  host Activity's content and bypassed the presentation path (no host callbacks, no stacking, no
  purchase attribution). Present paywalls with `BillDog.presentPaywall` / `register`.

### Fixed — signing in keeps the user's cached paywalls
- `logIn` and `switchUser` keep the incoming user's own cached paywalls (memory and disk) and
  drop every other identity's. They cleared the whole cache, although it is keyed by user; and
  identity starts each launch as the install's guest, so an app calling `logIn` at every launch
  discarded the disk cache the user built last session, and its first paywall went to the
  network. `logOut` still clears everything.

### Changed — WebView paywalls open on a renderer page started ahead of time
- Once the app has fetched a wall that draws in a WebView, the SDK boots one renderer page off
  screen, and the WebView paywall screen shows that page and only injects the paywall. Measured
  on a Redmi Note 11 Pro+: the paywall's data is on screen 139–228 ms after its screen starts,
  against 321–419 ms when the screen boots its own page.
- One page is kept; the next is booted after one is shown. It outlives the app going to the
  background, is released under memory pressure and on `reset()`, and is booted again when the
  app returns to the foreground. `PaywallPrerenderer.enabled = false` turns it off.
- The pool no longer builds a hidden WebView for every fetched wall (native walls included):
  no screen ever showed them.

### Fixed — a WebView paywall's images and fonts are downloaded ahead
- When a WebView wall is fetched, its images and fonts (and Google Fonts stylesheets with the
  font files they name) are downloaded to the app's cache directory, and its WebView is served
  them from disk. Nothing warmed them before — the WebView pool loads only the renderer — so
  the first time a user saw a WebView wall, its images and fonts arrived after the page.
  Bounded to 30 MB (5 MB per file), kept for 7 days; `cacheConfig.preloadAssets = false` skips
  the images. Anything not downloaded is fetched by the WebView as before.

### Fixed — a native paywall's images warmed before display
- Every image a paywall's server manifest lists (every device layout's, up to 64) is warmed.
  The list was cut to 12 after every URL had been marked warmed, so images 13+ were never
  requested — by that fetch or any later one — and drew cold, e.g. after a rotation.
- An image that failed to warm is tried again on the next fetch; it stayed marked warmed.
- An image is warmed again on its wall's next fetch whenever Coil no longer holds it in memory.
  It used to count as warm for the life of the process, though Coil empties its memory cache when
  the system tells a backgrounded app to trim memory (and halves it when memory runs low), and its
  disk cache trims itself to its size: the paywall then showed an empty space while it decoded the
  image, or downloaded it. Whether an image is ready is now asked of Coil's memory cache, under
  the key the paywall's own request uses: in memory → skipped; bytes still on disk → decoded from
  there with no request; neither → downloaded.

### Fixed — a paywall's offerings, prices and trial eligibility are ready before it opens
- Offerings are served from cache and refreshed in the background once older than 5 minutes
  (up to a day), instead of making the next paywall wait for a refetch. They are refreshed on
  return to the foreground and for a newly signed-in user, and another user's are never served.
- Trial eligibility is fetched with the offerings — at configure, on every refresh, for a new
  user and after a purchase or restore — so a paywall no longer waits for it. It is kept per
  user (another user's answer was served for up to 5 minutes after a sign-in) and refreshed in
  the background the same way.
- Trial eligibility is asked again whenever the caller's active entitlements change, however
  they changed — a purchase here or on another device, a restore, a web redemption, a renewal,
  an expiry, a `purchase()` the store answers as already owned — not only after a purchase
  made here. An answer that was being fetched when the purchase landed is not kept.
- A placement's offering is enriched from the Store products already fetched; only products
  not yet known are asked for. Every presentation used to query Play Billing again.
- Which renderer page a WebView paywall loads is decided once per downloaded renderer, off
  the main thread, instead of reading and scanning ~1.5 MB on the main thread at every open —
  for the screen's own page and the page started ahead of time alike.

### Changed — `register` presents a preloaded placement at once
- `register` presents the answer it kept for the placement — from `preloadPaywalls` or the
  previous `register`, for the same user, params and request context, up to 30 minutes old —
  without waiting for the server, and asks the server again in the background so a targeting
  change reaches the next presentation. It asked `get-wall-for-placement` on every call, even
  right after a preload, and nothing showed until the reply came back.
- Kept answers are dropped when the user, their attributes (`setUserAttributes`), their consent
  or their active entitlements change — however they changed: a purchase here or on another
  device, a restore, a web redemption, a renewal, an expiry. An answer being fetched when one of
  these changed is not kept. A gated wall's entitlement gate is still checked on the device at
  every `register`, and the server still records experiment exposure when it answers.

### Changed — plan changes and what a purchase could bill beside (purchase path)
- **New**: `PurchaseController.heldSubscriptions(): SubscriptionLookup` (`Held` / `None` / `Unknown`).
  BillDog asks it when it cannot read Play itself (billdog-core without billdog-paywall, or Play's
  query failed). The default is `Unknown`, so a purchase that depends on it fails closed rather than
  assume nothing is held. `RevenueCatAdapter` answers from a fresh `CustomerInfo` read
  (`FETCH_CURRENT`, bounded to 10 s). That needs RevenueCat ≥ 9 at runtime:
  `subscriptionsByProductIdentifier` does not exist in RevenueCat 8, which now answers `Unknown`
  instead of crashing.
- **BREAKING (behaviour)**: `upgradeSubscription` / `downgradeSubscription` replace exactly
  `currentProductId`. When that product was not held they used to replace the first active
  subscription, a plan the caller never named; now nothing is bought and a `ConfigurationError` names
  what is held. With no `currentProductId`, the one subscription that can be replaced is; with
  several, or none replaceable yet, nothing is bought. `PurchaseParams.oldProductId` follows the
  same rule.
- **BREAKING (behaviour)**: `PurchaseParams.replacementMode` without `oldProductId` is refused while
  a subscription is held, and whenever BillDog cannot tell what is held: no billdog-paywall and a
  purchase controller that does not implement `heldSubscriptions()` (`ConfigurationError`), or Play
  or the controller could not be read (`StoreUnavailable`). It used to be dropped silently, so such
  a purchase went through, including a first purchase through a non-reporting host controller,
  which now fails. Migrate by implementing `heldSubscriptions()` in your controller (the RevenueCat
  adapter does), linking billdog-paywall, or leaving `replacementMode` unset for a purchase that
  replaces nothing. With nothing held, the mode is ignored as before.
- A subscription whose payment is pending (prepaid plans) or that is not yet acknowledged counts as
  held: a `replacementMode` purchase is refused beside it, and it cannot be replaced until active.
- A request carrying `oldProductId` needs `PurchaseCapability.UPGRADES`, as one carrying
  `oldPurchaseToken` does. A replacement BillDog learned from a host carries `oldProductId` and no token.

### Changed — `PurchaseParams`
- `isPersonalizedOffer` is now Play's EU personalised-price disclosure: the caller's value, or, when
  unset, disclosed for EEA storefronts. It used to be ignored.
- New `oldProductId`: with `replacementMode`, a real Play replacement of that subscription. A
  `replacementMode` used to launch a plain purchase, a second subscription.
- `appAccountToken` is `@Deprecated` and ignored: BillDog binds every Play purchase to the
  customer's issued store-account token.
- Binary compatible with 1.1.0-beta.3: its eight-property constructor and `copy` remain, hidden.

### Fixed — a web purchase is redeemed only when the server says so
- `redeem-purchase` counts as a redemption only on an explicit `data.redeemed: true`. A 2xx without
  it (an empty body, `{}`, `data: null`, a `data` with no `redeemed`) used to read as redeemed, so
  `redeemAndNotify` reported `Success` and the pending token was forgotten. It is now handled as an
  answer that did not redeem: retried, then an error, and the token kept.

### Fixed — module registration order
- `BillDog.use(module)` works before or after `configure`: a module registered first now gets the
  network client and the config. `VirtualCurrencyManager` registered before `configure` used to fail
  every call with "BillDog is not configured".

### Fixed — customer-scoped writes
- `setDeviceToken`, the attribution setters, `setCustomerEmail` / `setCustomerPhoneNumber` /
  `setCustomerDisplayName` and `setDeviceIdentifiers` address the current app user id. They work right
  after `configure`, `logOut` and `switchUser` with no `getCustomerInfo()` first; between `logIn` and
  its fetch they used to write to the previous user.

## [1.1.0-beta.3] - 2026-10-12

### Fixed — `$app_open`
- `$app_open` is now sent: once per launch, when the app is first in the foreground (or at
  configure, if an activity is already on screen). The cold-start check read a session time
  that configure had already set, so every launch arrived as `$app_foregrounded` and an
  "App Opened" journey enrolled nobody.

## [1.0.2-beta.3] and earlier - 2026-10-03

### Changed — acknowledgement deadline, RevenueCat ≥ 9, adapter contract (PR-D4)
- Under BillDog's own controller, `PlayAckDeadlineWorker` acknowledges any Play purchase still
  unacknowledged 48 hours after it was made, so an outage never triggers Google's 3-day
  auto-refund. A purchase the server refused to record is left for Google to refund.
- **BREAKING**: `billdog-integration-revenuecat` requires RevenueCat Android ≥ 9 (Play Billing 8).
  The adapter returns store evidence (purchase token, order id, products, JSON, signature) and
  forwards RevenueCat's non-subscription purchases; entitlement mapping is removed.
- **BREAKING**: `BillDogPurchaseAdapter.getEntitlements`, `onEntitlementsChange`,
  `removeEntitlementsListener`, `BillDog.setEntitlements` and `BillDog.setPurchaseController` are
  removed. BillDog never accepts vendor entitlements; register an adapter only through
  `BillDog.configure(..., purchaseController = adapter)`.

### Changed — purchase recording (store-evidence ledger, PR-D3)
- **BREAKING**: purchases are recorded by posting Play store evidence (token, order id, products,
  `originalJson`, signature) to `record-store-evidence` through a durable Room-backed recorder.
  `PurchaseQueue`, `ReceiptProcessor`, `DeadLetterStore`, `BackoffPolicy` and `ApiClient.postReceipt`
  are removed; receipts an older SDK left queued are carried over by the v3→v4 database migration.
- **BREAKING**: a refused recording is `PurchaseResult.Failed` (`PurchaseOwnerConflict` /
  `ReceiptValidationError`), never success. The trial gate and its "acknowledge then deny" path are gone.
- BillDog acknowledges / consumes only under its own controller with `finishTransactions = true`, and
  only after the server says so. With a host controller it never acknowledges or consumes.
- Every purchase entry point runs through the controller's new `purchase(activity, request)`
  (`PurchaseRequest` / `PurchaseOutcome` / `RestoreOutcome`, `capabilities`); a request needing an
  undeclared capability (quantity, win-back, an upgrade) fails with a typed error. A host's
  purchase is recorded from its evidence, else derived from the store's purchases.
- `setObfuscatedAccountId` carries the customer's issued store-account token, never the app user id;
  the `"anonymous"` fallback is gone.
- `billdog-core` compiles against the base `billing:8.0.0` artifact; `io.billdog.store.PlayStoreObserver`
  observes Play for core-only and billdog-eng apps without billing-ktx.
- New: `syncStorePurchases()` (`syncPurchases()` deprecated), `recordPurchase(StoreEvidence)`,
  `storeAccountToken`, `purchaseRecordingIssues`, `BillDogDelegate.onPurchaseRecordingIssue`,
  `BillDogPurchaseAdapter.attach(host)`.

### Added
- Initial public release
- Jetpack Compose paywall rendering
- AI-generated paywall support
- Native component library
- Variable substitution
- Theme customization
- A/B testing support
- Real-time analytics tracking
- Multi-language localization
- Dynamic remote updates
- Material 3 theme integration
- Advanced Caching System with PaywallCacheManager
- GZIP compression (60% storage reduction)
- Multi-level caching (memory + SharedPreferences + internal storage)
- Intelligent prefetching based on access patterns
- Real-time cache analytics
- Background version checking
- Offline-first architecture
- Kotlin coroutines integration

### Component Support
- Text components with custom styling
- Image components with Coil integration
- Button components with Material Design
- Stack layouts (vertical/horizontal)
- Package subscription cards
- Feature lists with checkmarks
- Price displays with formatting
- Badges and labels
- Dividers and spacers
- Icon components (Material Icons)
- Video components with ExoPlayer

### API Features
- `PaywallView` composable - Main rendering component
- `PaywallConfiguration` - Configuration data class
- `PaywallAPI` - API client with coroutines
- `PaywallViewModel` - State management
- Kotlin coroutines support
- Flow-based reactive updates
- Error handling
- Loading states

## [1.0.0-beta.1] - 2026-06-16

### Added
- First stable release
- Complete documentation
- Example application
- Unit test coverage
- CI/CD workflows
- ProGuard rules

## Future Roadmap

### Planned for 1.1.0
- [ ] Animation customization
- [ ] More component types
- [ ] Enhanced analytics

### Planned for 1.2.0
- [ ] Wear OS support
- [ ] Android TV support
- [ ] Widget extensions
- [ ] Glance widgets
- [ ] Compose Multiplatform support

---

[Unreleased]: https://github.com/billdogai/billdog-android/compare/v1.0.0-beta.1...HEAD
[1.0.0-beta.1]: https://github.com/billdogai/billdog-android/releases/tag/v1.0.0-beta.1
