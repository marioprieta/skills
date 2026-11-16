---
name: react-native-theme-transition
description: "Animated dark mode and theme transitions for React Native, v2 Skia engine with 9 transition styles. TRIGGER when: code imports react-native-theme-transition, user works with createThemeTransition/ThemeTransitionProvider/useTheme, implements animated dark mode toggling, system theme following, theme persistence, React Navigation or Expo Router theme integration, picks between fade/circularReveal/heart/star/wipe/slide/split/pixelize/dissolve, or debugs transition issues (stuck overlay, flash, capture failure). DO NOT TRIGGER when: general React Native styling, non-animated theme switching, or web-only theming."
license: MIT
metadata:
  author: marioprieta
  version: "2.0.3"
  documentation_commit: "afc5e6585d0ad5c6de90c067ff6c3ea01ba13adb"
  tags: react-native, theme, dark-mode, animation, transition, expo, reanimated, skia, view-transition
---

# react-native-theme-transition (v2)

Animated theme transitions for React Native, powered by `@shopify/react-native-skia`
and Reanimated. The engine snapshots the inner view tree into a Skia `SkImage`,
mounts it on a permanently-mounted Skia Canvas, swaps the theme tokens
underneath, and animates the snapshot away with one of nine styles.

This skill and its references follow the library's 2.0.3 documentation.
For older installed packages, inspect their version and API before applying
the current engine details. Migration guidance is explicitly labeled v1 → v2.

## Installation

