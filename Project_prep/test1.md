
Got you. Here’s the **simplest “just works” path** for Angular 20.3 to stream a Deephaven table into a grid, **with only supported imports** and no mysterious loaders. We’ll:

1. **Use Deephaven’s official JS API** via a dynamic `import()` of `dh-core.js` served by your DH server (no npm runtime packages needed).
    
2. **Open a CoreClient**, **login with PSK** (if you use PSK), **resolve the table**, and
    
3. **Subscribe to a viewport** and push rows into Angular via RxJS.
    
4. **Avoid CORS** in dev with a small Angular proxy.
    

> This follows Deephaven’s doc style: `import dh from 'http://<DH>/jsapi/dh-core.js'`, `new dh.CoreClient(...)`, `login(...)`, `setViewport(...)`, `getViewportData()` or subscribe for updates. ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))

---

# 0) Environment vars

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,
  // Your Deephaven server (no trailing slash)
  DEEPHAVEN_BASE_URL: 'http://localhost:10000',
  // If using PSK auth; otherwise leave empty string
  DEEPHAVEN_PSK: 'very-secret-password'
};
```

---

# 1) (Dev only) Angular proxy to avoid CORS

`proxy.conf.json`

```json
{
  "/jsapi": {
    "target": "http://localhost:10000",
    "changeOrigin": true,
    "secure": false,
    "ws": true
  },
  "/grpc-web": {
    "target": "http://localhost:10000",
    "changeOrigin": true,
    "secure": false,
    "ws": true
  }
}
```

Update your dev script in `package.json`:

```json
{
  "scripts": {
    "start": "ng serve --proxy-config proxy.conf.json"
  }
}
```

This lets your Angular app hit `/jsapi/...` and `/grpc-web` **without** CORS headaches during `ng serve`. (DH’s examples also assume serving JS from `/jsapi` on the DH host. We proxy that path to match.) ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))

---

# 2) Angular service (connect + live viewport)

`src/app/deephaven.service.ts`

```ts
import { Injectable, NgZone } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { environment } from '../environments/environment';

type DhNamespace = any;      // from dynamic import of dh-core.js
type DhClient = any;         // dh.CoreClient
type DhTable = any;          // dh.Table

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNamespace;
  private client!: DhClient;

  constructor(private zone: NgZone) {}

  private async ensureDhLoaded(): Promise<void> {
    if (this.dh) return;
    // In dev, we go through Angular proxy: /jsapi/dh-core.js
    // In prod, you can point at `${environment.DEEPHAVEN_BASE_URL}/jsapi/dh-core.js`
    this.dh = (await import('/jsapi/dh-core.js')).default;
  }

  /** Connect & auth (PSK optional) */
  private async getClient(): Promise<DhClient> {
    await this.ensureDhLoaded();
    if (this.client) return this.client;

    this.client = new this.dh.CoreClient(environment.DEEPHAVEN_BASE_URL);

    // PSK auth (docs show this exact flow)
    // Skip if you don’t use PSK.
    if (environment.DEEPHAVEN_PSK) {
      await this.client.login({
        type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
        token: environment.DEEPHAVEN_PSK,
      });
    }

    return this.client;
  }

  /**
   * Stream a table by variable name in the session (or one you create).
   * Emits rows (array of objects) on each update.
   */
  async streamTable(tableName: string, maxRows = 200): Promise<Observable<any[]>> {
    const rows$ = new BehaviorSubject<any[]>([]);

    const client = await this.getClient();
    const ideConn = await client.getAsIdeConnection();
    const ide = await ideConn.startSession('python'); // default session

    // If your table already exists in the session with the given name (e.g., created by your DH script),
    // this grabs it; otherwise you can create a quick demo table like the docs do.
    // Remove this "create if missing" block in real usage.
    try {
      await ide.getTable(tableName);
    } catch {
      await ide.runCode(`
from deephaven import time_table
${tableName} = time_table("00:00:01").update_view([
  "I=i",
  "Msg = i % 2 == 0 ? \`Hello\` : \`World\`"
])`);
    }

    const table: DhTable = await ide.getTable(tableName);

    // Set a viewport and listen for updates
    table.setViewport(0, Math.max(0, maxRows - 1));

    const updateHandler = async () => {
      try {
        const vp = await table.getViewportData();
        const cols = table.columns;
        const out: any[] = [];

        // vp.rows is documented in the official example
        // Map each row into a simple object keyed by column names
        for (let i = 0; i < vp.rows.length; i++) {
          const row = vp.rows[i];
          const obj: Record<string, unknown> = {};
          for (let c = 0; c < cols.length; c++) {
            obj[cols[c].name] = row.get(cols[c]);
          }
          out.push(obj);
        }

        // Run the emit inside Angular zone so UI updates
        this.zone.run(() => rows$.next(out));
      } catch (e) {
        console.error('Viewport update error', e);
      }
    };

    // For viewports, table fires "updated" events (per docs)
    table.addEventListener('updated', updateHandler);

    // Also do an initial fetch
    await updateHandler();

    // You may expose a cleanup if you want:
    // return { rows$, close: () => table.removeEventListener('updated', updateHandler) }
    return rows$.asObservable();
  }
}
```

**Why this works / matches docs**

- It uses the **official `dh-core.js`** and **`CoreClient`** exactly as shown in Deephaven’s “Use the JS API” guide (import, `CoreClient`, PSK login).
    
- It uses **viewports** + **“updated”** events consistent with the JS API concepts doc (subscribe updates to the active viewport, read via `getViewportData()`). ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))
    

---

# 3) Angular component (simple grid)

`src/app/stream-grid/stream-grid.component.ts`

```ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { Subscription } from 'rxjs';
import { DeephavenService } from '../deephaven.service';

