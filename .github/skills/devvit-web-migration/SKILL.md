---
name: devvit-web-migration
description: Migrates a singleton-style Devvit app to a Devvit Web app
---

# Migration of Devvit Singleton apps to Devvit Web

This skill migrates singleton-style Devvit apps to the Devvit Web platform, using Hono for routing.

Tips - Make use of the @devvit/mcp MCP server for quick assistance understanding Devvit Web. Use `devvit_search` to quickly find relevant documentation and examples.

# Step 1 - Set up dependencies

Remove any references to `@devvit/public-api` and `@devvit/protos` from your package.json and code.

Install `@devvit/start` and `@devvit/web` pinned to the latest version.

Add `hono` and `@hono/node-server`, both using ^ notation.

In Dev Dependencies, add `@types/node` (using ^ notation), `devvit` (pinned to the latest version) and `vite` (pinned to `^7.3.5` or higher — `@devvit/start` has a peer requirement of `vite>=7.3.5`).

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

## tools/tsconfig.server.json

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

## tools/tsconfig.vite.json

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
├── server/
│   └── core/
│   └── forms/
│   └── menus/
│   └── tasks/
│   └── triggers/
│   └── validators/
│   └── index.ts
```

The index.ts file in `src/server` should have the following basic structure:

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

The devvit.json file should use top level keys instead of `blocks`, example below:

```json
{
    "$schema": "https://developers.reddit.com/schema/config-file.v1.json",
    "name": "your-app-name",
    "permissions": {
        "reddit": true
    },
    "scripts": {
        "build": "vite build",
        "dev": "vite build --watch"
    },
    "server": {
        "dir": "dist/server",
        "entry": "index.cjs"
    },
    "triggers": {
        "onAppInstall": "/internal/triggers/on-app-install",
        "onAppUpgrade": "/internal/triggers/on-app-upgrade",
        "onModAction": "/internal/triggers/on-mod-action"
    },
    "scheduler": {
        "tasks": {
            "myJob": "/internal/tasks/my-job"
        }
    },
    "menu": {
        "items": [
            {
                "label": "Submit Leaderboard Post",
                "forUserType": "moderator",
                "location": "subreddit",
                "endpoint": "/internal/menu/submit-leaderboard-post"
            }
        ]
    },
    "forms": {
        "manualSetPointsForm": "/internal/form/set-score-manually"
    },    
    "settings": {
        "subreddit": {
            "rules": {
                "type": "paragraph",
                "label": "Automod Neo Rules",
                "helpText": "Paste or enter YAML rules here. Most existing AutoModerator rules will work without modification.",
                "validationEndpoint": "/internal/validators/validate-automod-setting"
            }
        },
        "app": {
            "someAppScopedSetting": {
                "type": "string",
                "label": "Some App Setting",
                "helpText": "Enter the value for some app setting.",
                "validationEndpoint": "/internal/validators/validate-app-setting"
            }
        }
    }
}
```

# Step 4 - Migrate permissions

If there is a call to Devvit.configure() in main.ts, add equivalent permissions to the `permissions` section of devvit.json.

# Step 5 - Implement trigger and job logic

For each trigger and scheduled job in main.ts, create a file under either triggers/ or tasks/ in the server folder.

Trigger request types (e.g. `OnPostUpdateRequest`, `OnModActionRequest`, `OnAppInstallRequest`) and `TriggerResponse` are exported from `@devvit/web/shared`. If a trigger handler does not need the request body, there is no need to read it.

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
import type { OnPostUpdateRequest, TriggerResponse } from "@devvit/web/shared";
import type { Context } from "hono";

export const handlePostUpdate = async (c: Context) => {
    const request = await c.req.json<OnPostUpdateRequest>();

    // Do work that is in the directly referenced function, or the body

    return c.json<TriggerResponse>({ message: "post update handled" }, 200);
};
```

For scheduled jobs, use `TaskRequest<T>` and `TaskResponse` which are exported from `@devvit/web/server`. The generic type parameter `T` is the shape of the job's `data` field:

```ts
// src/server/tasks/myJob.ts
import type { TaskRequest, TaskResponse } from "@devvit/web/server";
import type { Context } from "hono";

export const handleMyJob = async (c: Context) => {
    const request = await c.req.json<TaskRequest<{ someParam?: string } | undefined>>();
    const someParam = request.data?.someParam;

    // Do work

    return c.json<TaskResponse>({ message: "job handled" }, 200);
};
```

Triggers, tasks etc. should be referenced in devvit.json (under the `web.triggers` and `web.scheduler` maps), and appropriate routing code added to `src/index.ts`. E.g.

```json
"triggers": {
    "onPostUpdate": "/internal/triggers/on-post-update"
},
"scheduler": {
    "tasks": {
        "myJob": {
            "endpoint": "/internal/tasks/my-job"
        },
        "myCronJob": {
            "endpoint": "/internal/tasks/my-cron-job",
            "cron": "0 0 * * *"
        }
    }
}
```

and

```ts
// Triggers
application.post("/internal/triggers/on-post-update", handlePostUpdate);

// Scheduler jobs
application.post("/internal/tasks/my-job", handleMyJob);
```

Where a job is scheduled to a fixed cron in an appInstall handler, define the cron inside devvit.json rather than using manual scheduling code. If a cron is defined dymanically at runtime (e.g. using a random factor), do not define a cron inside devvit.json but instead use an onAppInstall/onAppUpgrade trigger as before.

If there are more than one Devvit.addTrigger that implements the same trigger point, you need to implement **only one** trigger, but have it call both things internally.

# Step 6 - Migrate menu item code

For each instance of Devvit.addMenuItem in main.ts, create a file within the `src/server/menus` folder. 

Example code: 

