# Setup

Setup bellow prompts.

## Install dependencies

```bash
pnpm add -D husky
pnpm add -D @biomejs/biome
pnpm add -D vitest
pnpm add -D vite-tsconfig-paths
```

## Run scripts

```bash
pnpm exec husky init
```

## Write setting files

### ./package.json

Please add the following scripts to this file:

```json
{
  "packageManager": "pnpm@10.13.1"
}
```

```json
{
  "scripts": {
    "lint": "biome check .",
    "test": "vitest run --passWithNoTests"
  }
}
```

### ./biome.json

```json
{
  "vcs": {
    "enabled": true,
    "clientKind": "git"
  },
  "assist": {
    "actions": {
      "source": {
        "organizeImports": {
          "level": "on",
          "options": {
            "groups": [
              ":NODE:",
              ":PACKAGE:",
              ":BLANK_LINE:",
              "@/_lib/**",
              ":BLANK_LINE:",
              "@/_components/ui/**",
              ":BLANK_LINE:",
              "@/_schemas/**",
              "@/_services/**",
              "@/_actions/**",
              "@/_components/domain/**"
            ]
          }
        }
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "nursery": {
        "useSortedClasses": "error"
      }
    }
  },
  "files": {
    "includes": [
      "**",
      "!.next/**",
      "!node_modules/**",
      "!src/_components/ui/**",
      "!src/generated/prisma/**"
    ]
  }
}
```

### ./.vscode/settings.json

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": "explicit"
  }
}
```

### ./.vscode/extensions.json

```json
{
  "recommendations": [
    "biomejs.biome",
    "bradlc.vscode-tailwindcss",
    "GitHub.copilot"
  ]
}
```

### ./vitest.config.ts

```typescript
import tsconfigPaths from "vite-tsconfig-paths";
import { defineConfig } from "vitest/config";

export default defineConfig({
  plugins: [tsconfigPaths()],
});
```

### ./src/example.test.ts

```typescript
import { expect, test } from "vitest";

import { sum } from "./example";

test("adds 1 + 2 to equal 3", () => {
  expect(sum({ a: 1, b: 2 })).toBe(3);
});
```

### ./src/example.ts

```typescript
export function sum({ a, b }: { a: number; b: number }): number {
  return a + b;
}
```

### ./.husky/pre-commit

```bash
pnpm lint
pnpm test
```

### ./.github/workflows/ci.yaml

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          run_install: false
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm -r install
      - run: pnpm -r build
      - run: pnpm -r lint
      - run: pnpm -r test
```

## Format

```bash
pnpm exec biome format --write
```

## Remove

```bash
rm -rf src/app/favicon.ico
rm -rf src/app/page.tsx
rm -rf public/*.svg
```
