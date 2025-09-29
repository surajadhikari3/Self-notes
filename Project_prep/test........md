

Perfect — PSK auth, Angular **20.3**, and your DH table is named **`user`**. Below is a clean, copy-paste setup that streams that table in real time into an AG Grid.

---

# 1) Dev proxy (fixes CORS + WebSocket)

Create `proxy.conf.json` at your Angular project root:

```json
{
  "/jsapi": {
    "target": "http://localhost:10000",
    "secure": false,
    "ws": true,
    "changeOrigin": true
  },
  "/socket": {
    "target": "http://localhost:10000",
    "secure": false,
    "ws": true,
    "changeOrigin": true
  }
}
```

`package.json` → start script:

```json
{
  "scripts": {
    "start": "ng serve --proxy-config proxy.conf.json"
  }
}
```

> If your DH runs somewhere else, change the `target` host/port.

---

# 2) Deps

```bash
npm i ag-grid-community ag-grid-angular
npm i -D @deephaven/jsapi-types
```

Add AG Grid styles in `src/styles.css`:

```css
@import "ag-grid-community/styles/ag-grid.css";
@import "ag-grid-community/styles/ag-theme-alpine.css";
```

---

# 3) Environment with PSK

`src/environments/environment.ts` (mirror these to `environment.development.ts` if you use Angular’s file replacements)

```ts
export const environment = {
  production: false,
  // Because we proxy /jsapi and /socket through Angular dev server:
  deephavenBaseUrl: 'http://localhost:4200',
  deephavenAuth: {
    // PSK handler class name is required:
    type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
    // Put your DH PSK here for local/dev only
    token: 'YOUR_SUPER_SECRET_PSK'
  }
};
```

---

# 4) Deephaven service (connect + viewport stream)

`src/app/services/deephaven.service.ts`

```ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject, ReplaySubject } from 'rxjs';
import { environment } from '../../environments/environment';

// We’ll import the API at runtime from /jsapi/dh-core.js (served by DH)
type DhNamespace = typeof import('/jsapi/dh-core.js');

@Injectable({ providedIn: 'root' })
export class DeephavenService implements OnDestroy {
  private dh?: DhNamespace;
  private client?: any;         // dh.CoreClient
  private ide?: any;            // dh.IdeConnection / session
  private table?: any;          // dh.Table
  private viewport?: any;       // dh.TableViewportSubscription
  private unsubscribeViewport?: (() => void) | undefined;
  private connected = false;

  // Expose to components:
  readonly columns$ = new ReplaySubject<string[]>(1);
  readonly rows$ = new BehaviorSubject<any[]>([]);

  /** Connect once using PSK auth */
  private async ensureConnected(): Promise<void> {
    if (this.connected) return;

    const dh: DhNamespace = await import('/jsapi/dh-core.js') as any;
    this.dh = dh;

    this.client = new dh.CoreClient(environment.deephavenBaseUrl);
    await this.client.login(environment.deephavenAuth); // PSK login
    this.connected = true;
  }

  /**
   * Open a server-side table variable by name via an IDE session.
   * Your table variable name is 'user'.
   */
  async openUserTable(): Promise<void> {
    await this.ensureConnected();
    if (!this.dh || !this.client) throw new Error('Deephaven JSAPI not available');

    // Start Python session (works even if your table was created elsewhere; we just need a session to fetch by name)
    const ideConn = await this.client.getAsIdeConnection();
    this.ide = await ideConn.startSession('python');

    // If the 'user' table already exists on the server, just fetch it:
    this.table = await this.ide.getTable('user');

    await this.subscribeViewport();
  }

  /** Subscribe to a viewport so updates stream in real time */
  private async subscribeViewport(): Promise<void> {
    if (!this.table) throw new Error('No table available');

    // Column names (once)
    const cols = this.table.columns?.map((c: any) => c.name) ?? [];
    this.columns$.next(cols);

    // Viewport first 1,000 rows (adjust as you need)
    this.viewport = this.table.setViewport(0, 999);

    // Push current data immediately
    const sendRows = async () => {
      const vp = await this.viewport.getViewportData();
      const rows = vp.rows.map((row: any) => {
        const obj: any = {};
        for (const c of cols) obj[c] = row.get(c);
        return obj;
      });
      this.rows$.next(rows);
    };

    await sendRows();

    // Stream future updates
    this.unsubscribeViewport = this.viewport.addEventListener('updated', sendRows);
  }

  ngOnDestroy(): void {
    try { this.unsubscribeViewport?.(); } catch {}
    try { this.viewport?.close?.(); } catch {}
    try { this.ide?.close?.(); } catch {}
  }
}
```

