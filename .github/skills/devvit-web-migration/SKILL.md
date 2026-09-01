---
name: devvit-web-migration
description: Migrates a singleton-style Devvit app to a Devvit Web app
---

# Migration of Devvit Singleton apps to Devvit Web

This skill migrates singleton-style Devvit apps to the Devvit Web platform, using Hono for routing.

# Step 1 - Set up dependencies

Remove any references to `@devvit/public-api` and `@devvit/protos` from your package.json and code.

Install `@devvit/start` and `@devvit/web` pinned to the latest version.

Add `@hono/node-server`, using ^ notation.

In Dev Dependencies, add `@types/node` (using ^ notation), `devvit` (pinned to the latest version) and `vite` (using ^ notation).

# Step 2 - set up tsconfig and vite

Root-level tsconfig.json should read:

```json
{
  "files": [],
  "references": [
    { "path": "./tools/tsconfig.server.json" },
    { "path": "./tools/tsconfig.vite.json" }
  ]
}
```

Create a tools folder, and inside create the following files:

## tools/tsconfig.base.json

```json
{
  "$schema": "https://json.schemastore.org/tsconfig.json",
  "compilerOptions": {
    "composite": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitOverride": true,
    "noUncheckedIndexedAccess": true,
    "noUncheckedSideEffectImports": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "resolveJsonModule": true,
    "strict": true,
    "types": [],
    "isolatedModules": true,
    "esModuleInterop": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "skipLibCheck": true,
    "skipDefaultLibCheck": true,
    "sourceMap": true,
    "target": "ES2022",
    "lib": ["DOM", "ES2023", "esnext.disposable"],
    "jsx": "react-jsx"
  }
}
```

## tools/tsconfig/server.json

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "outDir": "../dist/types/server",
    "tsBuildInfoFile": "../dist/types/server/tsconfig.tsbuildinfo",
    "customConditions": [],
    "types": ["node", "vitest/globals"],
    "exactOptionalPropertyTypes": false,
    "rootDir": "../src/server"
  },
  "include": ["../src/server/**/*"],
  "exclude": []
}
```

## tools/tsconfig/vite.json

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "outDir": "../dist/types/tools",
    "tsBuildInfoFile": "../dist/types/tools/tsconfig.vite.tsbuildinfo",
    "types": ["node"],
    "rootDir": ".."
  },
  "include": ["../vite.config.ts", "../vitest.config.ts"],
  "exclude": []
}
```

## vite.config.ts

```ts
import { defineConfig } from "vite";
import { devvit } from "@devvit/start/vite";

export default defineConfig({
    plugins: [devvit()],
});
```

# Step 3 - Restructure project

Inside the `src` folder, create the following structure:

```
src/
├── index.ts
├── server/
│   └── core/
│   └── tasks/
│   └── triggers/
│   └── validators/
│   └── index.ts
└── client/
```

The index.ts file should have the following basic structure:

```ts
import { Hono } from "hono";
import { createServer, getServerPort } from "@devvit/web/server";
import { getRequestListener } from "@hono/node-server";

const application = new Hono();

// Triggers

// Settings validators

// Scheduler jobs

const server = createServer(getRequestListener(application.fetch));
server.on("error", (err) => {
    console.error(`server error; ${err.stack}`);
});

const port = getServerPort();
server.listen(port);
```

# Step 4 - Implement trigger and job logic

For each trigger and scheduled job in main.ts, create a file under either triggers/ or tasks/ in the server folder.

For example, if a trigger is defined currently as:

```ts
Devvit.addTrigger({
    event: "PostUpdate",
    onEvent: handleClientPostUpdate,
});
```

or

```ts
Devvit.addTrigger({
    event: "PostUpdate",
    onEvent: async (event, context) => {
      // do some work
    }
});
```


This would be replaced with an onPostUpdate trigger in the triggers/ folder, for example:

```ts
// src/server/triggers/onPostUpdate.ts
import { OnPostUpdateRequest, TriggerResponse } from "@devvit/web/shared";
import { Context } from "hono";

export const handlePostUpdate = async (c: Context) => {
    const now = Date.now();
    const request = await c.req.json<OnPostUpdateRequest>();

    // Do work that is in the directly referenced function, or the body

    return c.json<TriggerResponse>({ message: "post update handled" }, 200);
};
```

Triggers, tasks etc. should be referenced in devvit.json, and appropriate routing code added to the root index.json. E.g.

```json
"triggers": {
    "onPostUpdate": "/internal/triggers/on-post-update"
},
```

and 

```ts
// Triggers
application.post("/internal/triggers/on-post-update", handlePostUpdate);
```

# Step 4 - Move utility code

All  utility code should be moved into the `core/` folder. Keep the basic code structure including subfolders intact.

Export all utility functions from the `core/` folder into each folder's root index.ts file to make them easily accessible throughout the project without creating excessive imports.

# Step 5 - Change references to @devvit/public-api

All references to TriggerContext/JobContext/Context should be removed and replaced with an import to the sub-classes from the BaseContext.

E.g. `context.redis.get` becomes `redis.get` with a direct import from @devvit/web/server, and `context.reddit.getPostById` becomes `reddit.getPostById` with a direct import from @devvit/web/server.

Remove any references to the old context objects or their subclasses from function parameters including code that calls them.