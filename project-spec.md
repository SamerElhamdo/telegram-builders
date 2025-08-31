تمام — هذا **برومبت موحّد، جاهز للّصق في Codex/Cursor** لينفّذ المشروع كاملًا (واجهة + باك-إند) وفق مواصفات إنتاجية، مع عقود بيانات واضحة، نماذج، بنية مجلدات، واختبارات.
انسخه كما هو:

---

## SYSTEM

You are a senior full-stack engineer. Build a production-ready **Visual Telegram Bot Builder** consisting of:

* **Frontend**: React + TypeScript visual graph editor that renders Telegram-like message blocks and keyboards, saves the graph as JSON, runs a local simulator, and talks to the backend via REST.
* **Backend**: Django + DRF + aiogram v3 runtime that executes the published graph as a finite-state machine (FSM) via Telegram webhooks, uses Postgres, Redis, Celery, and is containerized with Docker Compose.

Deliver runnable code, docs, tests, and seed data. Follow the contracts below exactly.

---

## GLOBAL QUALITY BAR

* Lint/format: ESLint + Prettier (FE), ruff/black (BE).
* Typed everywhere (TypeScript types; Python type hints).
* ENV via `.env` / `.env.example`. No secrets in repo.
* OpenAPI docs at `/api/schema/` + Swagger UI `/api/docs/`.
* CI-like script (Makefile) to run dev stack, tests, and migrations.
* Clear README with start commands and troubleshooting.

---

## SHARED DOMAIN CONTRACTS

### Graph JSON (authoritative)

```ts
type NodeType = 'Start'|'Text'|'Media'|'Question'|'Condition'|'Action'|'Delay'|'End';

type NodeBase = {
  id: string;
  type: NodeType;
  position: { x: number; y: number };
  data: Record<string, any>;
};

type Edge = {
  id: string;
  source: string;
  target: string;
  data?: {
    label?: string;        // description on line/arrow
    condition?: string;    // expr (ctx/input) or equals to button text/key
  };
};

type GraphJSON = {
  nodes: NodeBase[];
  edges: Edge[];
  meta?: { title?: string; locale?: 'ar'|'en'; version?: number; };
};
```

### Node `data` payloads

* **Start**: `{ label?: string }`
* **Text**: `{ text: string; parse_mode?: 'Markdown'|'HTML'; keyboard?: KeyboardSpec }`
* **Media**: `{ kind:'photo'|'video'|'document'|'audio'; url:string; caption?:string; keyboard?: KeyboardSpec }`
* **Question**: `{ prompt:string; var:string; buttons: ButtonSpec[]; layout?: LayoutSpec }`
* **Condition**: `{ expr: string }` // safe expression using `ctx` and `input`
* **Action**: `{ url:string; method?:'GET'|'POST'; headers?:Record<string,string>; payload?:any; save_to?:string; }`
* **Delay**: `{ ms:number }`
* **End**: `{ label?:string }`

```ts
type ButtonSpec = { text:string; value?:string; callback_data?:string };
type LayoutSpec = { rows?:number; cols?:number; wide?:boolean };
type KeyboardSpec = {
  buttons: ButtonSpec[];
  layout?: LayoutSpec; // rows/cols/wide like Telegram inline keyboard
};
```

### REST API (must match)

Base URL: `${BASE_URL}`

* `GET /api/projects` → `Project[]`

```ts
type Project = { id:string; title:string; bot_id:string; created_at:string; last_version_id?:string };
```

* `POST /api/projects` body `{ title:string; bot_id:string }` → `Project`
* `GET /api/projects/:id/versions` → `GraphVersionSummary[]`

```ts
type GraphVersionSummary = { id:string; is_published:boolean; created_at:string };
```

* `GET /api/versions/:id` → `GraphVersion`

```ts
type GraphVersion = { id:string; is_published:boolean; created_at:string; graph_json:GraphJSON };
```

* `POST /api/projects/:id/versions` body `{ graph_json:GraphJSON }` → `GraphVersion`
* `POST /api/versions/:id/publish` → `{ ok:true, version_id:string, published_at:string }`
* `GET /api/bots` → `BotSummary[]`

```ts
type BotSummary = { id:string; name:string; is_active:boolean };
```

* **Webhook**: `POST /tg/webhook/:bot_uuid/` (Telegram → backend)
* **Auth**: Support `Authorization: Bearer <token>` (optional in dev). Add CORS config.

### Validation rules (server & client)

* Exactly one `Start` node.
* All edges connect existing nodes; no dangling nodes.
* Each `Question` has edges covering all buttons (by text/value).
* `Condition.expr` must be parseable (no raw eval).
* Detect infinite cycles without exit where applicable; raise on publish.

---

## PHASE A — FRONTEND (React)

### Stack

* React 18 + TypeScript + Vite
* **React Flow** for graph editing
* **Zustand** for editor state
* **TanStack Query** for API
* **Zod** + **React Hook Form** for node forms
* TailwindCSS; dark/light; lucide-react icons
* i18n (react-i18next) with `ar` default, `en` optional
* Vitest + Playwright (smoke E2E for graph save/publish)

