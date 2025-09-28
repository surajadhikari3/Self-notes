
Perfect — here’s a clean, copy-paste-ready Angular 20.3 setup that:

- Opens **one** Deephaven session over **PSK** and keeps it alive
    
- Subscribes to **three** server tables (left, right, and a bottom **fat** table)
    
- **Dynamically** builds AG Grid columns from Deephaven’s schema (no hardcoding)
    
- Streams updates to AG Grid in real time (no page refresh)
    
- Adds one **Quick Filter** box that filters **all three** grids
    

> Assumes AG Grid Community 34.2 and `@deephaven/jsapi-bootstrap`.  
> Replace environment values with your DH URL/PSK and real table names.

---

# 0) Install dependencies

```bash
npm i ag-grid-angular@34.2.0 ag-grid-community@34.2.0 @deephaven/jsapi-bootstrap
```

---

# 1) Global styles (AG Grid themes)

`src/styles.css`

```css
@import 'ag-grid-community/styles/ag-grid.css';
@import 'ag-grid-community/styles/ag-theme-quartz.css';

/* Optional dark-friendly base */
:root { color-scheme: dark; }
html, body { margin: 0; padding: 0; height: 100%; }
```

---

# 2) Environment config

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,

  // Deephaven connection (no trailing slash)
  DEEPHAVEN_URL: 'http://localhost:10000',  // <-- change to your DH base URL
  DEEPHAVEN_PSK: 'your-psk-here',           // <-- don't commit real secrets

  // Live tables that already exist on the server
  TABLE_LEFT: 'user_raw',
  TABLE_RIGHT: 'account_raw',
  TABLE_FAT: 'fat_table',   // wide / denormalized table prepared in DH

  // Viewport slice (tune for your dataset size)
  VIEWPORT_TOP: 0,
  VIEWPORT_BOTTOM: 9999,
};
```

`src/environments/environment.prod.ts`

```ts
export const environment = {
  production: true,

  // Set your real production values during your CI build
  DEEPHAVEN_URL: 'https://your-prod-dh.example.com',
  DEEPHAVEN_PSK: 'prod-psk',

  TABLE_LEFT: 'user_raw',
  TABLE_RIGHT: 'account_raw',
  TABLE_FAT: 'fat_table',

  VIEWPORT_TOP: 0,
  VIEWPORT_BOTTOM: 9999,
};
```

---

# 3) Deephaven service (single session + live viewport subscriptions)

`src/app/services/deephaven.service.ts`

```ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import loadDhCore from '@deephaven/jsapi-bootstrap';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhTable = any;
type DhViewportSub = any;

export interface LiveTableState {
  cols: string[];
  rows: any[];
  ready: boolean;
  error?: string;
}

@Injectable({ providedIn: 'root' })
export class DeephavenService implements OnDestroy {
  private dh!: DhNS;
  private client: any;
  private ide: any; // python session
  private connected = false;

  private tables = new Map<string, DhTable>();
  private viewSubs = new Map<string, DhViewportSub>();
  private streams = new Map<string, BehaviorSubject<LiveTableState>>();

