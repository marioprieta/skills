# Quick Start

This walkthrough goes from an empty Expo project to a working
animated theme switch in three files. It assumes you already
[installed the package](./getting-started.md). The same setup
applies to React Native CLI projects; only the file paths change.

## Setup

### Step 1
#### Define Your Themes

Create a single file that defines your themes and exports the
library's API. TypeScript infers theme names and color tokens
automatically. No manual generics.

```ts title="lib/theme.ts"
import { createThemeTransition } from 'react-native-theme-transition'

const light = {
  background:   '#ffffff',
  card:         '#f5f5f5',
  text:         '#000000',
  border:       '#e0e0e0',
  primary:      '#007AFF',
  notification: '#ff3b30',
}

const dark: Record<keyof typeof light, string> = {
  background:   '#000000',
  card:         '#1c1c1e',
  text:         '#ffffff',
  border:       '#333333',
  primary:      '#0A84FF',
  notification: '#ff453a',
}

export const { ThemeTransitionProvider, useTheme } = createThemeTransition({
  themes: { light, dark },
})
```

TypeScript now knows your theme names (`'light' | 'dark'`) and
your token keys (`'background' | 'card' | 'text' | …`). Both
propagate to `useTheme` and `setTheme` automatically.

### Step 2
#### Wrap Your App

Place `ThemeTransitionProvider` as high as possible: above your
navigation container, above any modals. The snapshot capture
includes everything inside the provider; anything outside it
won't appear in the overlay and may flash. See the
[placement rules](./api/provider.md#placement) for nesting
examples and modal caveats.

```tsx title="app/_layout.tsx"
import { ThemeTransitionProvider } from '@/lib/theme'

export default function RootLayout() {
  return (
        <ThemeTransitionProvider initialTheme="system">
          <App />
        </ThemeTransitionProvider>
  )
}
```

`initialTheme="system"` reads the OS appearance synchronously on
launch and subscribes to OS changes. See
[System Theme](./examples/system-theme.md) for details.

To remember the user's choice across launches (instead of always
starting in system mode), see the
[Persistence recipe](./recipes/persistence.md). It reads the
stored value once at boot and passes it as `initialTheme`.

### Step 3
#### Use in Components

Call `useTheme()` to read the painted theme, the user's
preference, the mutator, and the transition flag.

```tsx title="app/index.tsx"
import { Pressable, Text, View } from 'react-native'
import { useTheme } from '@/lib/theme'

export default function Home() {
  const { theme, preference, setTheme, isTransitioning } = useTheme()

  return (
        <View style={{ flex: 1, backgroundColor: theme.colors.background }}>
          <Text style={{ color: theme.colors.text }}>
            Current: {theme.name}
          </Text>
          <Pressable
            disabled={isTransitioning}
            onPress={() => setTheme(theme.name === 'dark' ? 'light' : 'dark')}
          >
            <Text style={{ color: theme.colors.primary }}>Toggle theme</Text>
          </Pressable>
        </View>
  )
}
```

`useTheme()` returns four fields:

- **`theme`**. What's painted right now. `theme.name` is always
  a concrete theme (never `'system'`), `theme.colors` holds the
  resolved tokens, `theme.scheme` is `'light'` or `'dark'`. Use
  it for styling, toggles, and asset lookups. Read `theme.scheme`
  whenever you want a binary dark/light flag (for example, to
  bridge into React Navigation's `dark` prop) — it stays correct
  even if you grow beyond two themes.
- **`preference`**. The theme the user explicitly picked. Can
  be `'system'`. Use it for picker highlights and persistence.
- **`setTheme`**. Change the preference. Accepts a concrete
  name or `'system'`.
- **`isTransitioning`**. `true` while the overlay animation is
  running. Use it to disable interactive controls.

See [useTheme](./api/use-theme.md#theme-vs-preference) for the
full decision table.

## See Also

- [createThemeTransition](./api/create-theme-transition.md). Every config option.
- [useTheme](./api/use-theme.md). The hook and `setTheme` reference.
- [Theme Picker](./examples/theme-picker.md). Segmented control with `'system'` highlight.
- [System Theme](./examples/system-theme.md). Follow OS appearance.
- [Expo Router integration](./recipes/expo-router.md). Typed navigation colors.
- [How transitions work](./guides/how-it-works.md). The capture, swap, animate model.
