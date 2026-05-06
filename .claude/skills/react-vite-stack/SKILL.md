---
name: react-vite-stack
description: Bootstrap a complete React frontend application with Vite + TypeScript + TanStack Router (file-based) + TanStack Query + Mantine UI (dark/light toggle) + Vitest + ESLint + Prettier. Use this whenever the user wants to create a new React app, scaffold a new frontend project, start a web application, or bootstrap any TypeScript React codebase — even if they don't explicitly name the libraries. If they mention "new frontend", "new React app", "scaffold an app", "start a project with Mantine", or similar, trigger this skill.
---

# React Vite Stack Bootstrapper

Scaffold a production-ready React frontend with:
- **Vite** + **TypeScript** (build tooling)
- **TanStack Router** (file-based routing, auto code splitting)
- **TanStack Query** + DevTools (server state management)
- **Mantine** v7 (UI components, dark/light mode toggle, system preference default)
- **Motion** (default animation library)
- **Vitest** + Testing Library (unit testing)
- **ESLint** + **Prettier** (linting and formatting)

## Step 1 — Ask for the app name

Ask: "What would you like to name your app?"

Use the name directly as the project directory (kebab-case is conventional, e.g. `my-app`). Store it as `<APP_NAME>` in the steps below.

## Step 2 — Scaffold with Vite

```bash
pnpm create vite <APP_NAME> --template react-ts
cd <APP_NAME>
```

## Step 3 — Install dependencies

Run all installs from inside the project directory:

```bash
# Mantine UI (core + hooks)
pnpm add @mantine/core @mantine/hooks

# TanStack Router + file-based routing Vite plugin
pnpm add @tanstack/react-router
pnpm add -D @tanstack/router-plugin

# TanStack Query + DevTools
pnpm add @tanstack/react-query @tanstack/react-query-devtools

# PostCSS — required for Mantine's CSS variables to work
pnpm add -D postcss postcss-preset-mantine postcss-simple-vars

# Animations
pnpm add motion

# Testing
pnpm add -D vitest @vitest/coverage-v8 @testing-library/react @testing-library/jest-dom jsdom

# Prettier + ESLint integration (Vite template already includes eslint, typescript-eslint, and the react plugins)
pnpm add -D eslint-config-prettier prettier
```

### Animation convention (agent default)

- Use `motion` (`motion/react`) as the default animation library.
- Prefer `AnimatePresence` + `motion.*` for enter/exit transitions.
- Prefer `layout` animations for list reordering/filtering.
- Avoid manual timeout-based CSS animation state unless Motion cannot cover the case.

## Step 4 — Write configuration files

### `postcss.config.cjs`
Required for Mantine — maps CSS custom properties to breakpoint values.

```js
module.exports = {
  plugins: {
    'postcss-preset-mantine': {},
    'postcss-simple-vars': {
      variables: {
        'mantine-breakpoint-xs': '36em',
        'mantine-breakpoint-sm': '48em',
        'mantine-breakpoint-md': '62em',
        'mantine-breakpoint-lg': '75em',
        'mantine-breakpoint-xl': '88em',
      },
    },
  },
}
```

### `vite.config.ts`
The TanStack Router plugin must come **before** the React plugin — it generates `src/routeTree.gen.ts` which React needs.

```ts
/// <reference types="vitest" />
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { TanStackRouterVite } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    TanStackRouterVite({ target: 'react', autoCodeSplitting: true }),
    react(),
  ],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
  },
})
```

### `eslint.config.js`
Replace the generated one entirely. Adds Prettier as a rule (formatting errors show as ESLint warnings) and ignores the auto-generated router file.

```js
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import tseslint from 'typescript-eslint'
import prettierConfig from 'eslint-config-prettier'

export default tseslint.config(
  { ignores: ['dist', 'src/routeTree.gen.ts'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', { allowConstantExport: true }],
    },
  },
  prettierConfig,
)
```

