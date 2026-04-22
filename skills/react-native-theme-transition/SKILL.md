---
name: react-native-theme-transition
description: "Animated dark mode and theme transitions for React Native, v2 Skia engine with 9 transition styles. TRIGGER when: code imports react-native-theme-transition, user works with createThemeTransition/ThemeTransitionProvider/useTheme, implements animated dark mode toggling, system theme following, theme persistence, React Navigation or Expo Router theme integration, picks between fade/circularReveal/heart/star/wipe/slide/split/pixelize/dissolve, or debugs transition issues (stuck overlay, flash, capture failure). DO NOT TRIGGER when: general React Native styling, non-animated theme switching, or web-only theming."
license: MIT
compatibility: "react >= 19.0.0, react-native >= 0.78.0, react-native-reanimated >= 4.0.0, @shopify/react-native-skia >= 2.0.0, react-native-worklets >= 0.5.0. Works in Expo Go on SDK 54+ (Skia and Reanimated are bundled). Bare React Native CLI also supported."
metadata:
  author: marioprieta
  version: "2.0.0"
  tags: react-native, theme, dark-mode, animation, transition, expo, reanimated, skia, view-transition
---

# react-native-theme-transition (v2)

Animated theme transitions for React Native, powered by `@shopify/react-native-skia`
and Reanimated. The engine snapshots the inner view tree into a Skia `SkImage`,
mounts it on a permanently-mounted Skia Canvas, swaps the theme tokens
underneath, and animates the snapshot away with one of nine styles.

## Installation

Expo (SDK 54+):

```bash
npx expo install react-native-theme-transition @shopify/react-native-skia \
  react-native-reanimated react-native-worklets
```

Add `react-native-worklets/plugin` as the **last plugin** in `babel.config.js`.
On Expo SDK 55+, do NOT add `react-native-reanimated/plugin` — `babel-preset-expo`
already includes it.

v2 works in Expo Go on SDK 54+ because Skia and Reanimated are now bundled
into the Go client. No `expo prebuild` is required for managed Expo apps.
Bare React Native CLI projects install the native modules normally via
`pod install` (iOS) and the Gradle sync (Android).

## Peer Dependencies

| Peer | Minimum | Why |
|---|---|---|
| `react` | `>=19.0.0` | The hook reads context via React 19's `use()` |
| `react-native` | `>=0.78.0` | Required by Skia 2.0 |
| `@shopify/react-native-skia` | `>=2.0.0` | Capture (`makeImageFromView`), overlay Canvas, shaders |
| `react-native-reanimated` | `>=4.0.0` | `progress` shared value, `withTiming` |
| `react-native-worklets` | `>=0.5.0` | Transitive requirement of Reanimated 4 |

`react-native-view-shot` is no longer a peer dependency (removed from v1).

## Key Rules

1. **Provider wraps everything.** Place `ThemeTransitionProvider` above navigation and above modals. The Skia snapshot only captures what is inside the provider's root `View`.
2. **All themes share the same token keys.** Mismatches throw at init. Type it with `Record<keyof typeof light, string>` on secondary themes.
3. **`'system'` is reserved.** It cannot be used as a theme name; it is a first-class value for `initialTheme` and `setTheme`.
4. **`initialTheme` is read once.** Changing the prop later does not do anything; call `setTheme` instead.
5. **`setTheme` returns `'accepted' | 'ignored'`.** `'ignored'` means the engine did nothing (same preference or mid-transition). Most callers discard the return value and rely on reactive state.
6. **`theme.name` is always concrete.** It can never be `'system'`. Use `preference` to detect system-following mode.
7. **`preference` updates synchronously inside `setTheme`.** Pickers use `preference` directly for highlight state; no local optimistic state and no `useEffect` needed.
8. **Provide `systemThemeMap` with custom theme names.** If your themes are not named `'light'`/`'dark'` and you want `initialTheme="system"` or `setTheme('system')` to work.
9. **`darkThemes` controls `Appearance.setColorScheme` for native UI elements.** Only needed with two or more dark themes. Derived from `systemThemeMap` otherwise.
10. **Never call `Appearance.setColorScheme` yourself.** The library manages it.
11. **Disable interactive elements with `isTransitioning`.** Do not restyle them mid-transition; the snapshot captures current visuals and a style change between tap and animation end produces a visible flash.
12. **Do not use `react-native`'s built-in `<Switch>` for theme toggling.** UISwitch runs its own ~250ms Core Animation without a completion callback, so the snapshot catches the thumb mid-slide and the result flickers. Build a custom toggle with plain styles.
13. **Use `theme.scheme` for binary light/dark UIs.** It is a derived `'light' | 'dark'` flag that stays correct even when you have custom themes like `'ocean'` or `'midnight'`.

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