### Features

1. **Projects Dashboard**: create/open/rename; shows publish status.
2. **Graph Editor**:

   * Node palette (Start, Text, Media, Question, Condition, Action, Delay, End)
   * Property panel per node type (forms w/ live validation)
   * Minimap, zoom, snap-to-grid, multi-select, undo/redo
   * Auto-layout (dagre) button
   * **Telegram-like rendering**: message blocks styled like Telegram chat:

     * avatar bubble, text Markdown, inline keyboard preview
     * keyboard layout controls (rows/cols/wide)
3. **Save Draft / Publish**:

   * Debounced autosave (800ms) to `/api/projects/:id/versions`
   * Validate → publish to `/api/versions/:id/publish`
4. **Simulator**:

   * Panel showing a chat preview that follows the current graph locally (no Telegram)
   * Input box to type user message; buttons clickable (simulate callback\_data)
   * Live `ctx` viewer

### Frontend Env

```
VITE_API_BASE=http://localhost:8000
VITE_DEFAULT_LOCALE=ar
```

### File Structure

```
frontend/
  src/
    api/ (client.ts, projects.ts, versions.ts, bots.ts)
    graph/
      editor/ (GraphEditor.tsx, NodeTypes/*, EdgeTypes/*, PropertyPanel/*)
      simulator/ (Simulator.tsx)
      validators/ (validateGraph.ts)
      types.ts
    pages/ (ProjectsPage.tsx, EditorPage.tsx, SettingsPage.tsx)
    store/ (useGraphStore.ts, useUIStore.ts)
    i18n/ (ar.json, en.json)
    app.tsx, main.tsx
  index.html
```

### Acceptance (FE)

* Create project → open editor → add nodes/edges → save/publish → success toast.
* Simulator highlights path transitions.
* Graph JSON exported/imported via menu.
* Keyboard layout renders rows/cols exactly like Telegram inline keyboard grid.

---

## PHASE B — BACKEND (Django + DRF + aiogram)

### Stack

* Python 3.12, Django 5, DRF
* aiogram v3 (bots), httpx for HTTP calls
* PostgreSQL, Redis
* Celery (worker + beat) for delays/bulk sends
* drf-spectacular for OpenAPI
* Field encryption for bot tokens (fernet)

### Apps

* `builder` (models/serializers/views/validators)
* `runtime` (FSM engine + Telegram handlers)
* `common` (utils, crypto, http client)

### Models (exact fields may include lengths/indexing)

```py
class Bot(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    owner = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    name = models.CharField(max_length=120)
    token = EncryptedTextField()               # fernet
    is_active = models.BooleanField(default=False)
    webhook_secret = models.CharField(max_length=64, default=secrets.token_hex(32))
    created_at = models.DateTimeField(auto_now_add=True)

class Project(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    bot = models.ForeignKey(Bot, on_delete=models.CASCADE, related_name='projects')
    title = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)

class GraphVersion(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name='versions')
    graph_json = models.JSONField()
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

class Session(models.Model):
    bot = models.ForeignKey(Bot, on_delete=models.CASCADE)
    chat_id = models.BigIntegerField()
    state = models.CharField(max_length=120, default='start')
    context = models.JSONField(default=dict)
    updated_at = models.DateTimeField(auto_now=True)
    class Meta: unique_together = ('bot','chat_id')

class EventLog(models.Model):
    bot = models.ForeignKey(Bot, on_delete=models.CASCADE, null=True)
    chat_id = models.BigIntegerField(null=True)
    level = models.CharField(max_length=10, default='INFO')
    event_type = models.CharField(max_length=50, null=True)
    payload = models.JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
```

### Validators (server)

* Pydantic schema mirrors `GraphJSON`.
* Enforce rules listed in “Validation rules”.
* Run on `POST /api/projects/:id/versions` and again on publish.

### Runtime (FSM)

* `runtime/engine.py`:

  * `load_published_graph(bot_id)` → from Redis (`graph:{bot_id}`) else DB
  * `handle_update(bot, update_dict)`:

    1. extract `chat_id`, `text`, `callback_data`
    2. get/create `Session`
    3. resolve current node; process by type:

       * **Text/Media**: send message (aiogram), then follow first outgoing edge (auto)
       * **Question**: if no input → send inline keyboard and stay; if button pressed → set `ctx[var]` and traverse matching edge (`condition == button.text || value || callback_data`)
       * **Condition**: evaluate `expr` against safe context `{ctx, input}` using a sandboxed evaluator (no `eval`)
       * **Action**: call HTTP (httpx) with templated payload (`{{ctx.*}}`, `{{chat_id}}`); if `save_to` present, store response JSON under `ctx[save_to]`; choose edge “success”/“error” if labeled, else first
       * **Delay**: schedule `send_delayed` Celery task with `countdown=ms/1000`, then follow first edge
       * **End**: do nothing
    4. update session state/context; write EventLog
