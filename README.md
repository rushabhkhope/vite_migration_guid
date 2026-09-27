

Skip to content
Using Gmail with screen readers
1 of 19,206
(no subject)
Inbox

rushabh khope
Attachments
10:52 AM (0 minutes ago)
to me


 One attachment
  •  Scanned by Gmail
# Migrating a React App from Create React App (CRA) to Vite

This guide walks through every step required to migrate an existing React project from **Create React App (`react-scripts`)** to **Vite**. Follow the steps in order. Each step explains **what** to do, **why** it is needed, and **how** to verify it.

> **Platform note:** Shell commands and npm scripts in this guide are written for **macOS** (and Linux). Windows-specific differences are called out where relevant.

---

## Table of Contents

1. [Remove CRA](#1-remove-cra)
2. [Add Vite](#2-add-vite)
3. [Update `package.json` Scripts](#3-update-packagejson-scripts)
4. [Move `index.html` to the Project Root](#4-move-indexhtml-to-the-project-root)
5. [Replace `%PUBLIC_URL%` in `index.html`](#5-replace-public_url-in-indexhtml)
6. [Create the Vite Config File](#6-create-the-vite-config-file)
7. [Configure Path Aliases](#7-configure-path-aliases)
8. [Add `vite-env.d.ts` (TypeScript Declarations)](#8-add-vite-envdts-typescript-declarations)
9. [Migrate Environment Variables](#9-migrate-environment-variables)
10. [Support SVGs as React Components (SVGR)](#10-support-svgs-as-react-components-svgr)
11. [Add ESLint to the Vite Dev Server](#11-add-eslint-to-the-vite-dev-server)
12. [Prevent Duplicate Package Instances (`dedupe`)](#12-prevent-duplicate-package-instances-dedupe)
13. [Handle CommonJS Packages](#13-handle-commonjs-packages)
14. [Upgrade Incompatible Packages](#14-upgrade-incompatible-packages)
15. [Testing: Migrate from Jest to Vitest](#15-testing-migrate-from-jest-to-vitest)
16. [Complete Reference `vite.config.mts`](#16-complete-reference-viteconfigmts)
17. [Final Migration Checklist](#17-final-migration-checklist)

---

## 1. Remove CRA

Remove `react-scripts` and the Babel plugins that CRA relied on. Vite uses **esbuild** (in development) and **Rollup** (for production builds) instead of Webpack + Babel, so these packages are no longer needed.

```bash
yarn remove react-scripts @babel/plugin-syntax-flow @babel/plugin-transform-react-jsx
```

**What each package did:**

| Package | Purpose in CRA |
|---|---|
| `react-scripts` | Bundled Webpack, Babel, Jest, the dev server, and all CRA build scripts |
| `@babel/plugin-syntax-flow` | Allowed Babel to parse Flow type annotations |
| `@babel/plugin-transform-react-jsx` | Converted JSX into `React.createElement` / JSX runtime calls |

> After removing `react-scripts`, the old `start`, `build`, `test`, and `eject` scripts in `package.json` will stop working. They are replaced in [Step 3](#3-update-packagejson-scripts).

---

## 2. Add Vite

Install Vite and the official React plugin:

```bash
yarn add vite @vitejs/plugin-react
```

> ⚠️ **Important:** Make sure these are added to **`dependencies`**, **not** `devDependencies`. Do **not** use the `-D` / `--dev` flag for this command.
>
> Verify in `package.json` that both packages appear under `"dependencies"`.

| Package | Purpose |
|---|---|
| `vite` | The build tool and dev server |
| `@vitejs/plugin-react` | Enables JSX/TSX transformation and React Fast Refresh (hot reloading that preserves component state) |

---

## 3. Update `package.json` Scripts

Replace the old CRA scripts with Vite scripts. These scripts are written to work on **macOS**.

```json
"scripts": {
  "start": "VITE_VERSION=$npm_package_version vite",
  "startw": "vite"
}
```

**Explanation:**

- **`start`** — Starts the Vite dev server and injects the app version from `package.json` as an environment variable named `VITE_VERSION`.
  - `$npm_package_version` is automatically set by yarn/npm to the `"version"` field of `package.json`.
  - Because the variable starts with `VITE_`, it becomes accessible in the app as `import.meta.env.VITE_VERSION`.
  - The `VAR=value command` syntax works on **macOS/Linux** shells.
- **`startw`** — Starts the Vite dev server without setting any inline variable. Use this on **Windows**, where the `VAR=value command` syntax is not supported by `cmd.exe`/PowerShell.

> 💡 **Tip (optional):** If you want one script that works on every OS, install `cross-env` and use `"start": "cross-env VITE_VERSION=$npm_package_version vite"`.

> 💡 **Recommended additional scripts (optional):** CRA also provided `build` and `test`. You will usually want equivalents:
> ```json
> "build": "vite build",
> "preview": "vite preview",
> "test": "vitest"
> ```

> Note: JSON does not allow a trailing comma after the last entry in an object — make sure the last script line has no comma.

---

## 4. Move `index.html` to the Project Root

### Why

CRA served `index.html` from the `public/` folder and injected the JavaScript bundle into it automatically. **Vite treats `index.html` as the entry point of the app** and expects it at the **project root** — not in `src/` or `public/`.

### What to do

1. Move `public/index.html` to the project root.
2. Verify that `index.html` now sits **parallel to (in the same folder as) `package.json`**:

```text
my-app/
├── index.html        ← moved here
├── package.json
├── vite.config.mts
├── public/
└── src/
```

```bash
mv public/index.html ./index.html
```

3. Add a script tag that points to your app's entry file, just before the closing `</body>` tag:

```html
<script type="module" src="/src/main.jsx"></script>
```

> Point `src` to your **actual** entry file. CRA projects usually use `src/index.js`, `src/index.jsx`, or `src/index.tsx` — either rename the file to `main.jsx`/`main.tsx` or change the `src` path to match (for example `/src/index.tsx`).

### About `type="module"`

- ⚠️ **Make sure to add `type="module"`** on the script tag. This tells the browser to load the file as an **ES module**, which is how Vite serves code during development.
- As noted in the original steps: if `type="module"` is added here, it does not need to be repeated in `package.json`.

> 📝 **Clarification:** These are two different settings that control two different things:
> - `type="module"` in **`index.html`** controls how the **browser** loads your app code. It is always required with Vite.
> - `"type": "module"` in **`package.json`** controls how **Node.js** treats `.js` files in the project (including config files). It is optional. If you don't add it, use the `.mts` config extension (see [Step 6](#6-create-the-vite-config-file)) so the Vite config is still parsed as ESM.

---

## 5. Replace `%PUBLIC_URL%` in `index.html`

### Why

`%PUBLIC_URL%` is a **CRA-specific placeholder** that Webpack replaced at build time. Vite does not understand it, so any remaining occurrences in `index.html` will produce broken links (for example to the favicon or manifest). Vite serves files in `public/` from the root path `/`.

### What to do

Replace all occurrences:

```bash
# Replace "%PUBLIC_URL%/" with "/"
sed -i '' 's|%PUBLIC_URL%/|/|g' index.html

# Remove any remaining "%PUBLIC_URL%"
sed -i '' 's|%PUBLIC_URL%||g' index.html
```

**Example result:**

```html
<!-- Before -->
<link rel="icon" href="%PUBLIC_URL%/favicon.ico" />

<!-- After -->
<link rel="icon" href="/favicon.ico" />
```

> The `sed -i ''` syntax is for **macOS**. On Linux, use `sed -i 's|...|...|g' index.html` (without the empty `''`).

Verify nothing is left:

```bash
grep -n "%PUBLIC_URL%" index.html   # should print nothing
```

---

## 6. Create the Vite Config File

### Choosing the file extension

Create a Vite config file at the project root (parallel to `package.json`). Choose the extension carefully:

| File | When to use |
|---|---|
| `vite.config.ts` | The most common choice. Vite handles the transpilation for you. Use this unless you have a specific reason not to. |
| `vite.config.mts` | Use this if your `package.json` does **not** have `"type": "module"` and you want to guarantee the config is parsed as ESM. It forces ES module syntax (`import`/`export`) regardless of the project's module setting. |
| `vite.config.cts` | The opposite: forces CommonJS (`require`/`module.exports`) even if your project is set to ESM. |
| `vite.config.js` | Plain JavaScript config. |

> ✅ **Project rule:** Because we add `type="module"` in `index.html` and do **not** add `"type": "module"` to `package.json`, **use `vite.config.mts`**. The rest of this guide uses `vite.config.mts`.

### Basic config

```ts
// vite.config.mts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    outDir: 'build' // CRA's default build output folder (Vite's default is "dist")
  }
});
```

**Explanation:**

- `defineConfig` — Gives you type checking and autocompletion for the config.
- `react()` — Enables JSX/TSX transformation and Fast Refresh.
- `build.outDir: 'build'` — Keeps the production output in `build/` (like CRA) so existing deployment pipelines, Dockerfiles, and CI scripts keep working without changes.

---

## 7. Configure Path Aliases

### The problem

The CRA project was using **path aliases / absolute imports** such as `app/...`, `store/...`, `services/...`, and `utils/...`. `react-scripts` resolved these automatically through the `baseUrl` setting in `tsconfig.json` (likely `"baseUrl": "src"`).

**Vite does not honor `tsconfig.json` paths by default**, so these imports will fail after migration. There are two ways to fix this. Option B is required if the codebase already uses imports like `store/configureStore`.

### Option A — `@` alias using `resolve.alias`

Add this to `vite.config.mts`:

```ts
import path from 'path';

export default defineConfig({
  // ...
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src')
    }
  }
});
```

This tells Vite that whenever it sees an import starting with `@`, it should resolve it to the `src` directory.

So if you write:

```ts
import MyComponent from '@/app/components/MyComponent';
```

Vite translates it to:

```ts
import MyComponent from '/absolute/path/to/your/project/src/app/components/MyComponent';
```

> Remember to add `import path from 'path';` at the top of the config.
>
> For TypeScript to understand `@/...` in the editor, also add a matching entry in `tsconfig.json`:
> ```json
> "compilerOptions": {
>   "baseUrl": ".",
>   "paths": { "@/*": ["src/*"] }
> }
> ```

### Option B (extra config) — Keep existing bare imports with `vite-tsconfig-paths`

If the code has imports like this (bare paths relative to `src`):

```ts
import { configureAppStore } from 'store/configureStore';
```

then install `vite-tsconfig-paths` to resolve them:

```bash
yarn add -D vite-tsconfig-paths
```

Then update `vite.config.mts`:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  build: { outDir: 'build' },
  base: '/',
  server: {
    open: true,
    port: 3000,
    hmr: { overlay: true },
    watch: { usePolling: true }
  },
  define: {
    'process.env': {}
  }
});
```

This is **exactly what CRA did** — `baseUrl: "src"` lets you write `import X from "store/configureStore"` and it resolves to `src/store/configureStore`. The `vite-tsconfig-paths` plugin makes Vite respect that setting, so **no import statements need to be rewritten**.

**Explanation of the other options in this config:**

| Option | Purpose |
|---|---|
| `tsconfigPaths()` | Reads `baseUrl` / `paths` from `tsconfig.json` and applies them to Vite's module resolution |
| `base: '/'` | Public base path the app is served from (same as CRA's default) |
| `server.open: true` | Opens the browser automatically when the dev server starts (like CRA) |
| `server.port: 3000` | Keeps CRA's default port so bookmarks, OAuth redirect URLs, and backend CORS settings still work (Vite's default is `5173`) |
| `server.hmr.overlay: true` | Shows compile/runtime errors as an overlay in the browser |
| `server.watch.usePolling: true` | Uses polling to detect file changes — useful in Docker, VMs, WSL, and network drives where native file events are unreliable (uses more CPU) |
| `define: { 'process.env': {} }` | Replaces `process.env` with an empty object so old code or libraries that reference `process.env` don't crash with `process is not defined` in the browser |

> ⚠️ `define: { 'process.env': {} }` only prevents crashes. It does **not** make `process.env.REACT_APP_*` values work — those must be migrated as described in [Step 9](#9-migrate-environment-variables).

---

## 8. Add `vite-env.d.ts` (TypeScript Declarations)

### What it is

`vite-env.d.ts` is a **TypeScript declaration file** that tells TypeScript about Vite-specific features that don't exist in standard TypeScript.

Without it, TypeScript doesn't know what these things are:

- **`import.meta.env`** — Vite's way to access environment variables (`VITE_API_URL`, etc.)
- **`import.meta.hot`** — Vite's Hot Module Replacement (HMR) API
- **Asset imports** — like `import logo from './logo.png'` (returns a URL string) or `import styles from './app.module.css'`

### Where to put it

The original steps mention placing it parallel to `package.json` **and** at `src/vite-env.d.ts`. Only **one** file is needed:

- **Recommended:** `src/vite-env.d.ts` — it is automatically picked up because `tsconfig.json` normally has `"include": ["src"]`.
- If you place it at the root (parallel to `package.json`), make sure it is listed in the `include` array of `tsconfig.json`, otherwise TypeScript will ignore it.

> If the project has a CRA-generated `src/react-app-env.d.ts`, delete it — it references `react-scripts`, which no longer exists.

### Minimal content

```ts
/// <reference types="vite/client" />
```

The full version, including SVG support, is shown in [Step 10](#10-support-svgs-as-react-components-svgr).

---

## 9. Migrate Environment Variables

### Why

- CRA exposes variables prefixed with **`REACT_APP_`** through **`process.env`**.
- Vite exposes variables prefixed with **`VITE_`** through **`import.meta.env`**.

Variables that don't start with `VITE_` are **not** exposed to the client code (this is a security feature so secrets aren't leaked into the bundle).

### Step 9.1 — Rename variables in all `.env` files

Change every variable name in `.env`, `.env.local`, `.env.development`, `.env.production`, etc.

```bash
# Before
REACT_APP_SYNCFUSION_LICENSE_KEY=your-key

# After
VITE_SYNCFUSION_LICENSE_KEY=your-key
```

### Step 9.2 — Replace usages across the codebase

Replace every `process.env.REACT_APP_*` reference in **all** files with `import.meta.env.VITE_*`:

```ts
// Before
process.env.REACT_APP_SYNCFUSION_LICENSE_KEY

// After
import.meta.env.VITE_SYNCFUSION_LICENSE_KEY
```

Another example:

```ts
const saleforceHost = import.meta.env.VITE_API_SF_BASE_URL || '';
```

The `|| ''` provides a safe fallback so the value is never `undefined`.

**Find every remaining reference:**

```bash
grep -rn "REACT_APP_" --include="*.{js,jsx,ts,tsx}" src .env*
grep -rn "process.env" src
```

**Bulk replace (macOS):**

```bash
# Rename in .env files
sed -i '' 's/REACT_APP_/VITE_/g' .env*

# Rename in source files
grep -rl "process.env.REACT_APP_" src | xargs sed -i '' 's/process\.env\.REACT_APP_/import.meta.env.VITE_/g'
```

> Also update CI/CD pipelines, Docker files, and hosting dashboards (Netlify, Vercel, Azure, etc.) that define `REACT_APP_*` variables.

> `process.env.NODE_ENV` should become `import.meta.env.MODE`, or use `import.meta.env.DEV` / `import.meta.env.PROD` (booleans).

### (Optional) Type your environment variables

Add this to `vite-env.d.ts` to get autocompletion and type safety:

```ts
interface ImportMetaEnv {
  readonly VITE_SYNCFUSION_LICENSE_KEY: string;
  readonly VITE_API_SF_BASE_URL: string;
  readonly VITE_VERSION: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## 10. Support SVGs as React Components (SVGR)

### How CRA handled SVGs

In CRA, Webpack is configured behind the scenes with a loader called **SVGR**. SVGR takes an SVG file and automatically converts it into a React component. So when you write:

```tsx
import { ReactComponent as VcheckLogo } from './vcheck-logo.svg';

<VcheckLogo />
```

CRA's Webpack is doing this under the hood:

1. Reads `vcheck-logo.svg` (which is just XML markup like `<svg><path d="..."/></svg>`).
2. SVGR transforms that XML into a React component function.
3. Exports it as `ReactComponent` so you can use it like `<VcheckLogo />`.

This is **not standard JavaScript behavior**. A `.svg` file is not a JS module — you can't normally import a component from it. CRA just made it feel that way by wiring up the transformation automatically.

### Why it breaks in Vite

**Vite doesn't include this transformation.** When Vite sees the SVG import, it treats the file as a **static asset** (like an image URL), not a React component. So `{ ReactComponent as VcheckLogo }` doesn't exist, and the import fails.

That's why you need **`vite-plugin-svgr`** — it adds the same SVG-to-React-component transformation that CRA had built in.

### Step 10.1 — Install the plugin

```bash
yarn add vite-plugin-svgr
```

### Step 10.2 — Add it to the `plugins` array in `vite.config.mts`

```ts
import svgr from 'vite-plugin-svgr';

export default defineConfig({
  plugins: [
    react(),
    svgr({
      svgrOptions: {
        exportType: 'named', // Enables named exports (ReactComponent), matching CRA
        ref: true,           // Adds a ref to the component (forwardRef)
        svgo: false,         // Disables SVGO optimization
        titleProp: true      // Adds a title prop to the component (accessibility)
      },
      include: '**/*.svg'    // Specifies the files to include (all SVGs)
    })
  ]
});
```

> With `include: '**/*.svg'`, SVG imports keep working with CRA's syntax `import { ReactComponent as Logo } from './logo.svg'`, so existing components don't need to change.

### Step 10.3 — Add SVG type declarations to `src/vite-env.d.ts`

This tells TypeScript that `.svg` files have a **named `ReactComponent` export** (the React component from SVGR) and a **default export** (the URL string).

```ts
/// <reference types="vite/client" />

declare module '*.svg' {
  import React from 'react';
  export const ReactComponent: React.FC<React.SVGProps<SVGSVGElement>>;
  const src: string;
  export default src;
}
```

---

## 11. Add ESLint to the Vite Dev Server

CRA showed ESLint errors in the terminal and browser during development. To keep that behavior, add `vite-plugin-eslint2`.

### Step 11.1 — Install the plugin

```bash
yarn add vite-plugin-eslint2
```

### Step 11.2 — Import it and add it to the `plugins` array in `vite.config.mts`

```ts
import eslint from 'vite-plugin-eslint2';

export default defineConfig({
  plugins: [
    react(),
    eslint({
      overrideConfigFile: 'eslint.config.js' // Path to the project's ESLint (flat) config
    })
  ]
});
```

> Make sure `eslint.config.js` exists at the project root. If the project still uses the old `.eslintrc` format that extended `react-app`, remove `eslint-config-react-app` references, since that config came from CRA.

---

## 12. Prevent Duplicate Package Instances (`dedupe`)

### Why

When multiple copies of the same library end up in the bundle (for example, from nested `node_modules`), you can get errors such as:

- **"Invalid hook call"** (two copies of React)
- MUI/Emotion theme not applying, or styles duplicated
- Syncfusion licence or component registration issues

`resolve.dedupe` forces Vite to always resolve these packages to a **single copy** from the project root.

### What to add

Add this inside the **`resolve`** section of `vite.config.mts`:

```ts
resolve: {
  // Prevent duplicate package instances
  dedupe: [
    'react',
    'react-dom',
    '@syncfusion/ej2-base',
    '@emotion/react',
    '@emotion/styled',
    '@mui/material',
    '@mui/system',
    '@mui/base'
  ]
}
```

---

## 13. Handle CommonJS Packages

### Why

Vite is built around **ES modules**. Some older packages in `node_modules` are published only as **CommonJS** (`require` / `module.exports`), or mix both styles. These can cause errors like `require is not defined` or missing default exports in production builds.

### What to add

In case any package uses CJS, use this `build` configuration:

```ts
build: {
  outDir: 'build',
  sourcemap: false,
  minify: false,
  // Handle CommonJS packages
  commonjsOptions: {
    include: [/node_modules/],
    transformMixedEsModules: true,
    ignoreTryCatch: false
  }
}
```

**Explanation:**

| Option | Purpose |
|---|---|
| `outDir: 'build'` | Keeps CRA's output folder |
| `sourcemap: false` | Does not generate source maps in production (smaller output, source code not exposed). Set to `true` if you need to debug production. |
| `minify: false` | Disables minification. Useful while debugging the migration; consider removing it (the default is `'esbuild'`) for smaller production bundles. |
| `commonjsOptions.include: [/node_modules/]` | Applies CommonJS-to-ESM conversion to all packages in `node_modules` |
| `transformMixedEsModules: true` | Handles files that mix `import` and `require` in the same module |
| `ignoreTryCatch: false` | Also converts `require()` calls that are wrapped in `try/catch` blocks |

---

## 14. Upgrade Incompatible Packages

If you have older packages, **update them to the latest versions compatible with Vite**.

For example, **MUI needs to be on v7.0.0** (or later):

```bash
yarn add @mui/material@^7.0.0 @mui/system@^7.0.0 @emotion/react @emotion/styled
```

> After major version upgrades (such as MUI v5/v6 → v7), check the library's official migration guide for breaking changes and update affected components.

Other packages worth checking:

- Anything that depends on `react-scripts` or Webpack-specific loaders
- Packages that reference `process.env` directly
- Old CommonJS-only packages (see [Step 13](#13-handle-commonjs-packages))

---

## 15. Testing: Migrate from Jest to Vitest

CRA bundled **Jest** inside `react-scripts`. After removing it, tests should run with **Vitest**, which uses the same Vite config and is largely Jest-compatible.

### Step 15.1 — Install Vitest (if not already installed)

```bash
yarn add -D vitest jsdom
```

Add a `test` section to `vite.config.mts` (optional but recommended):

```ts
/// <reference types="vitest" />

export default defineConfig({
  // ...
  test: {
    globals: true,          // Allows describe/it/expect without importing (like Jest)
    environment: 'jsdom'    // Browser-like environment for React components
  }
});
```

### Step 15.2 — Replace `require()`-based mocks with `vi.mocked`

Jest-style tests often used `require()` to grab a mocked function. Vite/Vitest works with ES modules, so `require` should be replaced.

**Remove this:**

```ts
const mockGetZipFilename = require('../../../index').getZipFilename;
```

**Replace it with:**

```ts
import { vi } from 'vitest';
import { getZipFilename } from '../../../index';

const mockGetZipFilename = vi.mocked(getZipFilename);
```

**Explanation:**

- `import { vi } from 'vitest';` — `vi` is Vitest's equivalent of Jest's `jest` object (`jest.fn()` → `vi.fn()`, `jest.mock()` → `vi.mock()`, `jest.spyOn()` → `vi.spyOn()`).
- `vi.mocked(fn)` — Does **not** create a mock itself; it tells TypeScript that the function is already mocked, so methods like `.mockReturnValue()` are correctly typed.

> ⚠️ The module must still be mocked with `vi.mock(...)` for `vi.mocked` to work:
> ```ts
> vi.mock('../../../index', () => ({
>   getZipFilename: vi.fn()
> }));
> ```

### Common Jest → Vitest replacements

| Jest | Vitest |
|---|---|
| `jest.fn()` | `vi.fn()` |
| `jest.mock()` | `vi.mock()` |
| `jest.spyOn()` | `vi.spyOn()` |
| `jest.clearAllMocks()` | `vi.clearAllMocks()` |
| `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| `require('module').fn` | `import { fn } from 'module'` + `vi.mocked(fn)` |

---

## 16. Complete Reference `vite.config.mts`

This combines every configuration from the steps above into one file:

```ts
// vite.config.mts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';
import svgr from 'vite-plugin-svgr';
import eslint from 'vite-plugin-eslint2';
import path from 'path';

export default defineConfig({
  plugins: [
    // JSX/TSX transformation + React Fast Refresh
    react(),

    // Resolve bare imports like 'store/configureStore' using tsconfig baseUrl (CRA behavior)
    tsconfigPaths(),

    // Convert SVG files into React components (CRA's built-in SVGR behavior)
    svgr({
      svgrOptions: {
        exportType: 'named', // Enables named exports
        ref: true,           // Adds a ref to the component
        svgo: false,         // Disables SVGO optimization
        titleProp: true      // Adds a title prop to the component
      },
      include: '**/*.svg'    // Specifies the files to include
    }),

    // Show lint errors during development (CRA behavior)
    eslint({
      overrideConfigFile: 'eslint.config.js'
    })
  ],

  resolve: {
    // Optional "@" alias → src
    alias: {
      '@': path.resolve(__dirname, './src')
    },

    // Prevent duplicate package instances
    dedupe: [
      'react',
      'react-dom',
      '@syncfusion/ej2-base',
      '@emotion/react',
      '@emotion/styled',
      '@mui/material',
      '@mui/system',
      '@mui/base'
    ]
  },

  // Public base path
  base: '/',

  // Dev server settings (mirrors CRA defaults)
  server: {
    open: true,
    port: 3000,
    hmr: { overlay: true },
    watch: { usePolling: true }
  },

  // Prevent "process is not defined" errors from legacy code/libraries
  define: {
    'process.env': {}
  },

  build: {
    outDir: 'build', // CRA's default build output
    sourcemap: false,
    minify: false,
    // Handle CommonJS packages
    commonjsOptions: {
      include: [/node_modules/],
      transformMixedEsModules: true,
      ignoreTryCatch: false
    }
  }
});
```

> `__dirname` is not a native ESM variable, but Vite makes it available when it loads the config file. If your editor or tooling complains, use this instead:
> ```ts
> import { fileURLToPath } from 'url';
> const __dirname = path.dirname(fileURLToPath(import.meta.url));
> ```

---

## 17. Final Migration Checklist

- [ ] Removed `react-scripts`, `@babel/plugin-syntax-flow`, `@babel/plugin-transform-react-jsx`
- [ ] Installed `vite` and `@vitejs/plugin-react` as **dependencies** (not devDependencies)
- [ ] Updated `package.json` scripts (`start`, `startw`, and optionally `build`, `preview`, `test`)
- [ ] Moved `index.html` from `public/` to the project root (parallel to `package.json`)
- [ ] Added `<script type="module" src="/src/main.jsx"></script>` pointing to the real entry file
- [ ] Replaced all `%PUBLIC_URL%` occurrences in `index.html`
- [ ] Created `vite.config.mts` with the React plugin and `outDir: 'build'`
- [ ] Configured path aliases (`@` alias and/or `vite-tsconfig-paths`)
- [ ] Added `src/vite-env.d.ts` (Vite client types + SVG declarations) and removed `react-app-env.d.ts`
- [ ] Renamed all `REACT_APP_*` variables to `VITE_*` in `.env` files, CI/CD, and hosting
- [ ] Replaced all `process.env.REACT_APP_*` with `import.meta.env.VITE_*` in code
- [ ] Installed and configured `vite-plugin-svgr`
- [ ] Installed and configured `vite-plugin-eslint2`
- [ ] Added `resolve.dedupe` for React, MUI, Emotion, and Syncfusion
- [ ] Added `build.commonjsOptions` for CommonJS packages
- [ ] Upgraded incompatible packages (e.g. MUI to v7.0.0+)
- [ ] Migrated tests to Vitest (`require` mocks → `vi.mocked`)
- [ ] Ran `yarn start` — app loads on `http://localhost:3000` with no console errors
- [ ] Ran `yarn build` — output generated in `build/`
- [ ] Ran the test suite successfully
README (1).md
Displaying README (1).md.