```ts
import { MenuItemRequest, UiResponse, T1 } from "@devvit/web/shared";
import type { Context } from "hono";

export const handleSetScoreManuallyMenu = async (c: Context) => {
    const menuRequest = await c.req.json<MenuItemRequest>();

    // Do something

    return c.json<UiResponse>({
        showToast: "Action completed.",
    });
};
```

If you need to show a form, return a `UiResponse` with a `showForm` field instead of `showToast` e.g.:

```ts
return c.json<UiResponse>({
    showForm: {
        name: "manualSetPointsForm",
        form: {
            title: `Manually set points for ${comment.authorName}`,
            fields: [
                {
                    name: "newScore",
                    type: "number",
                    defaultValue: currentScore.score,
                    label: `Enter a new score for ${comment.authorName}`,
                    helpText: "Warning: This will overwrite the score that currently exists",
                    required: true,
                },
            ],
        },
    },
});
```

Wire the menu item handlers in `src/server/index.ts` and devvit.json by adding routes for each handler. For example:

```ts
// Menus
application.post("/internal/menu/submit-leaderboard-post", handleSubmitLeaderboardPostMenu);
```

# Step 7 - Form handlers

For each form declared in main.ts, you need to create a file in the `src/server/forms` folder to handle that form, and wire it in `src/server/index.ts` and devvit.json similarly to menu items, triggers and tasks. Declare an interface that matches the form structure for type safety. Example code:

```ts
interface SetScoreManuallyFormValues {
    newScore: number;
}

export const handleSetScoreManuallyForm = async (c: Context) => {
    const { newScore } = await c.req.json<SetScoreManuallyFormValues>();
    if (!context.commentId) {
        return c.json<UiResponse>({
            showToast: "No comment selected.",
        });
    }

    // Do something with the new score

    return c.json<UiResponse>({
        showToast: "Score updated successfully.",
    });
};
```    

# Step 8 - Migrate settings defined in main.ts

Settings should be migrated from the old context-based approach to the new `settings` service from `@devvit/web/server`.

Look for calls to `Devvit.addSettings()` in main.ts and migrate them to devvit.json, preserving the grouping structure if any exists. Use the same names as the scheduled job names for the keys in the JSON file.

Any validation code in the original Devvit.addSettings call should be moved to new validation endpoints in the `src/server/validators` folder. Example settings validator:

```ts
import { SettingsValidationRequest, SettingsValidationResponse } from "@devvit/web/shared";
import { Context } from "hono";
import { parseWebhookUrl } from "../core";

export const validateDiscordOrSlackWebhook = async (c: Context) => {
    const validationRequest = await c.req.json<SettingsValidationRequest<string>>();

    if (!validationRequest.value) {
        return c.json<SettingsValidationResponse>({
            success: true,
        });
    }

    if (!parseWebhookUrl(validationRequest.value)) {
        return c.json<SettingsValidationResponse>({
            success: false,
            error: "Invalid Discord or Slack webhook URL format.",
        });
    }

    return c.json<SettingsValidationResponse>({ success: true }, 200);
};
```

# Step 9 - Move utility code

All utility code should be moved into the `core/` folder. Keep the basic code structure including subfolders intact.

Export all utility functions from the `core/` folder into each folder's root index.ts file to make them easily accessible throughout the project without creating excessive imports.

# Step 10 - Change references to @devvit/public-api

All references to `TriggerContext`/`JobContext`/`Context` should be removed and replaced with direct service imports from `@devvit/web/server`.

For example, `context.redis.get` becomes `redis.get` with `import { redis } from "@devvit/web/server"`, and `context.reddit.getPostById` becomes `reddit.getPostById` with `import { reddit } from "@devvit/web/server"`.

The available services exported from `@devvit/web/server` include: `redis`, `reddit`, `scheduler`, `settings`, `realtime`, `media`, `notifications`, `payments`, `cache`.

**`subredditName` is not a direct named export.** Access it via the request context proxy: `import { context } from "@devvit/web/server"` and use `context.subredditName`. The `context` object also exposes `context.appVersion`, `context.appSlug`, `context.userId`, `postId`, `commentId`, and other fields from `BaseContext`.

**Reddit ID type guards:** `isCommentId`/`isLinkId` from `@devvit/public-api/types/tid.js` are replaced by `isT1`/`isT3` from `@devvit/web/shared`. These are proper TypeScript type predicates (`id is T1`, `id is T3`) and should be used for narrowing when calling methods like `reddit.remove(id, isSpam)` that require branded ID types.

**Third-party packages built against `@devvit/public-api`:** Any helper libraries that import from `@devvit/public-api` (e.g. packages that accept `RedditAPIClient` or `RedisClient` from the old API) will not be type-compatible with the new service objects. Remove these from `package.json` and replace their functionality inline using the new `reddit`/`redis` imports. In particular:

* `devvit-helpers` imports from @devvit/public-api. Remove it from `package.json` and replace its functionality using the new `reddit`/`redis`/`scheduler` service objects.
* `@fsvreddit/fsv-devvit-helpers` has equivalent functionality in `@fsvreddit/fsv-devvit-web-helpers` in most cases. If an identically named function exists in that package (just without passing in a Context or RedisAPIClient), use it. Otherwise replace functionality inline.

Remove any references to the old context objects or their subclasses from function parameters including code that calls them.

# Step 11 - Add scripts to package.json

Add the following scripts if they do not yet exist:

```json
  "scripts": {
    "build": "vite build",
    "deploy": "npm run type-check && npm run lint && npm run test && devvit upload",
    "dev": "devvit playtest",
    "launch": "npm run deploy && devvit publish",
    "lint": "eslint \"src/**/*.{ts,tsx}\"",
    "login": "devvit login",
    "test": "vitest run --config vitest.config.ts",
    "type-check": "tsc --build"
  },
```