### `.prettierrc`
```json
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

### `.prettierignore`
```
src/routeTree.gen.ts
dist/
```

### `.vscode/settings.json`
Makes the auto-generated router file read-only and hidden from search so it doesn't clutter the workspace.

```json
{
  "files.readonlyInclude": {
    "**/routeTree.gen.ts": true
  },
  "search.exclude": {
    "**/routeTree.gen.ts": true
  }
}
```

## Step 5 — Write source files

### `src/test/setup.ts`
```ts
import '@testing-library/jest-dom'
```

### `src/main.tsx`
Sets up the router and query client. The router context includes the query client so routes can use it for prefetching.

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { RouterProvider, createRouter } from '@tanstack/react-router'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { routeTree } from './routeTree.gen'

const queryClient = new QueryClient()

const router = createRouter({
  routeTree,
  context: { queryClient },
})

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
    </QueryClientProvider>
  </StrictMode>,
)
```

### `src/routes/__root.tsx`
Root layout wrapping all pages. Provides Mantine (with dark mode support), a nav header, and TanStack Query DevTools. The `Header` component is a separate function so it can use `useMantineColorScheme`, which requires being inside a `MantineProvider`.

```tsx
import { createRootRoute, Link, Outlet } from '@tanstack/react-router'
import {
  MantineProvider,
  AppShell,
  Group,
  Text,
  ActionIcon,
  Anchor,
  createTheme,
  useMantineColorScheme,
} from '@mantine/core'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import '@mantine/core/styles.css'

const theme = createTheme({})

function ColorSchemeToggle() {
  const { colorScheme, toggleColorScheme } = useMantineColorScheme()
  return (
    <ActionIcon onClick={() => toggleColorScheme()} variant="default" size="lg" aria-label="Toggle color scheme">
      {colorScheme === 'dark' ? '☀️' : '🌙'}
    </ActionIcon>
  )
}

function Header() {
  return (
    <AppShell.Header>
      <Group h="100%" px="md" justify="space-between">
        <Group gap="lg">
          <Text fw={700} size="lg">
            <APP_NAME_PLACEHOLDER />
          </Text>
          <Anchor component={Link} to="/" underline="hover">
            Home
          </Anchor>
          <Anchor component={Link} to="/about" underline="hover">
            About
          </Anchor>
        </Group>
        <ColorSchemeToggle />
      </Group>
    </AppShell.Header>
  )
}

function RootLayout() {
  return (
    <MantineProvider theme={theme} defaultColorScheme="auto">
      <AppShell header={{ height: 60 }} padding="md">
        <Header />
        <AppShell.Main>
          <Outlet />
        </AppShell.Main>
      </AppShell>
      <ReactQueryDevtools initialIsOpen={false} />
    </MantineProvider>
  )
}

export const Route = createRootRoute({
  component: RootLayout,
})
```

**Important:** Replace `<APP_NAME_PLACEHOLDER />` with the actual app name as a plain string (e.g. `My App`). Use a human-readable title-cased version of the project name.

### `src/routes/index.tsx`
Welcome page demonstrating TanStack Query with a live fetch from JSONPlaceholder and Motion-based list animations.

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { useQuery } from '@tanstack/react-query'
import { AnimatePresence, motion } from 'motion/react'
import {
  Container,
  Title,
  Text,
  Card,
  Stack,
  Loader,
  Alert,
  Badge,
  Group,
} from '@mantine/core'

interface Todo {
  id: number
  title: string
  completed: boolean
}

const quickTips = ['Add your first feature route', 'Wire your real API', 'Ship 🚀']

async function fetchSampleTodo(): Promise<Todo> {
  const res = await fetch('https://jsonplaceholder.typicode.com/todos/1')
  if (!res.ok) throw new Error('Network response was not ok')
  return res.json()
}

