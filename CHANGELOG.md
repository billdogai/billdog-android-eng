# Changelog

All notable changes to the BillDog Android SDK will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