1. `setTheme('dark')` is called. `transitioningRef`, `preference`, and the touch-block shared value all write synchronously so picker UIs repaint before capture.
2. `runTransition` waits one frame (`SETTLE.beforeCapture`) so React flushes those sync writes to the screen, then calls `captureView` → `makeImageFromView` to grab a CPU-backed `SkImage` of the inner tree.
3. **Commit 1**: the snapshot is mounted on the permanently-mounted Skia Canvas.
4. Wait for Skia paint (`SETTLE.skiaPaint`, 2 frames) so the Canvas shows the snapshot before the inner tree swaps colors.
5. **Commit 2**: the inner tree swaps to the new theme underneath the now-covering overlay.
6. For reveal/shape/dual-image kinds, wait one more frame (`SETTLE.treeRepaint`). For `slide` and `pixelize`, capture a second snapshot of the inner tree in the new theme.
7. Reanimated animates `progress` 0 → 1 over the transition's duration. When the animation finishes, reset the overlay and fire callbacks.
8. Capture failure at step 2 → instant swap fallback. `onTransitionStart` and `onTransitionEnd` are skipped; only `onThemeChange` fires.

## Callback Order (animated path)

```
1. setTheme() sync writes (transitioningRef, preference, isBlocking)
2. configOnTransitionStart(name)
3. options.onTransitionStart(name)
4. beforeCapture settle (1 frame)
5. captureView (old)
6. Commit 1 + skiaPaint settle
7. Commit 2 (inner tree swap)
8. [optional] treeRepaint settle + captureView (new)
9. progress 0 → 1 animation
10. configOnTransitionEnd(name)
11. options.onTransitionEnd(name)
12. configOnThemeChange(name)
13. [system mode only] configOnThemeChange(correctedName) if OS appearance changed mid-transition
```

`onThemeChange` is the only callback guaranteed to fire on every code path
(including capture failure). In `'system'` mode it can fire twice in a single
`setTheme` call if the OS appearance changes during the animation.

## Reference Index

Each file below is a focused topic. Load the ones relevant to the task
instead of the whole tree.

### Getting Started
| File | What's in it |
|---|---|
| [references/overview.md](references/overview.md) | What the library is, highlights, comparison table, trade-offs, requirements |
| [references/getting-started.md](references/getting-started.md) | Install + peer deps (Expo SDK 54+, older Expo, RN CLI), babel config, New Architecture check |
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
| [references/guides/how-it-works.md](references/guides/how-it-works.md) | Two-commit snapshot/swap/animate model, SETTLE pattern (beforeCapture, skiaPaint, treeRepaint), why snapshots, why no Reduce Motion |
| [references/guides/callbacks.md](references/guides/callbacks.md) | When each callback fires (animated/instant/system-driven/capture-failure), ordering, full summary matrix |
| [references/guides/troubleshooting.md](references/guides/troubleshooting.md) | 14 symptom-cause-fix entries (picker doesn't highlight system, native Switch flickers, stuck overlay, iOS safe-area, Android ScrollView strip, Fast Refresh, TS errors, double transition on start, iOS 26 Liquid Glass, etc.) + complete error messages reference |

### Recipes
| File | What's in it |
|---|---|
| [references/recipes/expo-router.md](references/recipes/expo-router.md) | Canonical 3-provider stack with `@react-navigation/native` `ThemeProvider` bridge, safe-area fix, bottom sheets, modal caveat |
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
