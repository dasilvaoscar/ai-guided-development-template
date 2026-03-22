# TypeScript Scaffold

## Confirmation summary

Before creating any files, present this to the user:

> I'll set up the following:
> - **Package manager**: Yarn
> - **TypeScript**: strict mode, `@` → `src` path alias
> - **Tooling**: ESLint with `@typescript-eslint`
> - **Scripts**: `build` (tsc), `start`, `lint`
>
> Confirm to proceed?

Only scaffold after the user confirms. If they decline, ask what they'd prefer instead.

## Step 1 — Clarify minimal requirements

Ask only if unclear:
- **Runtime target**: "Node script / backend", "library", or "browser app"? (default: Node)
- **Package manager**: default Yarn; switch only if user asks.

If vague, assume: Node LTS, Yarn, single-package project.

## Step 2 — Initialize the project

- Check NVM is available.
- Decide Node LTS version (e.g. `22`). Create `.nvmrc` with the version string.
- Run `nvm install` then `nvm use`.
- If no `package.json`: `yarn init -y`. Never overwrite existing — extend it.

## Step 3 — Install dependencies

```bash
yarn add -D typescript @types/node eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
# only if user wants direct TS execution:
yarn add -D tsx
```

## Step 4 — Create `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "es2015",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src"]
}
```

The `@` alias resolves to `src/`. Inform the user that some runtimes need `tsconfig-paths` or a bundler alias config to honor this at runtime.

For browser-oriented projects, adjust `module` and `lib` only if asked.

## Step 5 — Create folder and entry file

```
src/index.ts
```

```ts
export function main() {
  console.log("Hello from TypeScript!");
}

if (require.main === module) {
  main();
}
```

Do not introduce `controllers`, `services`, or `modules` — the user designs the architecture.

## Step 6 — Add scripts to `package.json`

```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "lint": "eslint \"src/**/*.{ts,tsx}\""
  }
}
```

If the user wants hot reload: `"dev": "tsx watch src/index.ts"`.

## Step 7 — Add `.eslintrc.cjs`

```js
module.exports = {
  root: true,
  env: { es2021: true, node: true },
  parser: "@typescript-eslint/parser",
  parserOptions: { ecmaVersion: "latest", sourceType: "module" },
  plugins: ["@typescript-eslint"],
  extends: ["eslint:recommended", "plugin:@typescript-eslint/recommended"],
  ignorePatterns: ["dist"],
};
```

## Step 8 — Optional (only on request)

- **Prettier**: add with default config.
- **Tests**: add Vitest or Jest with a basic `tests/` folder.