* **Rate limits**: Redis-based throttle per bot to respect Telegram limits.
* **Aiogram**:

  * Create per-bot Bot instance (registry/cache).
  * Webhook view `POST /tg/webhook/:bot_uuid/` verifies secret header if configured.
  * Dev command: `manage.py poll_updates --bot <uuid>` (long polling).

### DRF Endpoints

Implement endpoints listed in **REST API** section, plus:

* `POST /api/bots` → create bot, store encrypted token, optionally call `setWebhook` to `${WEB_PUBLIC_URL}/tg/webhook/:bot_uuid/` with secret.
* `GET /health` and `/metrics` (basic).

### Settings / ENV

```
DJANGO_SECRET_KEY=change_me
DATABASE_URL=postgres://postgres:postgres@postgres:5432/app
REDIS_URL=redis://redis:6379/0
ALLOWED_HOSTS=*
WEB_PUBLIC_URL=https://localhost:8000
FERNET_KEY=base64url_32bytes
LOG_LEVEL=INFO
```

### Docker Compose

Services: `web` (Django+gunicorn), `worker` (Celery), `beat` (Celery beat), `postgres`, `redis`, `nginx` (optional).
Include healthchecks and volumes for Postgres. Auto-run `migrate` on startup.

### Makefile

```
make dev-up          # docker compose up
make migrate
make createsuperuser
make test
make fmt && make lint
```

### Tests

* Unit: validators, FSM transitions, templating.
* Integration: webhook → engine → aiogram send (mock).
* Seed: create a bot + project + a small “Home Menu” graph to validate publish.

---

## SEED GRAPH (Home Menu)

Use for FE simulator and BE tests:

```json
{
  "nodes": [
    { "id":"start","type":"Start","position":{"x":40,"y":40},"data":{"label":"Start"} },
    { "id":"home","type":"Text","position":{"x":40,"y":160},"data":{"text":"مرحباً! اختر خياراً:","keyboard":{"buttons":[{"text":"الخدمات"},{"text":"من نحن"},{"text":"تواصل"}],"layout":{"rows":3,"cols":1}}} },
    { "id":"q1","type":"Question","position":{"x":40,"y":320},"data":{"prompt":"القائمة الرئيسية","var":"menu_choice","buttons":[{"text":"الخدمات"},{"text":"من نحن"},{"text":"تواصل"}],"layout":{"rows":3,"cols":1}} },
    { "id":"services","type":"Text","position":{"x":-220,"y":520},"data":{"text":"قائمة الخدمات…","keyboard":{"buttons":[{"text":"رجوع","value":"back"}],"layout":{"rows":1,"cols":1}}} },
    { "id":"about","type":"Text","position":{"x":40,"y":520},"data":{"text":"نحن شركة برمجيات…","keyboard":{"buttons":[{"text":"رجوع","value":"back"}]}} },
    { "id":"contact","type":"Action","position":{"x":300,"y":520},"data":{"url":"https://example.com/lead","method":"POST","payload":{"chat_id":"{{chat_id}}","choice":"{{ctx.menu_choice}}"},"save_to":"lead"} },
    { "id":"end","type":"End","position":{"x":40,"y":720},"data":{"label":"End"} }
  ],
  "edges": [
    { "id":"e1","source":"start","target":"home" },
    { "id":"e2","source":"home","target":"q1" },
    { "id":"e3","source":"q1","target":"services","data":{"condition":"ctx.menu_choice == 'الخدمات'"} },
    { "id":"e4","source":"q1","target":"about","data":{"condition":"ctx.menu_choice == 'من نحن'"} },
    { "id":"e5","source":"q1","target":"contact","data":{"condition":"ctx.menu_choice == 'تواصل'"} },
    { "id":"e6","source":"services","target":"q1" },
    { "id":"e7","source":"about","target":"q1" },
    { "id":"e8","source":"contact","target":"end" }
  ],
  "meta":{"locale":"ar","version":1}
}
```

---

## ACCEPTANCE CRITERIA (end-to-end)

1. Run `docker compose up -d` → web at `:8000`, FE dev at `:5173` (or served by web in prod).
2. Create bot via API, set webhook automatically, store token encrypted.
3. Create project → build graph in UI → **Validate** → **Publish**.
4. FE simulator mirrors Telegram-like UI (message blocks + inline keyboards with rows/cols).
5. Send a real Telegram message to the bot → backend processes graph and replies; pressing an inline button routes to the correct node.
6. Delays dispatch via Celery; Action node hits mock external API and stores `ctx.lead`.

---

## TASKS FOR YOU (Codex)

* Scaffold both apps and implement all features as specified.
* Provide sample `.env.example`, `docker-compose.yml`, `Makefile`, and README with step-by-step launch (dev & prod).
* Include unit/integration tests and seed graph.
* Keep code modular and documented.

---

إذا بدك نسخة منفصلة للواجهة أو للباك-إند فقط، قلّي وأقسّمها لك فورًا.