@Component({
  selector: 'app-stream-grid',
  templateUrl: './stream-grid.component.html',
})
export class StreamGridComponent implements OnInit, OnDestroy {
  cols: string[] = [];
  rows: any[] = [];
  private sub?: Subscription;

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    const rows$ = await this.dh.streamTable('user', 200); // <-- your DH table variable name
    this.sub = rows$.subscribe(rows => {
      this.rows = rows;
      this.cols = rows.length ? Object.keys(rows[0]) : [];
    });
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
  }
}
```

`src/app/stream-grid/stream-grid.component.html`

```html
<div class="grid-container">
  <table class="grid">
    <thead>
      <tr>
        <th *ngFor="let c of cols">{{ c }}</th>
      </tr>
    </thead>
    <tbody>
      <tr *ngFor="let r of rows">
        <td *ngFor="let c of cols">{{ r[c] }}</td>
      </tr>
    </tbody>
  </table>
</div>
```

A basic CSS (optional):

```css
.grid-container {
  overflow: auto;
  max-height: 70vh;
  border: 1px solid #ddd;
}
.grid {
  width: 100%;
  border-collapse: collapse;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
  font-size: 14px;
}
.grid th, .grid td {
  padding: 6px 10px;
  border-bottom: 1px solid #eee;
  text-align: left;
  white-space: nowrap;
}
.grid thead th {
  position: sticky;
  top: 0;
  background: #fafafa;
}
```

---

## Notes & gotchas

- **Table source**  
    The snippet assumes the table **exists in the Python session** under the name you pass (`'user'`). If your table is created elsewhere (e.g., Kafka → script → publish), either:
    
    - create/import it in the session you start, **or**
        
    - resolve via the **Deephaven URI**/publish mechanism (if you publish tables and resolve them by URI). ([Medium](https://medium.com/%40deephavendatalabs/publish-data-like-you-never-have-before-8593561dd2f6 "Publish data like you never have before | by Deephaven Data Labs | Medium"))
        
- **Why dynamic import and not npm?**  
    Deephaven’s own guide imports the runtime directly from the **server’s `/jsapi/dh-core.js`** and uses `CoreClient` (no need for extra loaders like “loaddhcore”). This is the most version-proof, minimal route and avoids Angular bundler conflicts. ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))
    
- **Auth**  
    PSK login shown above is exactly as in docs. If your deployment uses a different auth type (token, SSO), swap the `login(...)` call accordingly. ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))
    
- **Real-time updates**  
    We use a **viewport** + **`updated`** event. For whole-table charts/etc., you can use **`table.subscribe()`** to get full-table subscriptions (be careful with very large tables). ([deephaven.io](https://deephaven.io/core/docs/reference/js-api/concepts/ "Javascript API Concepts | Deephaven"))
    
- **CORS / WebSocket**  
    The Angular proxy handles `/jsapi` and `/grpc-web` to the DH host during dev so you don’t fight CORS. In production, serve Angular behind the same reverse proxy (Nginx/Traefik) as Deephaven so they’re on the **same origin**.
    

---

If you drop these files in, run `npm start`, and ensure the `user` table exists in your DH Python session, you should see the rows stream into the table in real time. If you want this with **AG Grid** instead of a simple `<table>`, I can swap the template to AG Grid in the same service pattern.

--------------------------------

That error means Angular can’t actually load the file the service tries to import. Let’s fix it in a bullet-proof way:

## 1) Verify the file really exists

Open this in your browser (replace host/port):

- `http://localhost:10000/jsapi/` → you should see a small index page
    