  /** Create one client+session and keep it for the app lifetime. */
  async ensureConnected(): Promise<void> {
    if (this.connected) return;

    // 1) Load dh-core.js from your server (version-safe)
    this.dh = await loadDhCore({ baseUrl: `${environment.DEEPHAVEN_URL}/jsapi` });

    // 2) Create client & PSK login
    this.client = new this.dh.CoreClient(environment.DEEPHAVEN_URL);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: environment.DEEPHAVEN_PSK,
    });

    // 3) Start an IDE session to fetch tables by name
    const asIde = await this.client.getAsIdeConnection();
    this.ide = await asIde.startSession('python');

    this.connected = true;
  }

  /** Subscribe to a DH table by name and stream viewport updates to an observable. */
  async streamTable(tableName: string, top?: number, bottom?: number): Promise<Observable<LiveTableState>> {
    await this.ensureConnected();

    if (this.streams.has(tableName)) {
      return this.streams.get(tableName)!.asObservable();
    }

    const subject = new BehaviorSubject<LiveTableState>({ cols: [], rows: [], ready: false });
    this.streams.set(tableName, subject);

    try {
      const table: DhTable = await this.ide.getTable(tableName);
      this.tables.set(tableName, table);

      const vpTop = top ?? environment.VIEWPORT_TOP;
      const vpBottom = bottom ?? environment.VIEWPORT_BOTTOM;

      // Set viewport and subscribe to updates
      table.setViewport(vpTop, vpBottom);
      const sub: DhViewportSub = table.subscribe();

      const onUpdated = async () => {
        try {
          const view = await sub.getViewportData();
          const cols = table.columns.map((c: any) => c.name);
          const rows = view.rows.map((r: any) => {
            const obj: any = {};
            for (const c of cols) obj[c] = r.get(c);
            return obj;
          });
          subject.next({ cols, rows, ready: true });
        } catch (err: any) {
          subject.next({ cols: [], rows: [], ready: false, error: String(err?.message ?? err) });
        }
      };

      // first fill + listen for future ticks
      await onUpdated();
      sub.addEventListener('updated', onUpdated);

      // Save for cleanup
      this.viewSubs.set(tableName, sub);
    } catch (e: any) {
      subject.next({ cols: [], rows: [], ready: false, error: String(e?.message ?? e) });
    }

    return subject.asObservable();
  }

  ngOnDestroy(): void {
    for (const [, sub] of this.viewSubs) {
      try { sub.close?.(); } catch {}
    }
    this.viewSubs.clear();
    this.tables.clear();
  }
}
```

---

# 4) Three-grid layout with **dynamic columns** and shared search

`src/app/app.component.ts`

```ts
import { Component, OnInit, ViewChild, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AgGridAngular } from 'ag-grid-angular';
import type { ColDef, GridOptions, ValueFormatterParams } from 'ag-grid-community';
import { DeephavenService, LiveTableState } from './services/deephaven.service';
import { environment } from '../environments/environment';
import { map } from 'rxjs/operators';
import { combineLatest } from 'rxjs';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule, AgGridAngular],
  template: `
  <div class="app">
    <header>
      <h1>Deephaven → AG Grid (Live)</h1>
      <input class="search" type="text" placeholder="Search all grids…" (input)="onQuickFilter($event)" />
      <span class="status" [class.ok]="allReady()" [class.err]="hasError()">
        {{ status() }}
      </span>
    </header>

    <section class="top">
      <div class="pane">
        <h3>{{ leftName }}</h3>
        <ag-grid-angular
          #leftGrid
          class="ag-theme-quartz"
          style="width: 100%; height: 420px"
          [rowData]="leftRows"
          [columnDefs]="leftCols"
          [gridOptions]="gridOpts"
          (gridReady)="onGridReady($event)">
        </ag-grid-angular>
      </div>

      <div class="pane">
        <h3>{{ rightName }}</h3>
        <ag-grid-angular
          #rightGrid
          class="ag-theme-quartz"
          style="width: 100%; height: 420px"
          [rowData]="rightRows"
          [columnDefs]="rightCols"
          [gridOptions]="gridOpts"
          (gridReady)="onGridReady($event)">
        </ag-grid-angular>
      </div>
    </section>

    <section class="bottom">
      <h3>{{ fatName }}</h3>
      <ag-grid-angular
        #fatGrid
        class="ag-theme-quartz"
        style="width: 100%; height: 420px"
        [rowData]="fatRows"
        [columnDefs]="fatCols"
        [gridOptions]="gridOpts"
        (gridReady)="onGridReady($event)">
      </ag-grid-angular>
    </section>
  </div>
  `,
  styles: [`
    .app { padding: 12px; font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
    header { display:flex; align-items:center; gap:12px; margin-bottom:10px; }
    h1 { font-size: 18px; margin: 0 8px 0 0; }
    .search { flex: 1; max-width: 420px; padding:8px 10px; border-radius: 10px; border: 1px solid #3a3a3a; background:#121212; color:#eaeaea; }
    .status { font-size: 12px; opacity: 0.85 }
    .status.ok { color: #3adb84 }
    .status.err { color: #ff6b6b }
    .top { display:grid; grid-template-columns: 1fr 1fr; gap: 10px; }
    .pane { background: rgba(255,255,255,0.02); border: 1px solid rgba(255,255,255,0.08); border-radius: 12px; padding: 8px; }
    .bottom { margin-top: 10px; }
    h3 { margin: 6px 0 8px 4px; font-weight: 600; }
  `]
})
export class AppComponent implements OnInit {
  leftName = environment.TABLE_LEFT;
  rightName = environment.TABLE_RIGHT;
  fatName = environment.TABLE_FAT;

