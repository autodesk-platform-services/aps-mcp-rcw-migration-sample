# RCW model migration—implementation document

Port the migration capability of [aps-revit-rcw-migrate-automation](https://github.com/autodesk-platform-services/aps-revit-rcw-migrate-automation) to this MCP server. In an AI client, the user picks a source RCW model and a target folder in natural language. The server runs `SaveAsCloudModel` through Design Automation for Revit and shows live migration progress in an MCP App page.

> **Implementation status (as of the current commit)**: `check-migrator-setup`, `migrate-rcw-model`, `get-migration-status`, `cancel-migration` and `migration.html` are implemented, along with the related `list-folder-contents` (which returns `extensionType`) and `create-folder`. **`migrate-rcw-folder` (§8.3) is not implemented yet.** To migrate a whole folder, the model currently calls `migrate-rcw-model` once for each RCW model. The repo contains plugin bundles for Revit 2025 and 2026 only (`bundles/`); you must build 2027 yourself.

## 1. Confirmed design decisions

| # | Decision | Outcome |
|---|---|---|
| 1 | How to configure the AppBundle and Activity | A one-time script, `npm run setup-da` (run by an administrator), plus the read-only tool `check-migrator-setup`. No write operations are exposed to MCP users |
| 2 | How to get status | Poll the Design Automation workitem status. No webhook, ngrok or socket.io |
| 3 | Revit versions | Multi-version support: 2025, 2026 and 2027 (one AppBundle and one Activity per version) |
| 4 | Progress display | A new MCP App page, `migration.html`, shows migration progress inside the client |

## 2. Mapping of the original repo's features

| Original repo | Purpose | Where it goes in this project |
|---|---|---|
| `RCWMigratorPlugin/Command.cs` | Opens the model in DA, reads `params.json`, calls `doc.SaveAsCloudModel(accountGuid, projectGuid, folderUrn, modelName)` | The C# logic is **unchanged**. The source goes in `plugin/` for reference, and is compiled into a bundle zip for each Revit version |
| `routes/automationConfig.js` | Creates and updates the AppBundle, Activity and aliases | `scripts/setup-da.js` (made idempotent, with multi-version support) |
| `routes/revitAutomation.js` POST | Validates the C4RModel, gets the latest version's storage, submits the workitem | Tools `migrate-rcw-model` and `migrate-rcw-folder` |
| `routes/revitAutomation.js` GET / DELETE | Queries and cancels workitems | Tools `get-migration-status` and `cancel-migration` |
| `/callback/designautomation` + socket.io | Pushes completion notifications to the web page | Removed. The MCP App page polls `get-migration-status` instead |
| `routes/common/aps.js` `getHubFromProjectId` | Walks all hubs to find the target hub | Removed. The model passes the target `hubId` straight from the `list-hubs-projects` result |
| `wwwroot/` front end (jQuery tree) | Selects the source file and target folder | Removed. `list-hubs-projects` and `list-folder-contents` take over |
| `autodesk.forge.designautomation`, `request` | DA SDK, HTTP | Not brought in (both are outdated). The code calls the DA v3 REST API directly with Node's built-in `fetch` and `FormData` |

## 3. Overall architecture

```
AI client ──MCP──► mcp.js (tool registration + UI resource)
                     │
                     ├─ aps.js          User 3-legged token: Data Management (look up versions, validate type, list folders)
                     └─ automation.js   App 2-legged token: Design Automation v3 (submit / query / cancel workitems)
                                           │
                                           ▼
                                DA for Revit (revitcoreconsole + RCWMigrator.dll)
                                           │ SaveAsCloudModel (uses the user's 3-legged token passed with the workitem)
                                           ▼
                                Target ACC / BIM 360 project folder

MCP App page migration.html ──app.callServerTool('get-migration-status')──► mcp.js (polls every 5 seconds)
```

Two tokens:

- **User 3-legged token** (the existing OAuth proxy flow): reads the source model's version information. It is also passed to the workitem as `adsk3LeggedToken`, so Revit reads the source model and writes to the target project as the user.
- **App 2-legged token** (new, scope `code:all`): calls the DA API. It is used only on the server and is never sent to the client or the MCP App page.

## 4. List of file changes

| File | Type | Content |
|---|---|---|
| `automation.js` | New | 2-legged token cache, `daFetch`, engine and Activity lookup, workitem submit/query/cancel, report log tail extraction, job registry |
| `aps.js` | Modified | Extends the OAuth scopes. `getItemTip` also returns `storageId` and `versionType`. `getFolderContents` adds an `extensionType` field |
| `mcp.js` | Modified | Registers 5 new tools and the `migration` UI resource |
| `migration.html` | New | MCP App progress page |
| `scripts/setup-da.js` | New | Idempotent multi-version AppBundle and Activity setup script |
| `bundles/RCWMigratorApp-2025.zip` and others | New | Plugin bundle for each Revit version |
| `plugin/` | New | Plugin C# source (copied from the original repo; 2027 needs adjustments, see §5) |
| `package.json` | Modified | Adds the script `"setup-da": "node --env-file-if-exists=.env scripts/setup-da.js"` |
| `README.md` | Modified | Adds the environment variables and `setup-da` usage |

No new npm dependencies are added.

## 5. Plugin bundles and multi-version support

### 5.1 Current state

The original repo's plugin is built with `Autodesk.Revit.SDK 2026.0.0.9999` and `Autodesk.Forge.DesignAutomation.Revit 2026.0.0`, targeting `net8.0-windows`. `PackageContents.xml` declares `SeriesMin="R2025" SeriesMax="R2026"`. The original repo uses the same zip for both the 2025 and 2026 engines.

### 5.2 Version configuration

Define the version table in one place in `automation.js`, shared by the script and the tools:

```js
export const REVIT_VERSIONS = {
    '2025': { engine: 'Autodesk.Revit+2025', bundleZip: 'RCWMigratorApp-2025.zip' },
    '2026': { engine: 'Autodesk.Revit+2026', bundleZip: 'RCWMigratorApp-2026.zip' },
    '2027': { engine: 'Autodesk.Revit+2027', bundleZip: 'RCWMigratorApp-2027.zip' },
};
export const DEFAULT_REVIT_VERSION = '2027';
```

Naming rules (`<nickname>` defaults to `APS_CLIENT_ID`, `<alias>` defaults to `dev`):

- AppBundle: `RCWMigrator<ver>AppBundle`, full ID `<nickname>.RCWMigrator<ver>AppBundle+<alias>`
- Activity: `RCWMigrator<ver>Activity`, full ID `<nickname>.RCWMigrator<ver>Activity+<alias>`

### 5.3 Plugin bundle for each version

- **2025 / 2026**: use the original repo's `RCWMigratorApp.zip` directly, copied as `RCWMigratorApp-2025.zip` and `RCWMigratorApp-2026.zip`.
- **2027**: must be rebuilt:
  1. Change the NuGet references to `Autodesk.Revit.SDK 2027.*` and `Autodesk.Forge.DesignAutomation.Revit 2027.*`.
  2. Adjust `TargetFramework` to what Revit 2027 requires (**to be confirmed**: the .NET version the Revit 2027 API requires).
  3. Change `PackageContents.xml` to `SeriesMin="R2027" SeriesMax="R2027"`.
  4. Build on Windows and package as `RCWMigratorApp-2027.zip` (the zip root is `RCWMigratorApp.bundle/`).
- **Prerequisite check**: call `GET /da/us-east/v3/engines` and confirm `Autodesk.Revit+2027` is live. When the `setup-da` script hits an engine that doesn't exist, it skips that version with a notice and doesn't stop the other versions.

### 5.4 Version selection rules

- `SaveAsCloudModel` upgrades the model to the Revit version of the engine. It **can't downgrade**. If the target version is lower than the source model's version, the workitem fails.
- The tool parameter `revitVersion` is optional and defaults to `2027`. The tool description must say so, so the model uses the latest version when the user doesn't specify one.
- Before submitting, check that the Activity for the chosen version exists (cache the `check` result for 5 minutes). Otherwise, return a clear error: "The Revit 2027 migrator isn't configured yet. Ask an administrator to run npm run setup-da".

## 6. Authentication changes

### 6.1 User 3-legged scopes (`aps.js`)

```js
const SCOPES = [Scopes.DataRead, Scopes.DataWrite, Scopes.DataCreate, Scopes.AccountRead, Scopes.AccountWrite];
```

This matches the original repo (`data:read data:write account:read account:write`), plus `data:create` to cover creating new items. Once the feature works, remove `account:write` and `account:read` one at a time to check whether they're required, and keep the minimum set.

**Impact**: users who are already signed in must authorize again once (the existing token has no write permission).

### 6.2 App 2-legged token (`automation.js`)

```js
import { AuthenticationClient, Scopes } from '@aps_sdk/authentication';

const authClient = new AuthenticationClient();
let cached = { token: null, expiresAt: 0 };

export async function getAppToken() {
    if (cached.token && cached.expiresAt > Date.now() + 60 * 1000) return cached.token;
    const { access_token, expires_in } = await authClient.getTwoLeggedToken(
        process.env.APS_CLIENT_ID, process.env.APS_CLIENT_SECRET, [Scopes.CodeAll]);
    cached = { token: access_token, expiresAt: Date.now() + expires_in * 1000 };
    return cached.token;
}
```

### 6.3 ACC prerequisite

The APS app's Client ID must be added as a Custom Integration in **both the source and target** ACC / BIM 360 accounts. Otherwise, both the Data Management calls and `SaveAsCloudModel` fail.

## 7. Design Automation setup script (`scripts/setup-da.js`)

Usage:

```bash
npm run setup-da                       # Set up all versions
npm run setup-da -- --versions 2026,2027
```

The flow for each version (idempotent, safe to run repeatedly):

1. Query `GET /appbundles`:
   - Doesn't exist: `POST /appbundles` → `POST /appbundles/{id}/aliases` (the alias points to v1)
   - Exists: `POST /appbundles/{id}/versions` → `PATCH /appbundles/{id}/aliases/{alias}` (points to the new version)
2. Upload the zip with the returned `uploadParameters`: `multipart/form-data`, with all the `formData` fields first and the **`file` field last** (S3 requires this). Use Node's built-in `FormData` and `Blob`.
3. Query `GET /activities`:
   - Doesn't exist: `POST /activities` → `POST /activities/{id}/aliases`
   - Exists: `POST /activities/{id}/versions` → `PATCH` the alias (unlike the original repo, which skips an existing Activity, so changes to `commandLine` never took effect)
4. Print a summary table of the result for each version.

Activity definition:

```js
{
    id: `RCWMigrator${ver}Activity`,
    engine: REVIT_VERSIONS[ver].engine,
    appbundles: [`${nickname}.RCWMigrator${ver}AppBundle+${alias}`],
    commandLine: [`$(engine.path)\\\\revitcoreconsole.exe /i "$(args[rvtFile].path)" /al "$(appbundles[RCWMigrator${ver}AppBundle].path)"`],
    parameters: {
        rvtFile:     { verb: 'get', description: 'Input Revit cloud model', required: false },
        inputParams: { verb: 'get', description: 'Target location JSON', localName: 'params.json', required: false },
    },
    description: `Migrate Revit Cloud Worksharing model (Revit ${ver})`,
}
```

DA base URL: `https://developer.api.autodesk.com/da/us-east/v3`.

## 8. MCP tool design

All tools follow the existing style in `mcp.js`: `server.registerTool(name, { title, description, inputSchema: z.object(...), annotations }, handler)`.

### 8.1 `check-migrator-setup` (read-only)

- Input: none
- Logic: query the engine list and this app's Activity list, and report the status of each version
- Output:

```json
[
  { "revitVersion": "2025", "engineAvailable": true,  "activityReady": true },
  { "revitVersion": "2027", "engineAvailable": true,  "activityReady": false }
]
```

### 8.2 `migrate-rcw-model`

| Parameter | Type | Description |
|---|---|---|
| `sourceProjectId` | string | Source project ID (`b.` prefix) |
| `sourceItemId` | string | Source RCW model item ID |
| `targetHubId` | string | Target hub ID (`b.` prefix) |
| `targetProjectId` | string | Target project ID |
| `targetFolderId` | string | Target folder URN |
| `targetName` | string? | Target model name; defaults to the source file name |
| `revitVersion` | enum `2025`/`2026`/`2027`? | Defaults to `2027` |

Processing flow:

1. `getItemTip(sourceProjectId, sourceItemId)` gets the latest version.
2. Check that `versionType === 'versions:autodesk.bim360:C4RModel'`. Otherwise, return the error "Only Revit Cloud Worksharing models are supported".
3. Check that the Activity for `revitVersion` is ready (§5.4).
4. Assemble the workitem (keeping the parameter format of the original repo):

```js
const params = {
    TargetAccountGuid: targetHubId.replace(/^b\./, ''),
    TargetProjectGuid: targetProjectId.replace(/^b\./, ''),
    TargetFolderUrn: targetFolderId,
    TargetModelName: targetName ?? tip.name,
};
const userToken = await authenticationProvider.getAccessToken();
const workitem = {
    activityId: `${nickname}.RCWMigrator${revitVersion}Activity+${alias}`,
    arguments: {
        rvtFile: { url: tip.storageId, headers: { Authorization: `Bearer ${userToken}` } },
        inputParams: { url: 'data:application/json,' + JSON.stringify(params) },
        adsk3LeggedToken: userToken,
    },
};
```

> To embed the data URL, the original repo replaces double quotes with single quotes. During implementation, test with standard JSON (encoded with `encodeURIComponent`) first. If the plugin fails to parse it, fall back to the original repo's single-quote form.

5. `POST /workitems`, and write the job to the job registry (§9).
6. Return `structuredContent` (for the UI to render) and a short text:

```json
{
  "jobs": [{
    "workItemId": "abc123",
    "fileName": "Tower.rvt",
    "revitVersion": "2027",
    "target": { "projectId": "b.xxx", "folderId": "urn:..." },
    "status": "pending",
    "submittedAt": "2026-10-14T10:00:00Z"
  }]
}
```

- `_meta: { ui: { resourceUri: 'ui://aps-mcp/migration.html' } }`
- `annotations: { readOnlyHint: false, destructiveHint: false, idempotentHint: false }`

### 8.3 `migrate-rcw-folder` (in design, not implemented yet)

- Input: `sourceProjectId`, `sourceFolderId`, plus the target parameters from §8.2 and `revitVersion` (no `targetName`)
- Logic: list the source folder (**not recursive**) and keep items with `extensionType === 'items:autodesk.bim360:C4RModel'`, **up to 5** (the original repo's limit is kept; if there are more, return an error with the file count so the user can narrow the scope). Reuse the submission logic from §8.2 for each one.
- Output: the same structure as §8.2, with multiple entries in `jobs`, plus `skipped` (the list of non-RCW file names)
- Also bound to `migration.html`

### 8.4 `get-migration-status` (read-only)

- Input: `workItemIds: string[]`
- Logic: check that each ID belongs to the current user (§9), then call `GET /workitems/{id}`
- On a failure status (`failed*`), download `reportUrl` and put the **last 40 lines** in `reportTail`, so the model can explain the cause of the failure
- Output:

```json
{
  "jobs": [{
    "workItemId": "abc123",
    "fileName": "Tower.rvt",
    "status": "failedInstructions",
    "progress": null,
    "reportTail": "...Failed to save cloud model due to: ..."
  }]
}
```

- Callable by both the model and the MCP App page: `_meta: { ui: { visibility: ['model', 'app'] } }`

DA status values: `pending`, `inprogress`, `success`, `cancelled`, `failedDownload`, `failedInstructions`, `failedUpload`, `failedUploadOptional`, `failedLimitDataSize`, `failedLimitProcessingTime`. A terminal state is any value other than `pending` or `inprogress`.

### 8.5 `cancel-migration`

- Input: `workItemId`
- Logic: after checking ownership, `DELETE /workitems/{id}`, and update the registry status to `cancelled`
- Note: a task that has already reached `inprogress` may not be cancellable; say so in the returned message
- `annotations: { destructiveHint: true }`; `_meta.ui.visibility: ['model', 'app']` (the Cancel button on the page calls it)

### 8.6 Changes to existing tools

`create-folder` is also new (`projectId`, `parentFolderId`, `folderPath`). It creates a folder or a nested path in the target project, reusing levels that already exist, and is called only when the user explicitly asks. Use it to build the target directory structure before a migration.

`list-folder-contents` adds `extensionType` (`item.attributes.extension.type`) to each entry. The model uses it to recognize RCW models (`items:autodesk.bim360:C4RModel`).

## 9. Job registry and security

`automation.js` keeps an in-memory `Map<workItemId, Job>`:

```js
{ workItemId, ownerKey, fileName, revitVersion, source, target, status, submittedAt, finishedAt }
```

- **Ownership check**: `ownerKey` is the current MCP session's `clientId` plus the APS user ID (fetched once with `getUserInfo` and cached). `get-migration-status` and `cancel-migration` only operate on the user's own jobs, so user A can't query or cancel user B's tasks.
- The registry is lost when the server restarts. For an unknown ID, return "Job not found, or the server was restarted". Do **not** fall back to a query without the ownership check.
- Terminal jobs are kept for 24 hours and then cleaned up.
- Neither the 2-legged token nor the user token appears in any tool return value or in the UI.

## 10. MCP App progress page (`migration.html`)

### 10.1 Resource registration (`mcp.js`)

Follow the existing viewer resource:

```js
const MIGRATION_RESOURCE_URI = 'ui://aps-mcp/migration.html';
server.registerResource('migration', MIGRATION_RESOURCE_URI, { mimeType: 'text/html;profile=mcp-app' }, async (uri) => ({
    contents: [{
        uri: uri.toString(),
        mimeType: 'text/html;profile=mcp-app',
        text: MIGRATION_HTML,
        _meta: { ui: { domain: publicUrl, csp: { resourceDomains: ['https://cdn.jsdelivr.net'], connectDomains: [] } } },
    }],
}));
```

The page doesn't access any APS domain directly. All data comes through `callServerTool`, so `connectDomains` is empty.

### 10.2 Page behavior

```
┌──────────────────────────────────────────────────────────┐
│ RCW model migration → Revit 2027        2/3 complete     │
├──────────────────────────────────────────────────────────┤
│ Tower.rvt        ● Succeeded     00:03:12                │
│ Podium.rvt       ◐ In progress   00:01:40      [Cancel]  │
│ Site.rvt         ✕ Failed        failedInstructions [Details] │
└──────────────────────────────────────────────────────────┘
```

1. `app.ontoolresult` receives the `structuredContent.jobs` returned by `migrate-rcw-model` and `migrate-rcw-folder`, and renders the list.
2. Every 5 seconds, call `app.callServerTool({ name: 'get-migration-status', arguments: { workItemIds } })`, polling only jobs that haven't reached a terminal state. Stop once all jobs are terminal.
3. Status display: `pending` = queued, `inprogress` = in progress, `success` = succeeded, `cancelled` = cancelled, `failed*` = failed (shows the specific status code). Each row shows the elapsed time.
4. The Details button on a failed row expands `reportTail`.
5. Rows in `pending` or `inprogress` show a Cancel button → `callServerTool('cancel-migration')`.
6. When all jobs finish, call `app.updateModelContext()` and write a summary text (for example, "2 of 3 models succeeded; Site.rvt failed: failedInstructions"), so the model can answer follow-up questions directly.
7. If polling fails (network error), back off exponentially (5s → 10s → 20s, capped at 60s) and show a notice at the top of the page without stopping.
8. Keep the styling simple and suited to both the light and dark client themes (use `prefers-color-scheme`).

## 11. Environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `APS_CLIENT_ID` / `APS_CLIENT_SECRET` | Yes | — | Existing |
| `APS_DA_NICKNAME` | No | `APS_CLIENT_ID` | Use when the APS app has a DA nickname set |
| `APS_DA_ALIAS` | No | `dev` | Alias for the AppBundle and Activity |
| `APS_DA_REGION` | No | `us-east` | DA region |

The original repo's `APS_WEBHOOK_URL`, `APS_CALLBACK_URL` and `SERVER_SESSION_SECRET` are no longer needed.

## 12. Implementation steps

| Phase | Content | Done when |
|---|---|---|
| 1 | Copy the plugin source and the 2025/2026 zips; implement the basics of `automation.js` (token, `daFetch`) and `setup-da.js` | `npm run setup-da -- --versions 2026` succeeds; the bundle and activity are visible in DA |
| 2 | Implement `check-migrator-setup` | The tool returns the correct status for each version |
| 3 | Extend the scopes; add `storageId` / `versionType` / `extensionType` to `aps.js` | After signing in again, `list-folder-contents` recognizes RCW models |
| 4 | Implement `migrate-rcw-model`, `get-migration-status` and `cancel-migration` (text-only at first) | One model migrates end to end, and the new model appears in the target folder |
| 5 | Implement `migration.html` and bind it to the migration tools | Live progress shows in the client, the Cancel button works, and the model can read the summary when it finishes |
| 6 | Implement `migrate-rcw-folder` | 2–5 models migrate in one go, and the progress page shows multiple rows correctly |
| 7 | Build the 2027 plugin bundle, run `setup-da -- --versions 2027` | 2027 migration succeeds end to end |
| 8 | Trim the scopes, update the README | All use cases pass with the minimum scope set |

## 13. Test cases

Test environment: DAS - Canada hub → `AU 2026 RCW Demo Project` → `Project Files` (source: `Sample -001`, target: `AU 2026 Demo`).

| # | Scenario | Expected |
|---|---|---|
| 1 | Migrate one RCW model to another folder in the same project (2026) | `success`, and a model with the same name appears in the target folder |
| 2 | Migrate across projects (same hub) | `success` |
| 3 | Default version (no `revitVersion`) | Uses 2027 |
| 4 | Source is a regular RVT or PDF | Rejected before submission, with a message that only RCW is supported |
| 5 | Target version is lower than the source model's version | The workitem fails, `reportTail` contains the cause, and the model can explain it |
| 6 | The user has no write permission on the target folder | The workitem fails with a readable error message |
| 7 | Activity for the target version isn't configured | Rejected before submission, with a message to run `setup-da` |
| 8 | Folder migration with 3 RCW and 2 other files | 3 are submitted, and `skipped` lists the 2 |
| 9 | More than 5 RCW models in the folder | Rejected with a message |
| 10 | Cancel while `pending` | The status changes to `cancelled`, and the page stays in sync |
| 11 | Query or cancel someone else's workItemId from another account | Rejected |
| 12 | Query an old workItemId after a server restart | Returns "Job not found, or the server was restarted" |
| 13 | Run `setup-da` repeatedly | Creates a new version and updates the alias, with no errors |

## 14. Risks and open items

| Item | Description | Response |
|---|---|---|
| Revit 2027 engine and .NET version | Need to confirm DA offers `Autodesk.Revit+2027`, and the target framework of the 2027 plugin | Before phase 7, query `/engines` and check the Revit 2027 API documentation |
| User token lifetime is about 1 hour | The token is passed in as `adsk3LeggedToken`, so a large model that queues and processes for more than 1 hour may fail | Get a fresh token at submission; mention the risk for large models in the tool description; later, evaluate DA's support for token refresh |
| DA billing | Each workitem consumes APS Flex tokens | Keep the limit of 5 files per call; note it in the README |
| In-memory registry | Lost on restart, and not shared across multiple instances | Acceptable at workshop scale; use persistent storage for a production deployment |
| `inputParams` data URL format | Compatibility of standard JSON with the original repo's single-quote form | Confirm by testing in phase 4 |
| Linked models | The original repo explicitly doesn't support migrating linked Revit models | State this in the tool description |