- `http://localhost:10000/jsapi/dh-core.js` → should download/serve the JS file
    

If either is 404, your Deephaven server isn’t serving the JS API assets (or a reverse proxy isn’t routing `/jsapi`). Deephaven’s docs show that `dh-core.js` is served from `/jsapi/dh-core.js`. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))

## 2) Use a _full URL_ fallback (and a proxy in dev)

Instead of importing only `'/jsapi/dh-core.js'`, try proxy first (nice for CORS), then fall back to the absolute URL.

**proxy.conf.json** (dev only; same as before)

```json
{
  "/jsapi": {
    "target": "http://localhost:10000",
    "changeOrigin": true,
    "secure": false,
    "ws": true
  },
  "/grpc-web": {
    "target": "http://localhost:10000",
    "changeOrigin": true,
    "secure": false,
    "ws": true
  }
}
```

Run dev with: `ng serve --proxy-config proxy.conf.json`

## 3) Drop-in fix for your service

Update just the loader method:

```ts
// deephaven.service.ts
import { environment } from '../environments/environment';

private async ensureDhLoaded(): Promise<void> {
  if (this.dh) return;

  const proxyUrl = '/jsapi/dh-core.js';
  const absoluteUrl = `${environment.DEEPHAVEN_BASE_URL.replace(/\/+$/, '')}/jsapi/dh-core.js`;

  let lastErr: unknown;

  // 1) Try via Angular dev proxy (no CORS in dev)
  try {
    this.dh = (await import(/* @vite-ignore */ proxyUrl)).default;
    return;
  } catch (e) {
    lastErr = e;
    console.warn('Proxy import failed, falling back to absolute URL...', e);
  }

  // 2) Fallback: full absolute URL (works in prod or if you don’t use the proxy)
  try {
    this.dh = (await import(/* @vite-ignore */ absoluteUrl)).default;
    return;
  } catch (e2) {
    console.error('Failed to load dh-core.js from both proxy and absolute URL', { proxyUrl, absoluteUrl, lastErr, e2 });
    throw e2;
  }
}
```

> Why this works: Deephaven’s official guide imports the JS API directly from the server at `/jsapi/dh-core.js`. Using the full URL ensures it doesn’t depend on your app’s base href, while the proxy keeps CORS away in dev. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))

## 4) If it still fails, one of these is the culprit

- **`environment.DEEPHAVEN_BASE_URL` wrong** (extra slash, wrong port, wrong host).  
    Example good value: `http://localhost:10000` (no trailing slash).
    
- **Deephaven not exposing `/jsapi`** behind your reverse proxy. Add a route for `/jsapi` to the DH server.
    
- **Didn’t start Angular with the proxy.** Use `ng serve --proxy-config proxy.conf.json`.
    
- **CORS** (if you skip the proxy). Then the absolute URL request to `/jsapi/dh-core.js` must be allowed by your server.
    

## 5) Quick sanity check

From your Angular app’s dev console (F12 → Network), reload and confirm:

- A `GET /jsapi/dh-core.js` (status 200, served by either dev server proxy or DH directly).
    

Once `dh-core.js` is loading, the rest of the code (CoreClient → login → startSession → setViewport → `getViewportData`) will behave like in the docs example. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))

If you want, paste the exact Network tab error (status code + response) and I’ll pinpoint whether it’s a routing or CORS header issue.