  leftCols: ColDef[] = [];
  rightCols: ColDef[] = [];
  fatCols: ColDef[] = [];

  leftRows: any[] = [];
  rightRows: any[] = [];
  fatRows: any[] = [];

  // status signals
  allReady = signal(false);
  hasError = signal(false);
  status = signal('Connecting…');

  @ViewChild('leftGrid') leftGrid!: AgGridAngular;
  @ViewChild('rightGrid') rightGrid!: AgGridAngular;
  @ViewChild('fatGrid') fatGrid!: AgGridAngular;

  // ------------ Grid Options (shared) ------------
  gridOpts: GridOptions = {
    animateRows: true,
    rowSelection: 'single',
    suppressFieldDotNotation: true,
    defaultColDef: {
      sortable: true,
      filter: true,          // column menu filter + responds to quickFilterText
      resizable: true,
      minWidth: 120,
      flex: 1,
    },
    // For massive tables, consider deltaRowDataMode with a stable getRowId
  };

  constructor(private dh: DeephavenService) {}

  // ---- dynamic columns helper (infers simple types) ----
  private dhColsToColDefs(dhColNames: string[], sampleRow?: any): ColDef[] {
    const inferType = (name: string): 'number' | 'date' | 'text' | 'boolean' => {
      const v = sampleRow?.[name];
      if (typeof v === 'number') return 'number';
      if (typeof v === 'boolean') return 'boolean';
      // Deephaven Java Instant -> JS Date is usually mapped; keep simple:
      if (v instanceof Date) return 'date';
      return 'text';
    };
    const dateFmt = (p: ValueFormatterParams) =>
      p.value instanceof Date ? p.value.toISOString() : p.value;

    return dhColNames.map((name) => {
      const t = inferType(name);
      const col: ColDef = {
        headerName: name,
        field: name,
        sortable: true,
        filter: true,
        resizable: true,
        minWidth: 120,
        flex: 1,
      };
      if (t === 'number') col.filter = 'agNumberColumnFilter';
      if (t === 'date')   { col.filter = 'agDateColumnFilter'; col.valueFormatter = dateFmt; }
      return col;
    });
  }

  // only swap columnDefs if fields actually changed (avoids churn)
  private sameCols(a: ColDef[], b: ColDef[]) {
    if (a.length !== b.length) return false;
    return a.every((c, i) => c.field === b[i].field);
  }

  async ngOnInit() {
    const left$ = await this.dh.streamTable(this.leftName);
    const right$ = await this.dh.streamTable(this.rightName);
    const fat$ = await this.dh.streamTable(this.fatName);

    const mapToGrid = (s$: any) => s$.pipe(
      map((t: LiveTableState) => {
        const nextDefs = this.dhColsToColDefs(t.cols, t.rows?.[0]);
        return {
          colDefs: nextDefs,
          rows: t.rows,
          ready: t.ready,
          error: t.error,
        };
      })
    );

    combineLatest([mapToGrid(left$), mapToGrid(right$), mapToGrid(fat$)])
      .subscribe(([L, R, F]) => {
        // left
        if (!this.sameCols(this.leftCols, L.colDefs)) this.leftCols = L.colDefs;
        this.leftRows = L.rows;

        // right
        if (!this.sameCols(this.rightCols, R.colDefs)) this.rightCols = R.colDefs;
        this.rightRows = R.rows;

        // fat
        if (!this.sameCols(this.fatCols, F.colDefs)) this.fatCols = F.colDefs;
        this.fatRows = F.rows;

        const ready = L.ready && R.ready && F.ready;
        const err = !!(L.error || R.error || F.error);
        this.allReady.set(ready && !err);
        this.hasError.set(err);
        this.status.set(err ? 'Live error (see console)' : (ready ? 'Live' : 'Connecting…'));
      });
  }