Expo (install native peer versions matched to the app's SDK):

```bash
npx expo install react-native-theme-transition @shopify/react-native-skia \
  react-native-reanimated react-native-worklets
```

Expo's documented `babel-preset-expo` setup already configures the worklets
plugin; do not add a duplicate plugin. Bare React Native CLI projects add
`react-native-worklets/plugin` as the **last plugin**, use
`module:@react-native/babel-preset`, and install pods / sync Gradle normally.
Do not add `react-native-reanimated/plugin` alongside the worklets plugin.

The current runtime baseline is Expo **58.0.25**, React Native **0.88.1**,
React **19.3.0**, Skia **2.14.0**, Reanimated **4.7.x**, Worklets **0.13.x**.
Native Debug checks exercised all nine transitions in both directions on
iOS Simulator and Android Emulator. The Skia 2.4.18 / 2.6.2 / 2.14.0 CI
matrix checks types and builds; it does not prove device compatibility for
every combination. Earlier checks used Expo Go. No package-specific native
module or prebuild is required for Expo Go with matching native peers.

Do not infer an SDK 55 minimum from missing Skia 2 / Reanimated 4: SDK 54
already bundles them. Older runtime support needs its own validation; the
declared peer minimums below are not a tested compatibility matrix. See
[getting started](references/getting-started.md) before version-specific advice.

The [public Snack](https://snack.expo.dev/@mariops03/react-native-theme-transition)
currently offers SDK 55 at most. It needs the compatible SDK 55 Expo Go
client; the SDK 58 client cannot open it. Its web preview shows instructions
because the transition engine runs on native platforms.

## Peer Dependencies

| Peer | Minimum | Why |
|---|---|---|
| `react` | `>=19.0.0` | The hook reads context via React 19's `use()` |
| `react-native` | `>=0.78.0` | Declared library range; check native peer compatibility |
| `@shopify/react-native-skia` | `>=2.0.0` | Capture (`makeImageFromView`), overlay Canvas, shaders |
| `react-native-reanimated` | `>=4.0.0` | `progress` shared value, `withTiming` |
| `react-native-worklets` | `>=0.5.0` | Transitive requirement of Reanimated 4 |

`react-native-view-shot` is no longer a peer dependency (removed from v1).

## Key Rules

1. **Provider wraps everything.** Place `ThemeTransitionProvider` above navigation and above modals. The Skia snapshot only captures what is inside the provider's root `View`.
2. **All themes share the same token keys.** Mismatches throw at init. Type it with `Record<keyof typeof light, string>` on secondary themes.
3. **`'system'` is reserved.** It cannot be used as a theme name; it is a first-class value for `initialTheme` and `setTheme`.
4. **`initialTheme` is read once.** Changing the prop later does not do anything; call `setTheme` instead.
5. **`setTheme` returns `'accepted' | 'ignored'`.** `'ignored'` means a transition is in flight, or both the resolved target and system/manual mode are unchanged. Most callers discard the return value and rely on reactive state.
6. **`theme.name` is always concrete.** It can never be `'system'`. Use `preference` to detect system-following mode.
7. **Animated `preference` changes land with the colour-swap commit.** The old snapshot retains the previous picker selection; the new highlight appears as the overlay reveals the new theme. Pickers can still read `preference` directly. Instant switches and mode-only changes that keep the painted theme write it immediately.
8. **Provide `systemThemeMap` with custom theme names.** If your themes are not named `'light'`/`'dark'` and you want `initialTheme="system"` or `setTheme('system')` to work.
9. **`darkThemes` controls `Appearance.setColorScheme` for native UI elements.** Only needed with two or more dark themes. Derived from `systemThemeMap` otherwise.
10. **Never call `Appearance.setColorScheme` yourself.** The library manages it.
11. **`isTransitioning` covers the colour-swap commit through JS cleanup.** It excludes initial capture/setup. The engine blocks touches and rejects concurrent calls earlier; use the flag for visual disabling without assuming it marks capture start or actual display presentation.
12. **Do not use `react-native`'s built-in `<Switch>` for theme toggling.** UISwitch runs its own ~250ms Core Animation without a completion callback, so the snapshot catches the thumb mid-slide and the result flickers. Build a custom toggle with plain styles.
13. **Use `theme.scheme` for binary light/dark UIs.** It is a derived `'light' | 'dark'` flag that stays correct even when you have custom themes like `'ocean'` or `'midnight'`.
14. **Use the exported `ViewRef` for origins.** Import `type ViewRef` from the library and use `useRef<ViewRef>(null)` with a native `View`; RN 0.88 removed `View`'s instance-type use. See [types](references/types.md#originspec).
15. **Pass `colorScheme={theme.scheme}` to `expo-glass-effect`'s `GlassView`.** Its default `'auto'` follows native appearance, which the engine updates after the animation. This improves material colour alignment; the snapshot still freezes live glass. See [Liquid Glass](references/recipes/glass-effect.md).

## The Hook Shape

```ts
const { theme, preference, setTheme, isTransitioning } = useTheme()

// theme.name:    Names          (always concrete, never 'system')
// theme.colors:  Record<Tokens, string>
// theme.scheme:  'light' | 'dark'
// preference:    Names | 'system'  (the user's raw pick)
// setTheme:      (name | 'system', options?) => 'accepted' | 'ignored'
// isTransitioning: boolean
```

Two orthogonal concepts: `theme` is what is currently painted on screen (always
resolved, always concrete), and `preference` is what the user explicitly picked
(may be `'system'`). Hook return shape is exactly these four fields; nothing
more, nothing less.

## The 9 Transitions

| Transition | Kind | Invertible | Needs origin | Default duration |
|---|---|---|---|---|
| `fade` | fade | no | no | 350ms |
| `circularReveal` | reveal | yes | yes | 350ms |
| `wipe` | strip | no | no | 350ms |
| `slide` | strip | no | no | 350ms |
| `split` | strip | yes | no | 350ms |
| `heart` | shape | yes | yes | 800ms |
| `star` | shape | yes | yes | 800ms |
| `pixelize` | shader | no | no | 750ms |
| `dissolve` | shader | no | no | 750ms |

Pass a default in config (`createThemeTransition({ themes, transition: 'heart' })`)
or override per call (`setTheme('dark', { transition: 'wipe', direction: 'right' })`).
`SetThemeOptions` is a discriminated union — TypeScript only accepts the extra
fields valid for the chosen `transition`.

Runtime metadata is exported:

```ts
import { TRANSITION_META, TRANSITION_TYPES } from 'react-native-theme-transition'

TRANSITION_TYPES // readonly ['fade', 'circularReveal', 'wipe', 'slide', 'split', 'heart', 'star', 'pixelize', 'dissolve']
TRANSITION_META[transition] // { kind, needsOrigin, invertible, capturesNew, defaultDuration }
```

## How It Works (30-second version)

1. `setTheme('dark')` blocks touches and concurrent calls. A manual animated pick is stashed in a ref, avoiding a preference-driven context update before capture. System-mode changes and consumer callbacks may still enqueue React updates.
2. Capture the inner tree via `makeImageFromView` without a pre-capture frame wait.
3. **Commit 1**: the snapshot is mounted on the permanently-mounted Skia Canvas.
4. Wait one frame (`SETTLE.overlayCommit`), probe the Canvas with `makeImageSnapshotAsync`, then allow one compositing frame (`SETTLE.skiaPaintSynced`). If the probe is unavailable, use the platform fallback frame budget (`SETTLE.skiaPaint`, 2/3 frames).
5. **Commit 2**: swap the colour tokens and stashed preference underneath the overlay; `isTransitioning` becomes true.
6. For reveal/shape/dual-image kinds, wait one more frame (`SETTLE.treeRepaint`). For `slide` and `pixelize`, capture a second snapshot of the inner tree in the new theme.
7. Reanimated animates `progress` 0 → 1 on the UI thread. A separate JS `setTimeout(duration)` resets the overlay and fires cleanup callbacks; it is not a UI animation completion acknowledgement.
8. Capture failure falls back to an instant swap. Start callbacks have already fired; end callbacks are skipped and `onThemeChange` fires.

The paint probe renders the current picture offscreen. It acknowledges neither
a particular React commit nor on-screen presentation. The frame margins are a
workaround, not a universal flash-free or physical 120 Hz guarantee.
Reanimated's `withTiming` inherits `ReduceMotion.System`; to skip capture/setup
and the independent cleanup delay too, read the system setting in the app and
pass `{ animated: false }`.

## Callback Order (animated path)

```
1. setTheme() blocks touches/concurrent calls and stashes the animated pick
2. configOnTransitionStart(name)
3. options.onTransitionStart(name)
4. captureView (old), with no pre-capture wait
5. Commit 1 (mount snapshot)
6. overlayCommit frame + offscreen paint probe + composite margin (or fallback)
7. Commit 2 (colour tokens + preference, isTransitioning → true)
8. [optional] treeRepaint settle + captureView (new)
9. progress 0 → 1 animation
10. independent JS cleanup timer; configOnTransitionEnd(name)
11. options.onTransitionEnd(name)
12. configOnThemeChange(name)
13. [system mode only] configOnThemeChange(correctedName) if OS appearance changed mid-transition
```

`onThemeChange` fires after applied animated/instant changes and capture fallback,
not for ignored calls, mode-only no-ops, start-callback exceptions, or provider
unmounts. In `'system'` mode it can fire again for an OS correction. None of the
callbacks confirms display presentation. See [callbacks](references/guides/callbacks.md).

## Reference Index

Each file below is a focused topic. Load the ones relevant to the task
instead of the whole tree.

### Getting Started
| File | What's in it |
|---|---|
| [references/overview.md](references/overview.md) | What the library is, highlights, comparison table, trade-offs, requirements |
| [references/getting-started.md](references/getting-started.md) | Expo / RN CLI installation, Babel, tested SDK 58 baseline versus declared peer ranges, New Architecture |
| [references/quick-start.md](references/quick-start.md) | Three-step setup: define themes → wrap app → consume hook |
| [references/types.md](references/types.md) | Every exported TypeScript type with field-level reference, inference patterns, `satisfies` vs `:` footgun |

### API Reference
| File | What's in it |
|---|---|
| [references/api/create-theme-transition.md](references/api/create-theme-transition.md) | Factory config: `themes`, `animated`, `transition`, `systemThemeMap`, `darkThemes`, all three callbacks. Common configurations (minimal/standard/custom names/multiple darks). Init-time validation errors. |
| [references/api/provider.md](references/api/provider.md) | `ThemeTransitionProvider` props, `initialTheme` behavior, placement rules, modal/bottom sheet caveats |
| [references/api/use-theme.md](references/api/use-theme.md) | Hook return shape, `theme` vs `preference` decision table + common mistakes (Do/Don't), `setTheme` reference, runtime validation errors, behavior rules |

### Guides
| File | What's in it |
|---|---|
| [references/guides/how-it-works.md](references/guides/how-it-works.md) | Two-commit flow, offscreen paint probe and frame margins, deferred preference, JS cleanup, Reduce Motion and presentation limits |
| [references/guides/callbacks.md](references/guides/callbacks.md) | When each callback fires (animated/instant/system-driven/capture-failure), ordering, full summary matrix |
| [references/guides/troubleshooting.md](references/guides/troubleshooting.md) | 14 symptom-cause-fix entries (picker doesn't highlight system, native Switch flickers, stuck overlay, iOS safe-area, Android ScrollView strip, Fast Refresh, TS errors, double transition on start, iOS 26 Liquid Glass, etc.) + complete error messages reference |

### Recipes
| File | What's in it |
|---|---|
| [references/recipes/expo-router.md](references/recipes/expo-router.md) | Canonical 3-provider stack with `@react-navigation/native` `ThemeProvider` bridge, safe-area fix, bottom sheets, modal caveat |
| [references/recipes/glass-effect.md](references/recipes/glass-effect.md) | Themed GlassView wrapper, Expo Go availability, explicit colorScheme and static snapshot limitations |
| [references/recipes/persistence.md](references/recipes/persistence.md) | AsyncStorage-only: read on boot, pass as `initialTheme`, write in onPress; avoid double transition on startup |
| [references/recipes/zustand.md](references/recipes/zustand.md) | State manager bridge: Zustand (with persist), Redux, MMKV. Hydration-only pattern, not reactive |
| [references/recipes/migration.md](references/recipes/migration.md) | v1 → v2 upgrade: peer deps, hook shape, `setTheme` return, `select()` removal, reducedMotion, `config.duration`/`backgroundColor` removal, renamed fields. Plus migrations from plain Context, React Navigation, MMKV |
| [references/recipes/haptic-feedback.md](references/recipes/haptic-feedback.md) | `onTransitionStart` for haptics (config-level and per-call), `onThemeChange` for analytics, defer expensive renders |
| [references/recipes/multiple-scopes.md](references/recipes/multiple-scopes.md) | Multiple isolated transition systems in the same app (per-tenant dashboards, kid/parent zones, onboarding). Caveats around OS color scheme and snapshot isolation |
| [references/recipes/testing.md](references/recipes/testing.md) | Jest mocks for Skia, Reanimated, Worklets. Testing themed components, `setTheme` behavior, caveats |

### Examples
| File | What's in it |
|---|---|
| [references/examples/theme-toggle.md](references/examples/theme-toggle.md) | Custom switch (not `react-native`'s `<Switch>`), animated thumb, `theme.name`-driven |
| [references/examples/theme-picker.md](references/examples/theme-picker.md) | Segmented control with `preference` highlight and `'system'` row |
| [references/examples/theme-button.md](references/examples/theme-button.md) | Cycle-on-tap button |
| [references/examples/system-theme.md](references/examples/system-theme.md) | Follow OS appearance — setup, foreground vs background switches, gotchas |
| [references/examples/react-navigation.md](references/examples/react-navigation.md) | Non-Expo Router setup with `NavigationContainer` theme bridge |
| [references/examples/checkmark-list.md](references/examples/checkmark-list.md) | iOS Settings-style checkmark list driven by `preference` |
