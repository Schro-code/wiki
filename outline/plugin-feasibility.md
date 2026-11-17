# Outline: building four features as plugins

**Question:** how hard is it to add group sync, an approval gate for agents, document metadata and a documentalist agent, while touching Outline's own code as little as possible?

**Short answer:** all four can live in **separate plugin folders that Outline's own files never reference**. Three of them need **zero changes** to Outline's files. The approval gate needs either zero changes (if the agent calls it through a small helper) or a two-line change (if you want it inside Outline's MCP).

> Analysed on 2026-11-10 against Outline 1.12.0 (the cloned repo and the running image). Nothing here has been built or run. The code blocks marked **sketch** are designs, not tested code. The license question (BSL) is separate and still open. See the end of this page.

## 1. How a plugin plugs in (the foundation for all four)

### What Outline does at startup

Outline looks for plugin folders on disk and loads whatever it finds. No list of plugins exists in its code:

```ts
// server/utils/PluginManager.ts  (original code)
public static loadPlugins() {
  const rootDir = env.ENVIRONMENT === "test" ? "" : "build";
  glob
    .sync(path.join(rootDir, "plugins/*/server/index.[jt]s"))
    .forEach((filePath: string) => {
      try {
        require(path.join(process.cwd(), filePath));
      } catch (err) {
        // a broken plugin is logged, the rest of the app still starts
        Logger.error(`Failed to load plugin at ${filePath}`, err);
      }
    });
}
```

A plugin registers itself through **hooks**. These are the server hooks available:

```ts
// server/utils/PluginManager.ts  (original code)
export enum Hook {
  API = "api",                    // add your own /api/... endpoints
  AuthProvider = "authProvider",  // add a login method
  EmailTemplate = "emailTemplate",
  ImportSchema = "importSchema",
  IntegrationSettings = "integrationSettings",
  IssueProvider = "issueProvider",
  Processor = "processor",        // react to events (document published, user signed in...)
  SearchProvider = "searchProvider",
  MentionProvider = "mentionProvider",
  Task = "task",                  // background jobs, including scheduled ones
  UnfurlProvider = "unfurl",
  Uninstall = "uninstall",
  GroupSyncProvider = "groupSyncProvider",
}
```

Plugin endpoints are mounted **before** Outline's own routes:

```ts
// server/routes/api/index.ts  (original code)
// Register plugin API routes before others to allow for overrides
PluginManager.getHooks(Hook.API).forEach((hook) =>
  router.use("/", hook.value.routes())
);
```

### Two ways to ship a plugin

| Way | What you do | Changes to Outline's files | Rebuild the image? | Limits |
|---|---|---|---|---|
| **A. Server plugin as a mounted folder** | Compile the plugin, then mount it into the official image at `build/plugins/<name>/` with a Docker volume | **None** | **No** | Server only, no new screens |
| **B. Custom image** | Put `plugins/<name>/` in the source tree and build your own image | **None** (a new folder, not an edit) | Yes | Needed for any new screen in the browser |

Way A works because the compiled plugins in the official image only use **relative paths** into the server code (checked in the running container):

```js
// build/plugins/oidc/server/index.js  (inside the official image)
const _Logger = _interop_require_default(require("../../../server/logging/Logger"));
const _PluginManager = require("../../../server/utils/PluginManager");
```

A plugin compiled the same way, against the same Outline version, drops into `build/plugins/` and loads.

Browser-side plugins are different. They are found at **build time**, so a new screen always means Way B:

```ts
// app/utils/PluginManager.ts  (original code)
const r = import.meta.glob("../../plugins/*/client/index.{ts,js,tsx,jsx}");
```

And the browser only offers these hooks:

```ts
// app/utils/PluginManager.ts  (original code)
export enum Hook {
  Settings = "settings",          // a new page under Settings
  Imports = "imports",
  Icon = "icon",
  MentionProvider = "mentionProvider",
  InstanceUrl = "instanceUrl",
}
```

There is **no hook to add something to the document page itself**, such as a side panel. A plugin can add a **Settings page**, nothing more.

### The things that are NOT pluggable

| Thing | Where | Why it matters |
|---|---|---|
| **MCP tools** | `server/mcp/index.ts` imports a fixed list of tool files | A plugin cannot add a tool to Outline's `/mcp` without editing this file |
| **Database tables** | `server/migrations/` and `server/models/` (no plugin owns a model today) | A plugin that needs its own table must create it itself, or add files in core folders |
| **Scheduled jobs** | Run only if the `cron` service is on. The default `SERVICES` is `collaboration,websockets,worker,web`, without `cron`. | Configuration, not code: add `cron` to `SERVICES`, or call `/api/cron.hourly` from outside |
| **Scheduling granularity** | `TaskInterval` has only `daily` and `hourly` | A scheduled sync runs at most once an hour |

### Merge safety at a glance

| What you add | Conflicts on upstream merge? |
|---|---|
| A new folder `plugins/<name>/` | **No**, unless Outline adds a plugin with the same name. Use a prefix like `lda-`. |
| A mounted compiled folder (Way A) | **No** merge at all. But it must be **recompiled for each Outline version**. |
| A new file in `server/migrations/` | No text conflict, but it lives in a core folder |
| An edit to an existing Outline file | **Yes, possibly on every upgrade** |

The real upgrade risk is not merge conflicts. Plugins import Outline's **internal** code (models, commands, task base classes), and those can change between versions without notice. See section 7.

## 2. Group sync from Entra ID

### The original code, and why the paid path is closed

The built-in path is: login, then `SyncUserGroupsTask`, then a provider, then `groupsSyncer`. The task stops on the free plan:

```ts
// server/queues/tasks/SyncUserGroupsTask.ts  (original code)
const team = await Team.findByPk(user.teamId);
if (!team || !teamHasEntitlement(team, PlanFeature.GroupSync)) {
  return;   // free plan: nothing happens
}
const externalGroups = await syncProvider.fetchUserGroups(accessToken, settings);
```

So registering a `GroupSyncProvider` is useless on Community. The plugin must run **its own** job and never call this task.

What the plugin can reuse is the normal way Outline adds a member to a group. It's the same call the `groups.add_user` endpoint makes, and it records the event:

```ts
// server/routes/api/groups/groups.ts  (original code, groups.add_user)
const [groupUser, created] = await GroupUser.findOrCreateWithCtx(
  ctx,
  { where: { groupId: group.id, userId: user.id },
    defaults: { createdById: actor.id, permission: permission ?? GroupPermission.Member } },
  { name: "add_user" }
);
// and in groups.remove_user:
await groupUser.destroyWithCtx(ctx, { name: "remove_user" });
```

The Microsoft login plugin already stores each person's Entra object ID, so users can be matched by ID rather than by email:

```ts
// plugins/azure/server/auth/azure.ts  (original code)
providerId: profile.oid,
```

### The plugin design

Two triggers, both in one plugin folder:
- **At login:** a Processor listens to `users.signin` and syncs that one person. This gives login-time behaviour like the paid feature.
- **Every hour:** a scheduled task syncs every mapped group. This catches people who left a group.

```
plugins/lda-entra-groups/
├── plugin.json
└── server/
    ├── index.ts                 ← registers the processor and the task
    ├── env.ts                   ← tenant ID, client ID, secret, mapping file
    ├── graph.ts                 ← tiny Microsoft Graph client
    ├── syncGroup.ts             ← makes one Outline group match one Entra group
    ├── EntraSignInProcessor.ts
    └── EntraGroupSyncTask.ts
```

**Sketch: registration**

```ts
// plugins/lda-entra-groups/server/index.ts  (sketch)
import { PluginManager, Hook } from "@server/utils/PluginManager";
import config from "../plugin.json";
import env from "./env";
import EntraSignInProcessor from "./EntraSignInProcessor";
import EntraGroupSyncTask from "./EntraGroupSyncTask";

if (env.ENTRA_SYNC_CLIENT_ID && env.ENTRA_SYNC_TENANT_ID) {
  PluginManager.add([
    { ...config, type: Hook.Processor, value: EntraSignInProcessor },
    { ...config, type: Hook.Task, value: EntraGroupSyncTask },
  ]);
}
```

**Sketch: making one group match Entra**

```ts
// plugins/lda-entra-groups/server/syncGroup.ts  (sketch)
// mapping: { entraGroupId: "…", outlineName: "Siniestros" }, from a JSON file
export async function syncGroup(ctx, team, actor, mapping, entraMemberOids: string[]) {
  // our own marker; keeps these groups apart from manual ones with the same name
  const [group] = await Group.findOrCreate({
    where: { teamId: team.id, externalId: `entra:${mapping.entraGroupId}` },
    defaults: { name: mapping.outlineName, createdById: actor.id },
    transaction: ctx.state.transaction,
  });

  // Entra object ID → Outline user, via the record the Microsoft login created
  const auths = await UserAuthentication.findAll({
    where: { providerId: entraMemberOids },
    transaction: ctx.state.transaction,
  });
  const wanted = new Set(auths.map((a) => a.userId));

  const current = await GroupUser.findAll({
    where: { groupId: group.id },
    transaction: ctx.state.transaction,
  });

  for (const userId of wanted) {
    await GroupUser.findOrCreateWithCtx(
      ctx,
      { where: { groupId: group.id, userId },
        defaults: { createdById: actor.id, permission: GroupPermission.Member } },
      { name: "add_user" }
    );
  }
  for (const gu of current) {
    if (!wanted.has(gu.userId)) {
      await gu.destroyWithCtx(ctx, { name: "remove_user" });
    }
  }
}
```

**Sketch: the hourly job**

```ts
// plugins/lda-entra-groups/server/EntraGroupSyncTask.ts  (sketch)
export default class EntraGroupSyncTask extends CronTask {
  public get cron() {
    return { interval: TaskInterval.Hour };
  }

  public async perform() {
    const mappings = loadMappings();             // JSON file mounted next to the plugin
    for (const team of await Team.findAll()) {
      const actor = await findSyncUser(team);    // a dedicated technical admin user
      for (const m of mappings) {
        const oids = await graph.groupMemberOids(m.entraGroupId);   // follows paging
        await sequelize.transaction(async (transaction) => {
          await syncGroup(createContext({ user: actor, transaction }), team, actor, m, oids);
        });
      }
    }
  }
}
```

The login processor does the same for one user, with `graph.userGroupIds(oid)` (Graph `transitiveMemberOf`) and only the mappings that match.

### Difficulty and risks

| | |
|---|---|
| **Outline files changed** | **None** |
| **Way to ship** | A (mounted folder). No new screens needed. |
| **Effort** | Small to medium: about 200 to 300 lines plus tests |
| **Needs from IT** | An Entra app registration with an application permission to read group members (such as `GroupMember.Read.All`), granted by an Entra admin. The secret goes in a secrets store. |
| **Configuration** | Add `cron` to `SERVICES` for the hourly job |

Risks:
- People who never logged in don't exist in Outline yet. They join their groups at first login.
- Admins can still edit these groups by hand, and the next sync overwrites it. Tell admins not to touch `entra:` groups.
- Graph throttling and paging at thousands of users.
- I deliberately avoid `ExternalGroup` and `groupsSyncer`, the gated feature's own data and code, to keep a clear line.

## 3. Approval gate for agent writes

### The original code that matters

The MCP tool list is fixed. A plugin cannot add a tool to it:

```ts
// server/mcp/index.ts  (original code)
import { attachmentTools } from "./tools/attachments";
import { collectionTools } from "./tools/collections";
import { commentTools } from "./tools/comments";
import { documentTools } from "./tools/documents";
import { fetchTool } from "./tools/fetch";
import { revisionTools } from "./tools/revisions";
import { templateTools } from "./tools/templates";
import { userTools } from "./tools/users";
```

Processors run **after** an event, so they cannot block a write. The gate therefore has to work as "propose somewhere else, apply on approval," not "intercept the write."

Two pieces of the original code make that safe:

```ts
// server/models/ApiKey.ts  (original code)
// keys can be limited to single endpoints, such as /api/proposals.create
return AuthenticationHelper.canAccess(path, this.scope);
```

```ts
// server/commands/documentUpdater.ts  (original code, its options)
editMode?: TextEditMode;   // replace | append | prepend | patch
findText?: string;         // for patch: the text to replace
lastRevision?: number;     // only update if the document is still at this revision
```

`lastRevision` gives exactly the guard an approval gate needs: if a person edited the document after the agent proposed, the approval is refused instead of overwriting their work.

### The plugin design

```
plugins/lda-proposals/
├── plugin.json
├── server/
│   ├── index.ts           ← registers the API and creates its table at startup
│   ├── Proposal.ts        ← plugin-owned model and table
│   ├── api/proposals.ts   ← proposals.create | list | info | approve | reject
│   └── notify.ts          ← posts a comment on the document: "Agent proposal waiting"
└── client/                ← optional: a Settings page "Agent proposals" (needs Way B)
```

The flow:
1. The agent's key has scopes `read comments:create /api/proposals.create`. **It cannot call `documents.update`.**
2. The agent sends a proposal with the document, the new text or a find-and-replace, the revision it read and a summary.
3. The plugin stores it and posts a comment on the document so people see it.
4. A person with edit rights opens the proposal, sees the difference and approves or rejects it.
5. On approval the plugin applies it through `documentUpdater`, with `lastRevision`, as the approving person. History shows the person, and the proposal record keeps the agent's name.

**Sketch: the endpoints**

```ts
// plugins/lda-proposals/server/api/proposals.ts  (sketch)
router.post("proposals.create", auth(), validate(CreateSchema), transaction(), async (ctx) => {
  const agent = ctx.state.auth.user;
  const document = await Document.findByPk(ctx.input.body.documentId, { userId: agent.id });
  authorize(agent, "read", document);          // must be able to read it, not edit it

  const proposal = await Proposal.create({
    documentId: document.id,
    teamId: agent.teamId,
    proposedById: agent.id,
    baseRevision: ctx.input.body.baseRevision,
    editMode: ctx.input.body.editMode,          // "replace" or "patch"
    findText: ctx.input.body.findText,
    text: ctx.input.body.text,
    summary: ctx.input.body.summary,
    status: "pending",
  }, { transaction: ctx.state.transaction });

  await notify(ctx, document, proposal);       // a comment on the document
  ctx.body = { data: present(proposal) };
});

router.post("proposals.approve", auth(), validate(IdSchema), transaction(), async (ctx) => {
  const reviewer = ctx.state.auth.user;
  const proposal = await Proposal.findByPk(ctx.input.body.id, { lock: true, transaction: ctx.state.transaction });
  const document = await Document.findByPk(proposal.documentId, { userId: reviewer.id });
  authorize(reviewer, "update", document);     // only people who may edit can approve

  await documentUpdater(ctx, {
    document,
    text: proposal.text,
    editMode: proposal.editMode,
    findText: proposal.findText,
    lastRevision: proposal.baseRevision,       // refuse if the document changed meanwhile
  });
  await proposal.update({ status: "approved", reviewedById: reviewer.id }, { transaction: ctx.state.transaction });
  ctx.body = { success: true };
});
```

**Sketch: the plugin's own table, created at startup, with no core migration**

```ts
// plugins/lda-proposals/server/index.ts  (sketch)
sequelize.addModels([Proposal]);
await sequelize.getQueryInterface().createTable("lda_proposals", proposalColumns)
  .catch(ignoreIfExists);
PluginManager.add({ ...config, type: Hook.API, value: router });
```

### How the agent reaches it

The core MCP list is fixed, so there are three options:

| Option | Outline files changed | Effort | Notes |
|---|---|---|---|
| **Claude Code skill that calls the endpoint over HTTP** | None | Small | Simplest. Reads still go through Outline's MCP, proposals through the skill. |
| **The plugin serves its own small MCP endpoint** (for example `/api/proposals.mcp`, using the same MCP library) | None | Medium | The agent configures two MCP servers: Outline and proposals |
| **Register plugin tools in Outline's MCP** | `server/mcp/index.ts`, about two lines | Small | One edit to re-apply on every upgrade. Worth proposing upstream as a hook. |

### Where people review

| Option | Changes | Notes |
|---|---|---|
| **A Settings page "Agent proposals"** with the difference and Approve/Reject buttons | None to Outline's files, but needs Way B (custom image build) | Best experience. Settings is the only screen hook. |
| **The comment on the document links to a review page** served by your own small service | None | Works with Way A. Review happens outside Outline. |
| **Claude Code or a script lists and approves** | None | Fine for a pilot, not for business users |

### Difficulty and risks

| | |
|---|---|
| **Outline files changed** | **None**, or about two lines if you want the tool inside Outline's MCP |
| **Effort** | **Medium to large**. Server part: about 400 to 600 lines. The review screen with a readable difference is the bulk. |
| **Biggest piece** | The review screen (Way B means you build and own a custom image) |

Risks:
- Proposals that replace the whole text are hard to review. Prefer small patch proposals (`patch` mode).
- A plugin-owned table is unusual for Outline. Back it up with the rest of the database, and check it on upgrades.
- If agents also keep `documents:write`, the gate is pointless. The key scopes are the real enforcement.

## 4. Document metadata (owner, status, type, domain)

### The original code: there is no place for it

The document model has no tags, owner or status field. These are its free-form columns:

```ts
// server/models/Document.ts  (original code, selected columns)
@Column(DataType.BOOLEAN) template: boolean;
@Column(DataType.STRING)  icon: string | null;
@Column(DataType.STRING)  color: string | null;
@Column(DataType.JSONB)   sourceMetadata: SourceMetadata | null;   // used by imports
```

Custom fields do exist as a **paid** feature: the closed enterprise plugin's translations include "Data Attributes" and "New attribute". The free code has none of it.

There is also no browser hook for the document page, so a plugin **cannot show a metadata panel on the document** without editing Outline's files.

### Two designs

**Design A (recommended for a pilot): a metadata card inside the document**

Every document starts with a small fixed table that people and agents can read:

```
| Owner | Status | Type | Domain | Reviewed |
|---|---|---|---|---|
| Transformación | approved | operativa | Recibos | 2026-11-10 |
```

A Processor plugin checks it on publish and update, and comments when something is missing or invalid. An endpoint lists documents by metadata for agents and reports.

```ts
// plugins/lda-metadata/server/MetadataProcessor.ts  (sketch)
export default class MetadataProcessor extends BaseProcessor {
  static applicableEvents = ["documents.publish", "documents.update"];

  async perform(event) {
    const document = await Document.findByPk(event.documentId);
    const card = parseCard(DocumentHelper.toMarkdown(document));   // first table in the text
    const problems = validate(card, taxonomy);                     // allowed values from a JSON file
    if (problems.length) {
      await commentAsBot(document, `Metadata card: ${problems.join("; ")}`);
    }
    await MetadataIndex.upsert({ documentId: document.id, ...card }); // optional: for metadata.list
  }
}
```

- Outline files changed: **none**. Ship as Way A.
- Agents read the card through the normal MCP `fetch`. No extra tool is needed.
- Weakness: it's a convention in the text. People can break the table, and the plugin can only complain, not prevent it.

**Design B: a real metadata store**

A plugin-owned table, a `metadata.set/get/list` endpoint and a Settings page to edit it.
- Outline files changed: none, but the Settings page needs Way B.
- Weakness: the metadata is invisible on the document page itself, which defeats much of the purpose.

### Difficulty and risks

| | Design A | Design B |
|---|---|---|
| **Outline files changed** | None | None |
| **Effort** | Small to medium | Medium |
| **Visible on the document** | Yes, it's in the text | No |
| **Enforced** | No, only checked | Yes, in its own store |

Use **collections per domain** for the domain itself. That part needs no code.

## 5. Documentalist agent

### Mostly outside Outline

The agent itself (read, compare, flag contradictions, propose) is a **Claude Code skill plus Outline's existing MCP**. We already tested the pieces in use case 2:
- `fetch` and `list_documents` to read.
- `create_comment` with `anchorText` to flag an exact sentence.
- With section 3: `proposals.create` instead of editing.

### Triggering it automatically

Outline's **webhooks** plugin is free (I found no plan check in it) and already sends events such as `documents.publish` and `documents.update`. So the trigger needs **no new Outline code**:

```
Outline webhook (documents.publish)
  → your small runner service
  → claude -p with the documentalist skill and the agent's own key
  → comments or proposals back into Outline
```

If you prefer to keep everything inside Outline, a Processor plugin can do the same as the webhook:

```ts
// plugins/lda-documentalist/server/DocumentalistProcessor.ts  (sketch)
export default class DocumentalistProcessor extends BaseProcessor {
  static applicableEvents = ["documents.publish"];

  async perform(event) {
    // hand off to the runner; never run the model inside Outline's worker
    await fetch(env.DOCUMENTALIST_RUNNER_URL, {
      method: "POST",
      headers: { authorization: `Bearer ${env.DOCUMENTALIST_RUNNER_TOKEN}` },
      body: JSON.stringify({ documentId: event.documentId }),
    });
  }
}
```

### Difficulty and risks

| | |
|---|---|
| **Outline files changed** | **None** |
| **Effort in code** | Small: a skill, a small runner, the webhook setup |
| **Real effort** | The prompt, the taxonomy and the quality checks. Your Knowledge Studio POC already has much of this (a reader per source, cross-review, warnings), which you could reuse. |

Risks:
- Comment noise if it runs on every small update. Trigger on publish, or debounce.
- The agent invents details (seen in use case 2). Keep a person in the loop.
- Runner keys are powerful. Use a technical account with short-lived keys.

## 6. Summary

| Feature | Outline files changed | Ship as | Effort | Main risk |
|---|---|---|---|---|
| **Group sync** | None | Mounted folder (A) | Small to medium | Entra permission from IT; hourly at most for the scheduled part |
| **Approval gate** | None, or about 2 lines for MCP | A for the server, B for the review screen | **Medium to large** | Building and owning the review screen |
| **Metadata** | None | A (card in the text) or B (own store) | Small to medium | No document-page hook. A card can only be checked, not enforced. |
| **Documentalist** | None | Outside Outline, plus the webhooks plugin | Small in code | Prompt quality and comment noise |

## 7. Upgrades: what will actually break

Merge conflicts are not the problem, since everything lives in new folders. The risk is that plugins import Outline's **internal** code, which has no stability promise:

| Internal piece used | Used by |
|---|---|
| `PluginManager`, `Hook` | All |
| `BaseProcessor`, `CronTask`, `TaskInterval` | Group sync, metadata, documentalist |
| `Group`, `GroupUser`, `UserAuthentication`, `Document`, `Team` models | Group sync, proposals, metadata |
| `documentUpdater`, `createContext`, `auth`, `validate`, `transaction` | Proposals |

How to keep that cheap:
1. **One adapter file per plugin** that holds all imports from Outline. When something moves, you fix one file.
2. **Pin the Outline version** and compile the plugins against that exact tag, ideally in CI: check out the tag, copy the plugins into `plugins/`, build, publish the image or the compiled folder.
3. **A smoke test per plugin** that runs on every upgrade before production.
4. **Prefix plugin names** (`lda-...`) so an upstream plugin never collides.

## 8. Before building

- **License:** these are new files that don't touch the plan checks. The BSL text allows modifying and extending the code. Group sync and custom fields are still paid features in Outline's own offering, so get a written answer from Outline and a legal sign-off first, as discussed.
- **Order I'd suggest:**
  1. Documentalist: no Outline code, quick value.
  2. Metadata card (Design A).
  3. Group sync, once IT grants the Entra permission.
  4. Approval gate, last, after the pilot shows how much supervision agents need.
- **First technical test:** compile an empty "hello" plugin against 1.12.0 and mount it into the running container. That confirms Way A works before anyone writes real code.