  onQuickFilter(e: Event) {
    const q = (e.target as HTMLInputElement).value ?? '';
    this.leftGrid.api?.setGridOption('quickFilterText', q);
    this.rightGrid.api?.setGridOption('quickFilterText', q);
    this.fatGrid.api?.setGridOption('quickFilterText', q);
  }

  onGridReady(event: any) {
    // Optional: fit columns on initial render
    event.api.sizeColumnsToFit();
  }
}
```

---

# 5) Bootstrap (standalone app)

`src/main.ts`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent).catch(err => console.error(err));
```

---

## Run checklist

1. Update `environment.ts` with your **DEEPHAVEN_URL**, **DEEPHAVEN_PSK**, and the three **table names**.
    
2. `npm start` (or `ng serve`) and open the app.
    
3. You should see two live grids side-by-side (left/right) and the “fat” table full width below.
    
4. Type in the search box: it filters rows across **all three** grids.
    
5. Any update on the server tables streams into the grids automatically — **no page refresh** needed.
    

If you later want ultra-smooth updates on very large tables, tell me your stable row key (e.g., `row_id`) and I’ll flip on `deltaRowDataMode` with `getRowId`.
------------------------
oppp1

You’re seeing that error because you imported `loadDhCore` as a **named** export. It’s a **default** export in `@deephaven/jsapi-bootstrap`. Also, TypeScript sometimes needs `allowSyntheticDefaultImports`/`esModuleInterop` to recognize the default export cleanly.

Below is a **compatible, copy-paste** setup that works with your versions:

- `@deephaven/jsapi-bootstrap@1.5.x`
    
- Angular 20.3
    
- AG Grid 34.2
    

I’ve included two safe tweaks so you don’t fight TS config:

1. **Dynamic import** for `loadDhCore` (no TS flags needed).
    
2. Removed extra packages you don’t need in the browser (`ws`, `@deephaven/jsapi-types`).
    

---

# 0) package.json (deps that matter)

```json
{
  "private": true,
  "dependencies": {
    "@angular/common": "20.3.0",
    "@angular/compiler": "20.3.0",
    "@angular/core": "20.3.0",
    "@angular/forms": "20.3.0",
    "@angular/platform-browser": "20.3.0",
    "@angular/router": "20.3.0",

    "@deephaven/jsapi-bootstrap": "^1.5.3",

    "ag-grid-angular": "^34.2.0",
    "ag-grid-community": "^34.2.0",

    "rxjs": "^7.8.0",
    "tslib": "^2.3.0",
    "zone.js": "^0.15.0"
  },
  "devDependencies": {
    "@angular/build": "20.3.2",
    "@angular/cli": "20.3.2",
    "@angular/compiler-cli": "20.3.0",
    "@types/jasmine": "^5.1.0",
    "jasmine-core": "^5.9.0",
    "karma": "^6.4.0",
    "karma-chrome-launcher": "^3.2.0",
    "karma-coverage": "^2.2.0",
    "karma-jasmine": "^5.1.0",
    "typescript": "~5.5.0"
  }
}
```

> If you already installed `@deephaven/jsapi-types` or `ws`, you can keep them, but they’re not required here. Run `rm -rf node_modules package-lock.json && npm i` after updating.

---

# 1) Optional TS config (only if you want to use static default import)

If you prefer `import loadDhCore from '@deephaven/jsapi-bootstrap'`, add this to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "strict": true
  }
}
```

**OR** just use the dynamic import I show below and you don’t need these flags.

---

# 2) styles (AG Grid theme)

`src/styles.css`

```css
@import 'ag-grid-community/styles/ag-grid.css';
@import 'ag-grid-community/styles/ag-theme-quartz.css';

:root { color-scheme: dark; }
html, body { margin: 0; padding: 0; height: 100%; }
```

---

# 3) environments

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,

  DEEPHAVEN_URL: 'http://localhost:10000', // <— change to your DH base URL
  DEEPHAVEN_PSK: 'your-psk-here',

  TABLE_LEFT: 'user_raw',
  TABLE_RIGHT: 'account_raw',
  TABLE_FAT: 'fat_table',

  VIEWPORT_TOP: 0,
  VIEWPORT_BOTTOM: 9999,
};
```