---

# 5) Streaming grid component (standalone, Angular 20)

`src/app/streaming-grid/streaming-grid.component.ts`

```ts
import { Component, OnDestroy, OnInit, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AgGridAngular } from 'ag-grid-angular';
import { ColDef } from 'ag-grid-community';
import { Subscription } from 'rxjs';
import { DeephavenService } from '../services/deephaven.service';

@Component({
  selector: 'app-streaming-grid',
  standalone: true,
  imports: [CommonModule, AgGridAngular],
  template: `
    <div class="ag-theme-alpine" style="height: 80vh; width: 100%;">
      <ag-grid-angular
        [rowData]="rowData"
        [columnDefs]="colDefs"
        [deltaRowDataMode]="true"
        [getRowId]="getRowId"
        [rowBuffer]="0"
        [suppressAnimationFrame]="true"
      ></ag-grid-angular>
    </div>
  `
})
export class StreamingGridComponent implements OnInit, OnDestroy {
  private dh = inject(DeephavenService);

  colDefs: ColDef[] = [];
  rowData: any[] = [];
  private subCols?: Subscription;
  private subRows?: Subscription;

  // If your table has a natural key, use it instead of index.
  getRowId = (p: any) => p?.data?.__id ?? p.rowIndex;

  async ngOnInit() {
    await this.dh.openUserTable();

    this.subCols = this.dh.columns$.subscribe(names => {
      this.colDefs = names.map(n => ({ field: n, sortable: true, filter: true }));
    });

    this.subRows = this.dh.rows$.subscribe(rows => {
      // Add a synthetic id if you don’t have a key column
      this.rowData = rows.map((r, i) => ({ __id: i, ...r }));
    });
  }

  ngOnDestroy(): void {
    this.subCols?.unsubscribe();
    this.subRows?.unsubscribe();
  }
}
```

Use it in your app shell (standalone):

`src/app/app.component.ts`

```ts
import { Component } from '@angular/core';
import { StreamingGridComponent } from './streaming-grid/streaming-grid.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [StreamingGridComponent],
  template: `<app-streaming-grid></app-streaming-grid>`
})
export class AppComponent {}
```

`main.ts` stays the default bootstrap for standalone apps.

---

## Quick checklist

- Start DH (Community/Core) with PSK enabled (`DEEPHAVEN_PSK=YOUR_SUPER_SECRET_PSK` or JVM `-Dauthentication.psk=...`).
    
- Run Angular with the proxy: `npm start`.
    
- Open `http://localhost:4200`.
    
- You should see the **`user`** table streaming into the grid. If the table grows/ticks on the server, the grid updates live.
    

---

## Common gotchas (and fixes)

- **“loaddhcore is not a function” / cannot import JSAPI**  
    Always import from `/jsapi/dh-core.js` (served by your DH), not from npm. The dynamic `await import('/jsapi/dh-core.js')` is correct.
    
- **401 / still asks for auth**  
    Verify handler **type** string and **token** match server config exactly:
    
    ```ts
    await client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: 'YOUR_SUPER_SECRET_PSK'
    });
    ```
    
- **CORS / WebSocket blocked**  
    Use the proxy above in dev. In prod, front a reverse proxy so your Angular app and DH share origin (forward `/jsapi` and `/socket` to DH).
    
- **Big tables lag**  
    Lower the viewport (`setViewport(0, 199)`), and prefer server-side filters/sorts.
    

---

If you also have a **namespace path** for the same table (e.g., `['App', 'user']`), I can show you the `ObjectService` version too. But with your info, the IDE variable fetch (`getTable('user')`) is the simplest and streams in real time.