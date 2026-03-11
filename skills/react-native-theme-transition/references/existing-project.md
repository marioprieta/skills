# Adding to an Existing Project

Migration guides for different state management approaches.

## Table of contents

1. [General strategy](#general-strategy)
2. [From plain Context / useState](#from-plain-context--usestate)
3. [From Zustand](#from-zustand)
4. [From Redux / Redux Toolkit](#from-redux--redux-toolkit)
5. [From MMKV direct](#from-mmkv-direct)
6. [From React Navigation theming only](#from-react-navigation-theming-only)
7. [Preserving an existing useTheme hook](#preserving-an-existing-usetheme-hook)

---

## General strategy

The migration follows the same pattern regardless of your current approach:

1. **Install** the library and peer dependencies
2. **Create the theme file** — define tokens, call `createThemeTransition`
3. **Wrap with provider** — add `ThemeTransitionProvider` at the root
4. **Bridge** your existing state to the provider via a bridge component
5. **Replace** color references from your old system to `useTheme().colors`
6. **Remove** the old theme context/provider once fully migrated

The bridge pattern is the key concept. The library owns the animated transition
state internally. Your external state manager (Zustand, Redux, etc.) drives it
via a **hydration-only** bridge component that syncs the persisted preference on
app start. After hydration, the picker calls `select()` + store setter directly
in `onPress` — not through the bridge.

> **Important:** Don't use a reactive bridge (`useEffect → setTheme` on every
> store change) if your picker uses `select()`. It races with `select()`'s
> deferred timing and reverts the pill highlight.

### The bridge pattern (hydration-only)

```tsx
function ThemeBridge() {
  const { setTheme } = useTheme();

  useEffect(() => {
    const sync = () => setTheme(/* read from your state manager */);
    // Sync once on hydration. After this, the picker drives transitions directly.
    sync();
  }, [setTheme]);

  return null;
}
```

Place this inside the `ThemeTransitionProvider`, before any other children.

---

## From plain Context / useState

### Before

```tsx
// ThemeContext.tsx
const ThemeContext = createContext({ theme: 'light', colors: lightColors, toggle: () => {} });

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  const colors = theme === 'light' ? lightColors : darkColors;
  return (
    <ThemeContext.Provider value={{ theme, colors, toggle: () => setTheme(t => t === 'light' ? 'dark' : 'light') }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

### After

This is the simplest migration — you can fully replace the old context.

**Step 1:** Create the theme file:

```ts
// src/lib/theme.ts
import { createThemeTransition } from 'react-native-theme-transition';

// Reuse your existing color objects
export const { ThemeTransitionProvider, useTheme } = createThemeTransition({
  themes: { light: lightColors, dark: darkColors },
});
```

**Step 2:** Replace the old provider in your root layout:

```tsx
// Before:
<ThemeProvider>
  <App />
</ThemeProvider>

// After:
<ThemeTransitionProvider initialTheme="light">
  <App />
</ThemeTransitionProvider>
```

**Step 3:** Update components — replace `useContext(ThemeContext)` with `useTheme()`:

```tsx
// Before:
const { colors, toggle } = useContext(ThemeContext);

// After:
const { colors, name, setTheme } = useTheme();
const toggle = () => setTheme(name === 'light' ? 'dark' : 'light');
```

**Step 4:** Delete the old `ThemeContext.tsx`.

---

## From Zustand

Keep Zustand as the source of truth for persisted preference. Add a **hydration-only**
bridge that syncs once on app start, then let the picker drive transitions directly.

### Theme store (keep as-is or simplify)

```ts
// src/stores/theme-store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

export type ColorMode = 'system' | 'light' | 'dark';

interface ThemeState {
  colorMode: ColorMode;
  setColorMode: (mode: ColorMode) => void;
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      colorMode: 'system',
      setColorMode: (mode) => set({ colorMode: mode }),
    }),
    {
      name: 'theme-store',
      storage: createJSONStorage(() => AsyncStorage),
    },
  ),
);
```

### Theme file

```ts
// src/lib/theme.ts
import { createThemeTransition } from 'react-native-theme-transition';

const light = { /* your tokens */ };
const dark = { /* your tokens */ };

export const { ThemeTransitionProvider, useTheme } = createThemeTransition({
  themes: { light, dark },
});
```

### Bridge component (hydration-only)

```tsx
// In your root layout file
import { useThemeStore } from '@/stores/theme-store';
import { useTheme } from '@/lib/theme';

/** Syncs persisted Zustand preference on hydration — not on every change. */
function ThemeBridge() {
  const { setTheme } = useTheme();

  useEffect(() => {
    // getState() reads synchronously — no subscription needed. Runs once on hydration.
    const sync = () => setTheme(useThemeStore.getState().colorMode);
    if (useThemeStore.persist.hasHydrated()) {
      sync();
    }
    return useThemeStore.persist.onFinishHydration(sync);
  }, [setTheme]);

  return null;
}
```

### Root layout

```tsx
export default function RootLayout() {
  const colorMode = useThemeStore((s) => s.colorMode);

  return (
    <ThemeTransitionProvider initialTheme={colorMode}>
      <ThemeBridge />
      {/* rest of app */}
    </ThemeTransitionProvider>
  );
}
```

Note: `initialTheme={colorMode}` reads the Zustand value for the first frame.
If Zustand hydrates asynchronously, the initial value is the default ('system').
The bridge syncs once hydration completes.

### Settings screen

Call `select()` + store setter together in `onPress`:

```tsx
import { useThemeStore } from '@/stores/theme-store';
import { useTheme } from '@/lib/theme';

function ThemeSettings() {
  const colorMode = useThemeStore((s) => s.colorMode);
  const setColorMode = useThemeStore((s) => s.setColorMode);
  const { colors, isTransitioning, selected, select } = useTheme({
    initialSelection: colorMode,
  });

  return (
    <>
      {(['light', 'dark', 'system'] as const).map((mode) => (
        <Button
          key={mode}
          title={mode}
          onPress={() => {
            select(mode);       // visual transition
            setColorMode(mode); // persistence
          }}
          disabled={isTransitioning}
        />
      ))}
    </>
  );
}
```

---

## From Redux / Redux Toolkit

Same hydration-only bridge pattern, different selector.

### Slice (keep as-is)

```ts
// src/store/themeSlice.ts
const themeSlice = createSlice({
  name: 'theme',
  initialState: { colorMode: 'system' as 'system' | 'light' | 'dark' },
  reducers: {
    setColorMode: (state, action) => { state.colorMode = action.payload; },
  },
});
```

### Bridge (hydration-only)

```tsx
function ThemeBridge() {
  const store = useStore<RootState>();
  const { setTheme } = useTheme();
  const didSync = useRef(false);

  useEffect(() => {
    if (didSync.current) return;
    didSync.current = true;
    setTheme(store.getState().theme.colorMode);
  }, [store, setTheme]);

  return null;
}
```

`useStore` is from `react-redux`. `RootState` is your Redux root state type — see
[Redux Toolkit docs](https://redux-toolkit.js.org/tutorials/typescript#define-typed-hooks).

### Root — provider order

```tsx
<Provider store={store}>
  <ThemeTransitionProvider initialTheme={store.getState().theme.colorMode}>
    <ThemeBridge />
    <App />
  </ThemeTransitionProvider>
</Provider>
```

Redux Provider must wrap `ThemeTransitionProvider` so the bridge can read the store.
Settings screen: call `select(mode)` + `dispatch(setColorMode(mode))` together in `onPress`.

---

## From MMKV direct

> **Note:** MMKV requires native modules and is not compatible with Expo Go.
> Use a development build or the bare workflow.

MMKV reads are synchronous, so no bridge is needed — pass the stored value
directly as `initialTheme`:

```ts
// src/hooks/useStoredTheme.ts
import { useMMKVString } from 'react-native-mmkv';

export function useStoredTheme() {
  const [value, setValue] = useMMKVString('themePreference');
  const colorMode = (value ?? 'system') as 'system' | 'light' | 'dark';
  return { colorMode, setColorMode: setValue };
}
```

### Root layout

```tsx
export default function RootLayout() {
  const { colorMode } = useStoredTheme();

  return (
    <ThemeTransitionProvider initialTheme={colorMode}>
      <App />
    </ThemeTransitionProvider>
  );
}
```

### Settings screen

```tsx
import { useStoredTheme } from '@/hooks/useStoredTheme';
import { useTheme } from '@/lib/theme';

const OPTIONS = ['system', 'light', 'dark'] as const;

function ThemeSettings() {
  const { colorMode, setColorMode } = useStoredTheme();
  const { colors, isTransitioning, selected, select } = useTheme({
    initialSelection: colorMode,
  });

  return (
    <View style={{ flexDirection: 'row', gap: 8 }}>
      {OPTIONS.map((mode) => (
        <Pressable
          key={mode}
          onPress={() => {
            select(mode);        // visual transition
            setColorMode(mode);  // persistence
          }}
          disabled={isTransitioning}
          style={{
            flex: 1,
            padding: 12,
            borderRadius: 8,
            alignItems: 'center',
            backgroundColor:
              mode === selected ? colors.primary : 'transparent',
          }}
        >
          <Text style={{ color: mode === selected ? '#fff' : colors.text }}>
            {mode}
          </Text>
        </Pressable>
      ))}
    </View>
  );
}
```

---

## From React Navigation theming only

If your only theme system is `NavigationContainer`'s `theme` prop with
`DefaultTheme` / `DarkTheme`, the migration adds the animation layer on top.

### Before

```tsx
<NavigationContainer theme={isDark ? DarkTheme : DefaultTheme}>
```

### After

**Step 1:** Define themes matching Navigation's token structure:

```ts
// src/lib/theme.ts
import { createThemeTransition } from 'react-native-theme-transition';

const light = {
  primary: '#007AFF',
  background: '#ffffff',
  card: '#ffffff',
  text: '#000000',
  border: '#d8d8d8',
  notification: '#ff3b30',
};

const dark: Record<keyof typeof light, string> = {
  primary: '#0A84FF',
  background: '#000000',
  card: '#1c1c1e',
  text: '#ffffff',
  border: '#333333',
  notification: '#ff453a',
};

export const { ThemeTransitionProvider, useTheme } = createThemeTransition({
  themes: { light, dark },
});
```

**Step 2:** Bridge to NavigationContainer:

```tsx
function AppWithNavigation() {
  const { colors, name } = useTheme();

  const navTheme = {
    dark: name === 'dark',
    colors: {
      primary: colors.primary,
      background: colors.background,
      card: colors.card,
      text: colors.text,
      border: colors.border,
      notification: colors.notification,
    },
  };

  return (
    <NavigationContainer theme={navTheme}>
      <AppNavigator />
    </NavigationContainer>
  );
}

// Root:
<ThemeTransitionProvider initialTheme="system">
  <AppWithNavigation />
</ThemeTransitionProvider>
```

---

## Preserving an existing useTheme hook

If your codebase already has a `useTheme` hook with a different return shape and you
can't rename all call sites at once, re-export with an adapter:

```ts
// src/lib/theme.ts
import { createThemeTransition } from 'react-native-theme-transition';

const api = createThemeTransition({ themes: { light, dark } });

export const ThemeTransitionProvider = api.ThemeTransitionProvider;

// Adapter: match your old hook's return shape
export function useTheme() {
  const { colors, name, setTheme, isTransitioning } = api.useTheme();
  return {
    colors,
    theme: name,         // your old code used 'theme' not 'name'
    isDark: name === 'dark',
    toggle: () => setTheme(name === 'light' ? 'dark' : 'light'),
    setTheme,
    isTransitioning,
  };
}
```

This lets you migrate incrementally — old components keep working while you
update them one by one to use the new API shape.