`src/environments/environment.prod.ts`

```ts
export const environment = {
  production: true,

  DEEPHAVEN_URL: 'https://your-prod-dh',
  DEEPHAVEN_PSK: 'prod-psk',

  TABLE_LEFT: 'user_raw',
  TABLE_RIGHT: 'account_raw',
  TABLE_FAT: 'fat_table',

  VIEWPORT_TOP: 0,
  VIEWPORT_BOTTOM: 9999,
};
```

---

# 4) Deephaven service (uses **dynamic import** for `loadDhCore`)

`src/app/services/deephaven.service.ts`

```ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhTable = any;
type DhViewportSub = any;

export interface LiveTableState {
  cols: string[];
  rows: any[];
  ready: boolean;
  error?: string;
}

@Injectable({ providedIn: 'root' })
export class DeephavenService implements OnDestroy {
  private dh!: DhNS;
  private client: any;
  private ide: any; // python session
  private connected = false;

  private tables = new Map<string, DhTable>();
  private viewSubs = new Map<string, DhViewportSub>();
  private streams = new Map<string, BehaviorSubject<LiveTableState>>();

  /** Create one client+session and keep it for the app lifetime. */
  async ensureConnected(): Promise<void> {
    if (this.connected) return;

    // ---- Dynamic import avoids TS default-export quirks ----
    const { default: loadDhCore } = await import('@deephaven/jsapi-bootstrap');

    // 1) Load dh-core.js from your server (version-safe)
    this.dh = await loadDhCore({ baseUrl: `${environment.DEEPHAVEN_URL}/jsapi` });

    // 2) Create client & PSK login
    this.client = new this.dh.CoreClient(environment.DEEPHAVEN_URL);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: environment.DEEPHAVEN_PSK,
    });

    // 3) Start an IDE session to fetch tables by name
    const asIde = await this.client.getAsIdeConnection();
    this.ide = await asIde.startSession('python');

    this.connected = true;
  }

  /** Subscribe to a DH table by name and stream viewport updates to an observable. */
  async streamTable(tableName: string, top?: number, bottom?: number): Promise<Observable<LiveTableState>> {
    await this.ensureConnected();

    if (this.streams.has(tableName)) {
      return this.streams.get(tableName)!.asObservable();
    }

    const subject = new BehaviorSubject<LiveTableState>({ cols: [], rows: [], ready: false });
    this.streams.set(tableName, subject);

    try {
      const table: DhTable = await this.ide.getTable(tableName);
      this.tables.set(tableName, table);

      const vpTop = top ?? environment.VIEWPORT_TOP;
      const vpBottom = bottom ?? environment.VIEWPORT_BOTTOM;

      table.setViewport(vpTop, vpBottom);
      const sub: DhViewportSub = table.subscribe();

      const onUpdated = async () => {
        try {
          const view = await sub.getViewportData();
          const cols = table.columns.map((c: any) => c.name);
          const rows = view.rows.map((r: any) => {
            const obj: any = {};
            for (const c of cols) obj[c] = r.get(c);
            return obj;
          });
          subject.next({ cols, rows, ready: true });
        } catch (err: any) {
          subject.next({ cols: [], rows: [], ready: false, error: String(err?.message ?? err) });
        }
      };

      await onUpdated();
      sub.addEventListener('updated', onUpdated);
      this.viewSubs.set(tableName, sub);
    } catch (e: any) {
      subject.next({ cols: [], rows: [], ready: false, error: String(e?.message ?? e) });
    }

    return subject.asObservable();
  }

  ngOnDestroy(): void {
    for (const [, sub] of this.viewSubs) {
      try { sub.close?.(); } catch {}
    }
    this.viewSubs.clear();
    this.tables.clear();
  }
}
```

---

# 5) App component (dynamic columns + shared quick filter)

`src/app/app.component.ts`