function WelcomePage() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['sample-todo'],
    queryFn: fetchSampleTodo,
  })

  return (
    <Container size="sm" mt="xl">
      <Stack gap="lg">
        <div>
          <Title>Welcome!</Title>
          <Text c="dimmed" mt="xs">
            Your app is ready. This page demonstrates TanStack Query fetching live data.
          </Text>
        </div>

        <Card withBorder radius="md" p="lg">
          <Text fw={500} mb="sm">
            Sample query — fetching a todo from JSONPlaceholder:
          </Text>
          {isLoading && <Loader size="sm" />}
          {error && (
            <Alert color="red" title="Error">
              {error.message}
            </Alert>
          )}
          {data && (
            <Group gap="xs">
              <Badge color={data.completed ? 'green' : 'orange'}>
                {data.completed ? 'Done' : 'Pending'}
              </Badge>
              <Text size="sm">{data.title}</Text>
            </Group>
          )}
        </Card>

        <Card withBorder radius="md" p="lg">
          <Text fw={500} mb="sm">
            Motion demo — animated quick tips:
          </Text>
          <AnimatePresence mode="popLayout" initial={false}>
            {quickTips.map((tip) => (
              <motion.div
                key={tip}
                layout
                initial={{ opacity: 0, y: 8 }}
                animate={{ opacity: 1, y: 0 }}
                exit={{ opacity: 0, y: -8 }}
                transition={{ duration: 0.18, ease: 'easeOut' }}
              >
                <Text size="sm">• {tip}</Text>
              </motion.div>
            ))}
          </AnimatePresence>
        </Card>

        <Text size="sm" c="dimmed">
          Remove this demo card and replace it with your own content. The stack is
          configured and ready to go.
        </Text>
      </Stack>
    </Container>
  )
}

export const Route = createFileRoute('/')({
  component: WelcomePage,
})
```

### `src/routes/about.tsx`
```tsx
import { createFileRoute } from '@tanstack/react-router'
import { Container, Title, Text, Stack, List, ThemeIcon } from '@mantine/core'

function AboutPage() {
  const stack = [
    'React 18 + TypeScript',
    'Vite (build tool)',
    'TanStack Router (file-based routing)',
    'TanStack Query (server state)',
    'Mantine v7 (UI components)',
    'Motion (animations)',
    'Vitest + Testing Library (tests)',
    'ESLint + Prettier (code quality)',
  ]

  return (
    <Container size="sm" mt="xl">
      <Stack gap="lg">
        <div>
          <Title>About</Title>
          <Text c="dimmed" mt="xs">
            This app was bootstrapped with the react-vite-stack skill.
          </Text>
        </div>
        <div>
          <Text fw={500} mb="sm">
            Stack:
          </Text>
          <List
            spacing="xs"
            icon={
              <ThemeIcon size={20} radius="xl" color="blue">
                ✓
              </ThemeIcon>
            }
          >
            {stack.map((item) => (
              <List.Item key={item}>{item}</List.Item>
            ))}
          </List>
        </div>
      </Stack>
    </Container>
  )
}

export const Route = createFileRoute('/about')({
  component: AboutPage,
})
```

## Step 6 — Clean up default Vite boilerplate

Delete files that are replaced by the router structure:
```bash
rm src/App.tsx src/App.css
```

You can also remove `src/index.css` and its import from `src/main.tsx` — Mantine's `styles.css` import in `__root.tsx` handles global styles. Or keep `index.css` for any custom global overrides.

## Step 7 — Update package.json scripts

Add a `test` and `coverage` script if not already present:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview",
    "test": "vitest",
    "coverage": "vitest run --coverage"
  }
}
```

## Step 8 — Verify

```bash
pnpm dev
```

The browser should open to the welcome page with the header showing the app name, a Home/About navigation, a dark mode toggle, and a live TanStack Query fetch. The About route should be navigable.

If `src/routeTree.gen.ts` doesn't exist yet, starting the dev server generates it automatically.

Tell the user: "Your app is ready at `http://localhost:5173`. The welcome page shows a live TanStack Query fetch, the About page is wired up, and the dark mode toggle uses your system preference by default."
