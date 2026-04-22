# System Theme

The library reads the OS color scheme synchronously on mount and
subscribes to changes. Pass `initialTheme="system"` and the very
first frame is already on the right theme. No flash, no async
hydration, no race.

## Basic Setup

```tsx
<ThemeTransitionProvider initialTheme="system">
  <App />
</ThemeTransitionProvider>
```

With custom theme names, add `systemThemeMap` so the library knows
which of your themes corresponds to the OS `light` / `dark`:

```ts
export const { ThemeTransitionProvider, useTheme } = createThemeTransition({
  themes: { sunrise, midnight },
  systemThemeMap: { light: 'sunrise', dark: 'midnight' },
})
```

## Detecting System Mode

`theme.name` is always the concrete resolved theme. It is never
`'system'`. To detect whether the user chose to follow the OS, read
`preference`:

```tsx
const { preference } = useTheme()
const isFollowingSystem = preference === 'system'
```

`theme` describes what's painted; `preference` describes what the
user picked. See [useTheme](../api/use-theme.md#theme-vs-preference)
for the full decision table.

```tsx
const { theme, preference } = useTheme()

// For styling: read from theme.colors
<View style={{ backgroundColor: theme.colors.background }} />

// For picker highlights: compare preference so 'system' can win
const isSystemSelected = preference === 'system'
```

## Entering and Exiting System Mode

```tsx
setTheme('system')  // enter: subscribes to OS appearance changes
setTheme('dark')    // exit: manual mode, OS changes are ignored
```

## What Happens Under the Hood

- **Foreground.** An OS appearance change runs a full animated transition.
- **Background.** An OS appearance change is applied as an instant switch; nothing to animate.
- **Returning to foreground.** The library re-reads the OS scheme and corrects instantly if it changed while the app was backgrounded.

## StatusBar Sync

The library doesn't touch the status bar. Sync it yourself using
`theme.scheme`. It's always `'light'` or `'dark'`, regardless of
how many themes you define or whether the user is in system mode.

```tsx
import { StatusBar } from 'expo-status-bar'
import { useTheme } from '@/lib/theme'

function StatusBarSync() {
  const { theme } = useTheme()
  return <StatusBar style={theme.scheme === 'dark' ? 'light' : 'dark'} />
}
```

## Native UI Elements

The library calls `Appearance.setColorScheme` internally so alerts,
date pickers, and keyboards match the active theme. In system mode
it uses `'unspecified'` so the OS drives them. With multiple dark
themes, tell the library which ones count as dark:

```ts
createThemeTransition({
  themes: { light, dark, ocean },
  darkThemes: ['dark', 'ocean'],
})
```

> **Warning.** Do **not** call `Appearance.setColorScheme` yourself. The library
> tracks system mode through that call; manual writes desync it and
> break the next `setTheme('system')` on Android.

## See Also

- [Theme Picker](../examples/theme-picker.md). Picker that includes a `'system'` row.
- [Persisted Preference](../recipes/persistence.md). Restore the user's pick on launch.
- [createThemeTransition](../api/create-theme-transition.md#systemthememap). The `systemThemeMap` field.
- [Callbacks](../guides/callbacks.md). System-driven changes go through `onThemeChange`.