```ts
import { Component, OnInit, ViewChild, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AgGridAngular } from 'ag-grid-angular';
import type { ColDef, GridOptions, ValueFormatterParams } from 'ag-grid-community';
import { DeephavenService, LiveTableState } from './services/deephaven.service';
import { environment } from '../environments/environment';
import { map } from 'rxjs/operators';
import { combineLatest } from 'rxjs';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule, AgGridAngular],
  template: `
  <div class="app">
    <header>
      <h1>Deephaven → AG Grid (Live)</h1>
      <input class="search" type="text" placeholder="Search all grids…" (input)="onQuickFilter($event)" />
      <span class="status" [class.ok]="allReady()" [class.err]="hasError()">
        {{ status() }}
      </span>
    </header>

    <section class="top">
      <div class="pane">
        <h3>{{ leftName }}</h3>
        <ag-grid-angular
          #leftGrid
          class="ag-theme-quartz"
          style="width: 100%; height: 420px"
          [rowData]="leftRows"
          [columnDefs]="leftCols"
          [gridOptions]="gridOpts"
          (gridReady)="onGridReady($event)">
        </ag-grid-angular>
      </div>

      <div class="pane">
        <h3>{{ rightName }}</h3>
        <ag-grid-angular
          #rightGrid
          class="ag-theme-quartz"
          style="width: 100%; height: 420px"
          [rowData]="rightRows"
          [columnDefs]="rightCols"
          [gridOptions]="gridOpts"
          (gridReady)="onGridReady($event)">
        </ag-grid-angular>
      </div>
    </section>

    <section class="bottom">
      <h3>{{ fatName }}</h3>
      <ag-grid-angular
        #fatGrid
        class="ag-theme-quartz"
        style="width: 100%; height: 420px"
        [rowData]="fatRows"
        [columnDefs]="fatCols"
        [gridOptions]="gridOpts"
        (gridReady)="onGridReady($event)">
      </ag-grid-angular>
    </section>
  </div>
  `,
  styles: [`
    .app { padding: 12px; font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
    header { display:flex; align-items:center; gap:12px; margin-bottom:10px; }
    h1 { font-size: 18px; margin: 0 8px 0 0; }
    .search { flex: 1; max-width: 420px; padding:8px 10px; border-radius: 10px; border: 1px solid #3a3a3a; background:#121212; color:#eaeaea; }
    .status { font-size: 12px; opacity: 0.85 }
    .status.ok { color: #3adb84 }
    .status.err { color: #ff6b6b }
    .top { display:grid; grid-template-columns: 1fr 1fr; gap: 10px; }
    .pane { background: rgba(255,255,255,0.02); border: 1px solid rgba(255,255,255,0.08); border-radius: 12px; padding: 8px; }
    .bottom { margin-top: 10px; }
    h3 { margin: 6px 0 8px 4px; font-weight: 600; }
  `]
})
export class AppComponent implements OnInit {
  leftName = environment.TABLE_LEFT;
  rightName = environment.TABLE_RIGHT;
  fatName = environment.TABLE_FAT;

  leftCols: ColDef[] = [];
  rightCols: ColDef[] = [];
  fatCols: ColDef[] = [];

  leftRows: any[] = [];
  rightRows: any[] = [];
  fatRows: any[] = [];

  allReady = signal(false);
  hasError = signal(false);
  status = signal('Connecting…');

  @ViewChild('leftGrid') leftGrid!: AgGridAngular;
  @ViewChild('rightGrid') rightGrid!: AgGridAngular;
  @ViewChild('fatGrid') fatGrid!: AgGridAngular;

  gridOpts: GridOptions = {
    animateRows: true,
    rowSelection: 'single',
    suppressFieldDotNotation: true,
    defaultColDef: {
      sortable: true,
      filter: true,
      resizable: true,
      minWidth: 120,
      flex: 1,
    },
  };

  constructor(private dh: DeephavenService) {}

  private dhColsToColDefs(dhColNames: string[], sampleRow?: any): ColDef[] {
    const inferType = (name: string): 'number' | 'date' | 'text' | 'boolean' => {
      const v = sampleRow?.[name];
      if (typeof v === 'number') return 'number';
      if (typeof v === 'boolean') return 'boolean';
      if (v instanceof Date) return 'date';
      return 'text';
    };
    const dateFmt = (p: ValueFormatterParams) =>
      p.value instanceof Date ? p.value.toISOString() : p.value;

    return dhColNames.map((name) => {
      const t = inferType(name);
      const col: ColDef = {
        headerName: name,
        field: name,
        sortable: true,
        filter: true,
        resizable: true,
        minWidth: 120,
        flex: 1,
      };
      if (t === 'number') col.filter = 'agNumberColumnFilter';
      if (t === 'date')   { col.filter = 'agDateColumnFilter'; col.valueFormatter = dateFmt; }
      return col;
    });
  }

  private sameCols(a: ColDef[], b: ColDef[]) {
    if (a.length !== b.length) return false;
    return a.every((c, i) => c.field === b[i].field);
  }

  async ngOnInit() {
    const left$ = await this.dh.streamTable(this.leftName);
    const right$ = await this.dh.streamTable(this.rightName);
    const fat$ = await this.dh.streamTable(this.fatName);

    const mapToGrid = (s$: any) => s$.pipe(
      map((t: LiveTableState) => {
        const nextDefs = this.dhColsToColDefs(t.cols, t.rows?.[0]);
        return { colDefs: nextDefs, rows: t.rows, ready: t.ready, error: t.error };
      })
    );

    combineLatest([mapToGrid(left$), mapToGrid(right$), mapToGrid(fat$)])
      .subscribe(([L, R, F]) => {
        if (!this.sameCols(this.leftCols, L.colDefs)) this.leftCols = L.colDefs;
        this.leftRows = L.rows;

        if (!this.sameCols(this.rightCols, R.colDefs)) this.rightCols = R.colDefs;
        this.rightRows = R.rows;

        if (!this.sameCols(this.fatCols, F.colDefs)) this.fatCols = F.colDefs;
        this.fatRows = F.rows;

        const ready = L.ready && R.ready && F.ready;
        const err = !!(L.error || R.error || F.error);
        this.allReady.set(ready && !err);
        this.hasError.set(err);
        this.status.set(err ? 'Live error (see console)' : (ready ? 'Live' : 'Connecting…'));
      });
  }

  onQuickFilter(e: Event) {
    const q = (e.target as HTMLInputElement).value ?? '';
    this.leftGrid.api?.setGridOption('quickFilterText', q);
    this.rightGrid.api?.setGridOption('quickFilterText', q);
    this.fatGrid.api?.setGridOption('quickFilterText', q);
  }

  onGridReady(event: any) {
    event.api.sizeColumnsToFit();
  }
}
```

---

# 6) Bootstrap

`src/main.ts`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent).catch(err => console.error(err));
```

---

## Why this fixes your error

- You were using `import { loadDhCore } from '@deephaven/jsapi-bootstrap'` (named import).  
    The package provides a **default export**. The dynamic import line  
    `const { default: loadDhCore } = await import('@deephaven/jsapi-bootstrap');`  
    works regardless of TS config and avoids the “no exported member” error.
    

If you still want a static import, switch to:

```ts
import loadDhCore from '@deephaven/jsapi-bootstrap';
```

and ensure `allowSyntheticDefaultImports` (and ideally `esModuleInterop`) are enabled in `tsconfig.json`.

Need me to flip this to `deltaRowDataMode` with a stable row ID for very large/fast tables? Tell me your key field and I’ll drop in the changes.


------------------------------


Good catch—that TS error happens because `@deephaven/jsapi-bootstrap`’s typing sometimes doesn’t expose a `default` field even though the runtime export is effectively “default”. Easiest fix: use a **runtime-safe dynamic import with a fallback** (and cast to `any`) so it works whether the module exposes `default` or not.

Replace just the `ensureConnected()` import bit in your `DeephavenService` with this:

```ts
// Deephaven service: fix the bootstrap import to be runtime-safe
async ensureConnected(): Promise<void> {
  if (this.connected) return;

  // ✅ Robust dynamic import: works whether the package exposes .default or not
  const mod: any = await import('@deephaven/jsapi-bootstrap');
  const loadDhCore = (mod?.default ?? mod) as (opts: { baseUrl: string }) => Promise<any>;

  // 1) Load dh-core.js from your server
  this.dh = await loadDhCore({ baseUrl: `${environment.DEEPHAVEN_URL}/jsapi` });

  // 2) Create client & PSK login
  this.client = new this.dh.CoreClient(environment.DEEPHAVEN_URL);
  await this.client.login({
    type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
    token: environment.DEEPHAVEN_PSK,
  });

  // 3) Start an IDE session to fetch tables by name
  const asIde = await this.client.getAsIdeConnection();
  this.ide = await asIde.startSession('python');

  this.connected = true;
}
```

That’s the only change you need to clear the “Property 'default' does not exist…” error.

### (Optional but nice) TS config

If you prefer a static default import later, add these to `tsconfig.json` and you can write `import loadDhCore from '@deephaven/jsapi-bootstrap';`:

```json
{
  "compilerOptions": {
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

### (Optional) Local typing shim

If your editor still nags, you can also add a tiny shim so TypeScript knows there’s a default export:

`src/types/jsapi-bootstrap.d.ts`

```ts
declare module '@deephaven/jsapi-bootstrap' {
  const loadDhCore: (opts: { baseUrl: string }) => Promise<any>;
  export default loadDhCore;
}
```

(Then ensure `"typeRoots"` includes `"src/types"` or just rely on TS’s default include.)

---

If anything else pops up (e.g., bundler vs node module resolution), tell me your exact `tsconfig.json` and I’ll tune it.

----------------------------------


Thanks for the screenshots — two things are happening:

1. **RxJS deprecation warning**: you accidentally hit the deprecated _rest-args_ overload of `combineLatest`.
    
2. **“R is possibly undefined”**: TypeScript can’t infer the tuple types from your `combineLatest` call, so it widens them to `any | undefined`.
    

Below is a drop-in fix that makes both go away by:

- Using the **object overload** of `combineLatest` (not deprecated), and
    
- Giving the stream a precise **type** so `L`, `R`, and `F` are never `undefined`.
    

### What to change (in your `app.component.ts`)

**A) Add a payload interface once near the top**

```ts
import type { ColDef } from 'ag-grid-community';
import { combineLatest, map } from 'rxjs';

interface GridPayload {
  colDefs: ColDef[];
  rows: any[];
  ready: boolean;
  error?: string;
}
```

**B) Make `mapToGrid` return that type**

```ts
const mapToGrid = (s$: any) =>
  s$.pipe(
    map((t: any): GridPayload => ({
      colDefs: this.dhColsToColDefs(t.cols, t.rows?.[0]),
      rows: t.rows ?? [],
      ready: !!t.ready,
      error: t.error,
    }))
  );
```

**C) Use the _object_ form of `combineLatest` (no deprecation), and typed values**

```ts
const leftGrid$  = mapToGrid(left$);
const rightGrid$ = mapToGrid(right$);
const fatGrid$   = mapToGrid(fat$);

combineLatest({
  L: leftGrid$,
  R: rightGrid$,
  F: fatGrid$,
}).subscribe(({ L, R, F }) => {
  // left
  if (!this.sameCols(this.leftCols, L.colDefs)) this.leftCols = L.colDefs;
  this.leftRows = L.rows;

  // right
  if (!this.sameCols(this.rightCols, R.colDefs)) this.rightCols = R.colDefs;
  this.rightRows = R.rows;

  // fat
  if (!this.sameCols(this.fatCols, F.colDefs)) this.fatCols = F.colDefs;
  this.fatRows = F.rows;

  const ready = L.ready && R.ready && F.ready;
  const err = !!(L.error || R.error || F.error);
  this.allReady.set(ready && !err);
  this.hasError.set(err);
  this.status.set(err ? 'Live error (see console)' : (ready ? 'Live' : 'Connecting…'));
});
```

That’s it. The object overload of `combineLatest` is the recommended path in RxJS 7→8, and the explicit `GridPayload` typing stops the `possibly undefined` errors.

If you still see a deprecation message, double-check you’re importing `combineLatest` from **'rxjs'** (not from `'rxjs/operators'`).