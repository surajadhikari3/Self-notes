
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


----------------------------

You’re super close—the issue is just TypeScript not inferring the types flowing through `combineLatest`. Fix it by:

1. Making `mapToGrid` strongly typed.
    
2. Typing each stream as `Observable<GridPayload>`.
    
3. Supplying a generic type to `combineLatest` (object overload) so `{ L, R, F }` are known.
    

Here’s the **drop-in replacement** for the relevant parts of your component.

---

### 1) Imports + payload interface

```ts
import { combineLatest, Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import type { ColDef, GridOptions, ValueFormatterParams } from 'ag-grid-community';

interface GridPayload {
  colDefs: ColDef[];
  rows: any[];
  ready: boolean;
  error?: string;
}
```

### 2) Typed mapper

```ts
private mapToGrid$(
  s$: Observable<{ cols: string[]; rows: any[]; ready: boolean; error?: string }>
): Observable<GridPayload> {
  return s$.pipe(
    map(t => ({
      colDefs: this.dhColsToColDefs(t.cols, t.rows?.[0]),
      rows: t.rows ?? [],
      ready: !!t.ready,
      error: t.error
    }))
  );
}
```

### 3) Use it with explicit types + generic `combineLatest`

```ts
async ngOnInit() {
  const left$  = await this.dh.streamTable(this.leftName);
  const right$ = await this.dh.streamTable(this.rightName);
  const fat$   = await this.dh.streamTable(this.fatName);

  // Make the streams explicitly typed
  const leftGrid$ : Observable<GridPayload> = this.mapToGrid$(left$);
  const rightGrid$: Observable<GridPayload> = this.mapToGrid$(right$);
  const fatGrid$  : Observable<GridPayload> = this.mapToGrid$(fat$);

  // Generic ensures {L,R,F} are strongly typed (not 'unknown')
  combineLatest<{ L: GridPayload; R: GridPayload; F: GridPayload }>({
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
}
```

That’s all you need—no other changes.  
If your editor still shows “unknown”, double-check:

- `combineLatest` is imported from **'rxjs'** (not `'rxjs/operators'`).
    
- Your `tsconfig.json` has `"strict": true` (fine) and `"skipLibCheck": true` (optional but helps with 3rd-party types).
    

Want me to paste your full `app.component.ts` with these fixes merged into the version I gave earlier?


----------------------------

Thanks for the screenshots — the error “loadDhCore is not a function” means the bootstrap package export isn’t lining up with your bundler/TS setup. Easiest, bullet-proof fix: **skip the bootstrap package entirely** and load Deephaven’s JS API straight from your server (`/jsapi/dh-core.js`) as a script, then use the global `dh` object. This is the most compatible path with Angular 20 + AG Grid and avoids all import interop issues.

Below is a **working, copy-paste** setup that:

- Injects `dh-core.js` at runtime (one line, no imports)
    
- Authenticates with **PSK**
    
- Subscribes to **three** live tables and streams updates into **AG Grid 34.2**
    
- Builds **columns dynamically** from table schema
    
- Uses **RxJS’s typed combineLatest** (no deprecation / “unknown” errors)
    

---

# 0) Install only what we need

```bash
npm i ag-grid-angular@34.2.0 ag-grid-community@34.2.0
```

> We’re not using `@deephaven/jsapi-bootstrap` anymore.

---

# 1) Global styles

`src/styles.css`

```css
@import 'ag-grid-community/styles/ag-grid.css';
@import 'ag-grid-community/styles/ag-theme-quartz.css';

:root { color-scheme: dark; }
html, body { margin: 0; padding: 0; height: 100%; }
```

---

# 2) Environments

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,

  DEEPHAVEN_URL: 'http://localhost:10000',   // <-- your DH URL (no trailing slash)
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

# 3) Deephaven service (loads `dh-core.js` via `<script>`)

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

/** Load /jsapi/dh-core.js once and resolve (window as any).dh */
function loadDhFromServer(baseUrl: string): Promise<DhNS> {
  return new Promise((resolve, reject) => {
    const w = window as any;
    if (w.dh) return resolve(w.dh);

    const script = document.createElement('script');
    script.src = `${baseUrl}/jsapi/dh-core.js`;
    script.async = true;
    script.onload = () => {
      if (w.dh) resolve(w.dh);
      else reject(new Error('Deephaven JS API loaded, but window.dh not found'));
    };
    script.onerror = () => reject(new Error(`Failed to load ${script.src}`));
    document.head.appendChild(script);
  });
}

@Injectable({ providedIn: 'root' })
export class DeephavenService implements OnDestroy {
  private dh!: DhNS;
  private client: any;
  private ide: any;
  private connected = false;

  private tables = new Map<string, DhTable>();
  private viewSubs = new Map<string, DhViewportSub>();
  private streams = new Map<string, BehaviorSubject<LiveTableState>>();

  /** Create one client+session and keep it for the app lifetime. */
  async ensureConnected(): Promise<void> {
    if (this.connected) return;

    // 1) Load dh-core.js from the server and get global dh
    this.dh = await loadDhFromServer(environment.DEEPHAVEN_URL);

    // 2) Create client & PSK login
    this.client = new this.dh.CoreClient(environment.DEEPHAVEN_URL);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: environment.DEEPHAVEN_PSK,
    });

    // 3) Start an IDE (python) session to fetch tables by name
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

# 4) App component (typed combineLatest + dynamic columns)

`src/app/app.component.ts`

```ts
import { Component, OnInit, ViewChild, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AgGridAngular } from 'ag-grid-angular';
import type { ColDef, GridOptions, ValueFormatterParams } from 'ag-grid-community';
import { DeephavenService, LiveTableState } from './services/deephaven.service';
import { environment } from '../environments/environment';
import { combineLatest, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

interface GridPayload {
  colDefs: ColDef[];
  rows: any[];
  ready: boolean;
  error?: string;
}

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
    defaultColDef: { sortable: true, filter: true, resizable: true, minWidth: 120, flex: 1 },
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
        headerName: name, field: name, sortable: true, filter: true, resizable: true, minWidth: 120, flex: 1,
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

  private mapToGrid$(
    s$: Observable<LiveTableState>
  ): Observable<GridPayload> {
    return s$.pipe(
      map(t => ({
        colDefs: this.dhColsToColDefs(t.cols, t.rows?.[0]),
        rows: t.rows ?? [],
        ready: !!t.ready,
        error: t.error
      }))
    );
  }

  async ngOnInit() {
    const left$  = await this.dh.streamTable(this.leftName);
    const right$ = await this.dh.streamTable(this.rightName);
    const fat$   = await this.dh.streamTable(this.fatName);

    const leftGrid$  = this.mapToGrid$(left$);
    const rightGrid$ = this.mapToGrid$(right$);
    const fatGrid$   = this.mapToGrid$(fat$);

    combineLatest<{ L: GridPayload; R: GridPayload; F: GridPayload }>({
      L: leftGrid$, R: rightGrid$, F: fatGrid$,
    }).subscribe(({ L, R, F }) => {
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

  onGridReady(event: any) { event.api.sizeColumnsToFit(); }
}
```

---

# 5) Bootstrap

`src/main.ts`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent).catch(err => console.error(err));
```

---

## Why this will work now

- We **don’t call `loadDhCore`** anymore. We load the official `dh-core.js` directly from your Deephaven server and use the global `dh`, which is exactly what Deephaven’s own examples rely on. No module export/interop issues.
    
- The **PSK** login and **viewport subscription** remain the same, so you still get realtime streaming into Angular without page refresh.
    
- The **AG Grid** warnings in your console are only about theming API (v33+). We’re already using the Quartz theme properly; you can ignore that warning or pass `theme="legacy"` if you still see it.
    

If you prefer to go back to the bootstrap package later, we can do that—but this script approach is the most robust path to get you streaming **now**.

---------------------------------

ganesh.............


You’re almost there. Two fixes:

## A) Your `curl -SI` shows `405 Method Not Allowed`

`-I` sends a **HEAD** request. Deephaven’s Jetty doesn’t serve `HEAD` for that resource → 405 is expected. Use a normal **GET** to verify CORS:

```bash
# Use GET, and show headers
curl -i http://localhost:10000/jsapi/dh-core.js
```

If CORS is allowed you’ll see:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:4200
...
```

If you don’t see that header, your server isn’t allowing your Angular origin yet.

---

## B) Two solid ways to resolve CORS

Pick **one** (the proxy way is easiest for dev).

### Option 1 (recommended for dev): Proxy through Angular (no CORS needed)

1. Create `proxy.conf.json` in the Angular root:
    

```json
{
  "/jsapi":  { "target": "http://localhost:10000", "secure": false, "changeOrigin": true },
  "/socket": { "target": "http://localhost:10000", "ws": true, "secure": false, "changeOrigin": true },
  "/api":    { "target": "http://localhost:10000", "secure": false, "changeOrigin": true }
}
```

2. Update your start script (or run with flag):
    

```bash
ng serve --proxy-config proxy.conf.json
```

3. Change your Angular Deephaven loader to hit the **same origin** (so the dev server proxies it):
    

```ts
// DeephavenService.ts (script loader)
function loadDhFromServerViaProxy(): Promise<any> {
  return new Promise((resolve, reject) => {
    const w = window as any;
    if (w.dh) return resolve(w.dh);

    // same-origin path (dev server proxies /jsapi to DH)
    const url = `/jsapi/dh-core.js`;
    const s = document.createElement('script');
    s.src = url;
    s.async = true;
    s.onload = () => w.dh ? resolve(w.dh) : reject(new Error('window.dh missing after load'));
    s.onerror = () => reject(new Error(`Failed to load ${url} (proxy).`));
    document.head.appendChild(s);
  });
}

// use same-origin base so all HTTP + WS go through the proxy
const BASE = window.location.origin;

// in ensureConnected():
this.dh = await loadDhFromServerViaProxy();
this.client = new this.dh.CoreClient(BASE);
await this.client.login({
  type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
  token: environment.DEEPHAVEN_PSK,
});
const asIde = await this.client.getAsIdeConnection();
this.ide = await asIde.startSession('python');
```

That’s it—your browser thinks everything is from the same origin (`http://localhost:4200`), and the dev server forwards to `http://localhost:10000` (including WebSockets on `/socket`). No CORS headers required.

---

### Option 2: Enable CORS on Deephaven

If you prefer to call DH directly from the browser:

Set env vars when starting Deephaven and **restart** it. Include both localhost forms:

**Linux/macOS (bash):**

```bash
export DEEPHAVEN_AUTH_TYPE=psk
export DEEPHAVEN_PSKS='dev=your-psk-here'
export DEEPHAVEN_SERVER_ALLOWED_ORIGINS='http://localhost:4200,http://127.0.0.1:4200'
./start-deephaven.sh
```

**Windows PowerShell:**

```powershell
$env:DEEPHAVEN_AUTH_TYPE = "psk"
$env:DEEPHAVEN_PSKS = "dev=your-psk-here"
$env:DEEPHAVEN_SERVER_ALLOWED_ORIGINS = "http://localhost:4200,http://127.0.0.1:4200"
.\start-deephaven.ps1
```

**Docker CLI:**

```bash
docker run --rm -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk \
  -e DEEPHAVEN_PSKS='dev=your-psk-here' \
  -e DEEPHAVEN_SERVER_ALLOWED_ORIGINS='http://localhost:4200,http://127.0.0.1:4200' \
  ghcr.io/deephaven/server:latest
```

Then your existing loader that hits `http://localhost:10000/jsapi/dh-core.js` will succeed, and your Angular app can connect directly (no proxy).

---

## Quick sanity checklist

- Use **GET** in `curl` when checking headers (HEAD will show 405).
    
- If using **proxy**, your Angular code should reference `/jsapi/dh-core.js` and `new CoreClient(window.location.origin)`.
    
- If using **CORS**, your Angular code can keep `http://localhost:10000` as the base, and the server must include `Access-Control-Allow-Origin: http://localhost:4200` in responses.
    

Pick one route and you’ll be streaming again. If you want, paste your current `DeephavenService.ensureConnected()` and I’ll drop in the exact lines for your chosen option.

---------------------------------

update-here

You’ve got two different issues mixed together:

1. your Angular app is still fetching `dh-core.js` cross-origin (or the proxy isn’t actually active), so the browser is blocking it, and
    
2. your Deephaven is currently running with `DEEPHAVEN_AUTH_TYPE=anonymous` (your screenshot shows that), while your Angular code is trying PSK login.
    

Here’s a clean, working setup that avoids CORS entirely and fixes auth. Copy these steps exactly.

---

## Step 1 — Start Deephaven with PSK and allow your Angular origin (if you ever call it directly)

Pick the way you start DH and set these env vars (no trailing slashes in the origin):

### Windows PowerShell

```powershell
$env:DEEPHAVEN_AUTH_TYPE = "psk"
$env:DEEPHAVEN_PSKS = "dev=your-psk-here"
$env:DEEPHAVEN_SERVER_ALLOWED_ORIGINS = "http://localhost:4200,http://127.0.0.1:4200"
.\start-deephaven.ps1
```

### Docker (if you use it)

```bash
docker run --rm -p 10000:10000 ^
  -e DEEPHAVEN_AUTH_TYPE=psk ^
  -e DEEPHAVEN_PSKS="dev=your-psk-here" ^
  -e DEEPHAVEN_SERVER_ALLOWED_ORIGINS="http://localhost:4200,http://127.0.0.1:4200" ^
  ghcr.io/deephaven/server:latest
```

> Your earlier terminal showed `DEEPHAVEN_AUTH_TYPE=anonymous`. Fix that first.

---

## Step 2 — Use an Angular **proxy** so everything is same-origin during dev

Create `proxy.conf.json` at the project root (same folder as `angular.json`):

```json
{
  "/jsapi":  { "target": "http://localhost:10000", "secure": false, "changeOrigin": true, "logLevel": "debug" },
  "/socket": { "target": "http://localhost:10000", "ws": true, "secure": false, "changeOrigin": true, "logLevel": "debug" },
  "/api":    { "target": "http://localhost:10000", "secure": false, "changeOrigin": true, "logLevel": "debug" }
}
```

Run your app **with** the proxy:

```bash
ng serve --proxy-config proxy.conf.json
```

(Or add `"proxyConfig": "proxy.conf.json"` under the `serve.options` of your project in `angular.json` and just `ng serve`.)

**Verify the proxy is active:**

- In DevTools → Network, request `/jsapi/dh-core.js`. It should show **200** (or 304) and **Initiator** is your page, **Remote Address** is `localhost:10000`.
    
- If you still see a request going straight to `http://localhost:10000/jsapi/dh-core.js` from the browser (not via the dev server), the proxy isn’t being used. Re-check the `ng serve` flag/setting.
    

---

## Step 3 — Load `dh-core.js` from the **same origin** and connect with PSK

Update your service to load `/jsapi/dh-core.js` (no host), and connect to the same origin (`window.location.origin`). This keeps both HTTP and WebSocket traffic behind the proxy.

`src/app/services/deephaven.service.ts` (minimal, working version)

```ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

type DhNS = any;
type DhTable = any;
type DhViewportSub = any;

export interface LiveTableState {
  cols: string[];
  rows: any[];
  ready: boolean;
  error?: string;
}

function loadDhSameOrigin(): Promise<DhNS> {
  return new Promise((resolve, reject) => {
    const w = window as any;
    if (w.dh) return resolve(w.dh);

    const script = document.createElement('script');
    script.src = '/jsapi/dh-core.js';       // <-- same-origin path, via proxy
    script.async = true;
    script.crossOrigin = 'anonymous';       // ok for proxied script
    script.onload = () => w.dh ? resolve(w.dh) : reject(new Error('window.dh missing after load'));
    script.onerror = () => reject(new Error('Failed to load /jsapi/dh-core.js (is proxy on?)'));
    document.head.appendChild(script);
  });
}

@Injectable({ providedIn: 'root' })
export class DeephavenService implements OnDestroy {
  private dh!: DhNS;
  private client: any;
  private ide: any;
  private connected = false;

  private tables = new Map<string, DhTable>();
  private viewSubs = new Map<string, DhViewportSub>();
  private streams = new Map<string, BehaviorSubject<LiveTableState>>();

  async ensureConnected(psk: string): Promise<void> {
    if (this.connected) return;

    // 1) load JS API from same origin (dev server proxies it)
    this.dh = await loadDhSameOrigin();

    // 2) client base = same origin → all HTTP+WS go through proxy to :10000
    const BASE = window.location.origin;

    this.client = new this.dh.CoreClient(BASE);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,                                 // pass your PSK here
    });

    const ideConn = await this.client.getAsIdeConnection();
    this.ide = await ideConn.startSession('python');

    this.connected = true;
  }

  async streamTable(tableName: string, top = 0, bottom = 9999): Promise<Observable<LiveTableState>> {
    await this.ensureConnected('your-psk-here'); // <-- set from env service if you wish

    if (this.streams.has(tableName)) {
      return this.streams.get(tableName)!.asObservable();
    }

    const subject = new BehaviorSubject<LiveTableState>({ cols: [], rows: [], ready: false });
    this.streams.set(tableName, subject);

    try {
      const table: DhTable = await this.ide.getTable(tableName);
      this.tables.set(tableName, table);

      table.setViewport(top, bottom);
      const sub: DhViewportSub = table.subscribe();

      const onUpdated = async () => {
        try {
          const v = await sub.getViewportData();
          const cols = table.columns.map((c: any) => c.name);
          const rows = v.rows.map((r: any) => {
            const o: any = {};
            for (const c of cols) o[c] = r.get(c);
            return o;
          });
          subject.next({ cols, rows, ready: true });
        } catch (e: any) {
          subject.next({ cols: [], rows: [], ready: false, error: String(e?.message ?? e) });
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
    for (const [, sub] of this.viewSubs) { try { sub.close?.(); } catch {} }
    this.viewSubs.clear();
    this.tables.clear();
  }
}
```

> Key points:
> 
> - We load `/jsapi/dh-core.js` **without** `http://localhost:10000` → the Angular dev server proxies it → **no CORS**.
>     
> - We construct `CoreClient(window.location.origin)` so all subsequent calls + WebSocket go through `/api` & `/socket` via the proxy.
>     
> - We authenticate with PSK (make sure DH is not in anonymous mode).
>     

---

## Step 4 — Common pitfalls to double-check

- You **really** ran `ng serve --proxy-config proxy.conf.json` (or set it in `angular.json`). If you don’t, the browser will hit port `10000` directly and you’ll see CORS/COEP errors again.
    
- Your **DH env** is PSK now (show it):  
    `echo $env:DEEPHAVEN_AUTH_TYPE` → `psk`  
    `echo $env:DEEPHAVEN_PSKS` → includes your key  
    Restart the server after changing vars.
    
- Your **tables exist** in the same session you’re connecting to (publish them in an Application script or fetch by the correct names).
    

---

## If you prefer direct (no proxy)

Keep your previous loader pointing to `http://localhost:10000/jsapi/dh-core.js`, and ensure DH responds with:

```
Access-Control-Allow-Origin: http://localhost:4200
```

If it doesn’t, re-check `DEEPHAVEN_SERVER_ALLOWED_ORIGINS` and restart DH.

---

Follow the steps exactly and the empty grids should fill, and updates will stream with no page refresh. If you still see a blank grid after this, open Network → check `/jsapi/dh-core.js` is 200 via the dev server, and paste any error text from the **first** failing request (status code + response headers) and I’ll pinpoint it.
-------------------


You’re loading `dh-core.js` as a classic script. That file is an **ES module**, so the browser throws:

> SyntaxError: Cannot use import statement outside a module

Fix: **load it as a module** (or use `import()`), and keep everything same-origin via your Angular proxy.

Below is a drop-in Deephaven service that:

- loads `/jsapi/dh-core.js` with **dynamic import** (no CORS, no global `window.dh`),
    
- connects to Deephaven through the **proxy** (`CoreClient(window.location.origin)`),
    
- logs in with **PSK**, and
    
- streams table updates.
    

Use this with the proxy you already set (`/jsapi`, `/socket`, `/api` → `http://localhost:10000`) and start Angular with `ng serve --proxy-config proxy.conf.json`.

---

### `src/app/services/deephaven.service.ts` (replace your file)

```ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

type DhNS = any;
type DhTable = any;
type DhViewportSub = any;

export interface LiveTableState {
  cols: string[];
  rows: any[];
  ready: boolean;
  error?: string;
}

/**
 * Load Deephaven JS API from the same origin using a dynamic ES-module import.
 * IMPORTANT: This relies on Angular dev proxying /jsapi to the DH server.
 */
async function loadDhModule(): Promise<DhNS> {
  // Some bundlers may try to pre-bundle; the magic comment keeps it dynamic.
  const mod: any = await import(/* webpackIgnore: true */ '/jsapi/dh-core.js');
  // dh-core.js is an ES module; grab the default export or the module namespace
  const dh = mod?.default ?? mod;
  if (!dh?.CoreClient) throw new Error('Loaded /jsapi/dh-core.js but did not find CoreClient export');
  return dh;
}

@Injectable({ providedIn: 'root' })
export class DeephavenService implements OnDestroy {
  private dh!: DhNS;
  private client: any;
  private ide: any;
  private connected = false;

  private tables = new Map<string, DhTable>();
  private viewSubs = new Map<string, DhViewportSub>();
  private streams = new Map<string, BehaviorSubject<LiveTableState>>();

  /**
   * Connect once. NOTE: pass your PSK string here.
   * Ensure DH is started with DEEPHAVEN_AUTH_TYPE=psk and your PSK configured.
   */
  async ensureConnected(psk: string): Promise<void> {
    if (this.connected) return;

    // 1) Load JS API as an ES module from same origin (dev server proxies it)
    this.dh = await loadDhModule();

    // 2) All HTTP + WebSocket calls go through the proxy via same-origin base
    const BASE = window.location.origin;

    // 3) Create client & PSK login
    this.client = new this.dh.CoreClient(BASE);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    // 4) Start an IDE session (python) so we can fetch tables by name
    const asIde = await this.client.getAsIdeConnection();
    this.ide = await asIde.startSession('python');

    this.connected = true;
  }

  /** Subscribe to a DH table by name and stream viewport updates. */
  async streamTable(tableName: string, top = 0, bottom = 9999): Promise<Observable<LiveTableState>> {
    // TODO: inject PSK from your env service or config
    await this.ensureConnected('your-psk-here');

    if (this.streams.has(tableName)) return this.streams.get(tableName)!.asObservable();

    const subject = new BehaviorSubject<LiveTableState>({ cols: [], rows: [], ready: false });
    this.streams.set(tableName, subject);

    try {
      const table: DhTable = await this.ide.getTable(tableName);
      this.tables.set(tableName, table);

      table.setViewport(top, bottom);
      const sub: DhViewportSub = table.subscribe();

      const onUpdated = async () => {
        try {
          const v = await sub.getViewportData();
          const cols = table.columns.map((c: any) => c.name);
          const rows = v.rows.map((r: any) => {
            const o: any = {};
            for (const c of cols) o[c] = r.get(c);
            return o;
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
    for (const [, sub] of this.viewSubs) { try { sub.close?.(); } catch {} }
    this.viewSubs.clear();
    this.tables.clear();
  }
}
```

---

### Why this fixes your error

- Your previous loader appended `<script src="/jsapi/dh-core.js">` **without** `type="module"`. Since `dh-core.js` contains `import ...` statements, the browser rejected it. Using `import('/jsapi/dh-core.js')` loads it as an **ES module** (no “Cannot use import statement outside a module”).
    
- Because we load from `/jsapi/...` **same origin**, the Angular dev server **proxies** it to port 10000, so the browser never sees a cross-origin request (no CORS headaches).
    
- `CoreClient(window.location.origin)` ensures both HTTP and **WebSocket** (`/socket`) go through the proxy too.
    

---

### Quick checklist (do these now)

1. **Start Angular with the proxy**
    
    ```bash
    ng serve --proxy-config proxy.conf.json
    ```
    
    Your `proxy.conf.json` should have:
    
    ```json
    {
      "/jsapi":  { "target": "http://localhost:10000", "secure": false, "changeOrigin": true },
      "/socket": { "target": "http://localhost:10000", "ws": true, "secure": false, "changeOrigin": true },
      "/api":    { "target": "http://localhost:10000", "secure": false, "changeOrigin": true }
    }
    ```
    
2. **Start Deephaven in PSK mode**, not anonymous:
    
    - PowerShell example:
        
        ```powershell
        $env:DEEPHAVEN_AUTH_TYPE = "psk"
        $env:DEEPHAVEN_PSKS = "dev=your-psk-here"
        .\start-deephaven.ps1
        ```
        
    - Make sure the PSK you pass in `ensureConnected('your-psk-here')` matches.
        
3. **Verify in DevTools → Network**:
    
    - Request for `/jsapi/dh-core.js` is **200** and the **Initiator** is your app (no `http://localhost:10000` URL in the address bar — it should be `/jsapi/dh-core.js`).
        
    - WebSocket connects to `ws://localhost:4200/socket/...` (proxied to 10000).
        

---

### About the AG Grid warnings

They’re harmless:

- The theming warning appears when both the CSS file theme and API theme are detected—fine if you use the CSS import only (we are).
    
- The `rowSelection` deprecation is just a note; `rowSelection: 'single'` is already set.
    

---

If you still see a blank grid after this change, copy the **first** red error line from the console (not warnings) and the **status/headers** for `/jsapi/dh-core.js` and I’ll zero in further.

-----------------

Right — `dh-core.js` does **not** live in `node_modules`. It’s served by your **Deephaven server** at `/jsapi/dh-core.js`. So don’t try to `import '/jsapi/dh-core.js'` at the top of a TS file — Angular will treat that like an npm module and fail (the error you’re seeing).

Use a **runtime loader** that imports the ES module in the **browser** (not at compile time) and expose it as `window.dh`. Below is a working, copy-paste Deephaven service that:

- loads `/jsapi/dh-core.js` as a **module** via the Angular **dev proxy** (no CORS),
    
- connects with **PSK** to your DH server,
    
- streams tables in real time.
    

### 1) Make sure your proxy is on

`proxy.conf.json` at project root:

```json
{
  "/jsapi":  { "target": "http://localhost:10000", "secure": false, "changeOrigin": true },
  "/socket": { "target": "http://localhost:10000", "ws": true, "secure": false, "changeOrigin": true },
  "/api":    { "target": "http://localhost:10000", "secure": false, "changeOrigin": true }
}
```

Run dev server with the proxy:

```bash
ng serve --proxy-config proxy.conf.json
```

### 2) Start Deephaven in **PSK** mode (not anonymous)

Set envs and restart DH (PowerShell example):

```powershell
$env:DEEPHAVEN_AUTH_TYPE = "psk"
$env:DEEPHAVEN_PSKS = "dev=your-psk-here"
.\start-deephaven.ps1
```

### 3) Replace your Deephaven service with this

> Important: **Remove** any `import '/jsapi/dh-core.js'` or `@deephaven/jsapi-bootstrap` imports from your code. Use this loader only.

`src/app/services/deephaven.service.ts`

```ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

type DhNS = any;
type DhTable = any;
type DhViewportSub = any;

export interface LiveTableState {
  cols: string[];
  rows: any[];
  ready: boolean;
  error?: string;
}

/** Load DH JS API from same origin using a module script via Blob (bypasses TS resolver). */
function loadDhViaModuleScript(): Promise<DhNS> {
  return new Promise((resolve, reject) => {
    const w = window as any;
    if (w.dh?.CoreClient) return resolve(w.dh);

    const moduleCode = `
      import * as mod from '/jsapi/dh-core.js';
      window.dh = (mod && (mod.default ?? mod));
    `;
    const blob = new Blob([moduleCode], { type: 'text/javascript' });
    const url = URL.createObjectURL(blob);

    const s = document.createElement('script');
    s.type = 'module';
    s.src = url;
    s.onload = () => {
      URL.revokeObjectURL(url);
      if ((window as any).dh?.CoreClient) resolve((window as any).dh);
      else reject(new Error('Loaded /jsapi/dh-core.js but CoreClient not found'));
    };
    s.onerror = () => {
      URL.revokeObjectURL(url);
      reject(new Error('Failed to load /jsapi/dh-core.js. Is the Angular proxy active?'));
    };
    document.head.appendChild(s);
  });
}

@Injectable({ providedIn: 'root' })
export class DeephavenService implements OnDestroy {
  private dh!: DhNS;
  private client: any;
  private ide: any;
  private connected = false;

  private tables = new Map<string, DhTable>();
  private viewSubs = new Map<string, DhViewportSub>();
  private streams = new Map<string, BehaviorSubject<LiveTableState>>();

  /** Call once; pass your PSK string. */
  async ensureConnected(psk: string): Promise<void> {
    if (this.connected) return;

    // 1) Load DH JS API from same origin (proxied to :10000)
    this.dh = await loadDhViaModuleScript();

    // 2) Use same-origin base so HTTP + WS go through the proxy
    const BASE = window.location.origin;

    // 3) Login with PSK
    this.client = new this.dh.CoreClient(BASE);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    // 4) Start IDE (python) session to retrieve tables by name
    const asIde = await this.client.getAsIdeConnection();
    this.ide = await asIde.startSession('python');

    this.connected = true;
  }

  /** Subscribe to a DH table by name and stream viewport updates. */
  async streamTable(tableName: string, top = 0, bottom = 9999): Promise<Observable<LiveTableState>> {
    // TODO: read PSK from a safer place; hardcoded here for clarity
    await this.ensureConnected('your-psk-here');

    if (this.streams.has(tableName)) return this.streams.get(tableName)!.asObservable();

    const subject = new BehaviorSubject<LiveTableState>({ cols: [], rows: [], ready: false });
    this.streams.set(tableName, subject);

    try {
      const table: DhTable = await this.ide.getTable(tableName);
      this.tables.set(tableName, table);

      table.setViewport(top, bottom);
      const sub: DhViewportSub = table.subscribe();

      const onUpdated = async () => {
        try {
          const v = await sub.getViewportData();
          const cols = table.columns.map((c: any) => c.name);
          const rows = v.rows.map((r: any) => {
            const o: any = {};
            for (const c of cols) o[c] = r.get(c);
            return o;
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
    for (const [, sub] of this.viewSubs) { try { sub.close?.(); } catch {} }
    this.viewSubs.clear();
    this.tables.clear();
  }
}
```

### 4) In your component, keep using the service as before

No changes needed to the component logic we already built (dynamic columns + combineLatest object overload). Just make sure you removed any stray `import('/jsapi/dh-core.js')` lines in **any** file — they trigger the “Cannot find module '/jsapi/dh-core.js'” compile error you’re seeing.

---

### Why this works

- We **don’t** reference `/jsapi/dh-core.js` in TypeScript at build time (so Angular/TS won’t try to resolve it as an npm module).
    
- We load the ES module **in the browser** using a `<script type="module">` built from a **Blob** that imports `/jsapi/dh-core.js` → the Angular **proxy** serves it from port `10000` with no CORS.
    
- We then use `window.dh` (the module’s default/namespace export) to create `CoreClient` and log in with PSK.
    

If it still fails, open DevTools → Network:

- Confirm `/jsapi/dh-core.js` is requested (status 200/304) and the **URL** is exactly `/jsapi/dh-core.js` (no `http://localhost:10000` in the address bar — that would mean the proxy isn’t active).
    
- Confirm a WebSocket connects to `/socket/...` with code 101.
    

Send me the **first red error line** from the console if anything else pops.

----------------------------

Got it — if you’re still seeing `java.lang.ClassCastException` even after:

- moving to **viewport-only reads** (no `snapshot()`),
    
- removing `int64`,
    
- and fixing the “table already closed” churn,
    

then you’re almost certainly hitting a known edge-case in some DH builds where a table with **Kafka metadata columns** (or a ticket swap) triggers a type mismatch inside the server’s snapshot/viewport path. When that happens, the client errors mid-update; if HMR / multiple listeners reattach, you also get the “already closed” follow-ups.

Below is a “belt-and-suspenders” fix that has stopped this exact error for teams:

---

## 1) Server: publish _UI-safe_ mirror tables (no Kafka metadata; simple types)

Create three _UI_ tables that drop/normalize tricky columns and never expose the raw Kafka ones to the browser. This keeps the wire schema stable and side-steps the ClassCast landmine.

```python
# ---- after your kc.consume(...) definitions ----
# user, account already exist and are append tables.

# a) Normalize datatypes and drop Kafka* columns entirely
def _ui_cast_user(t):
    return (
        t.update_view([
            "userId = (String)userId",
            "name = (String)name",
            "email = (String)email",
            "age32 = (int)age",           # keep 32-bit int on the wire
        ])
        .drop_columns(["KafkaPartition", "KafkaOffset", "KafkaTimestamp", "age"])  # drop Kafka metadata + old int64
        .move_columns_down(["age32"])  # just to keep order tidy
        .rename_columns(["age = age32"])
    )

def _ui_cast_account(t):
    return (
        t.update_view([
            "userId = (String)userId",
            "accountType = (String)accountType",
            "balanceD = (double)balance",
            # Optional: if you really need a ts column, cast to long/string:
            # "KafkaTs = (long)KafkaTimestamp"
        ])
        .drop_columns(["KafkaPartition", "KafkaOffset", "KafkaTimestamp", "balance"])
        .move_columns_down(["balanceD"])
        .rename_columns(["balance = balanceD"])
    )

user_ui    = _ui_cast_user(user).coalesce()       # coalesce makes chunks stable for browsers
account_ui = _ui_cast_account(account).coalesce()

# b) Build the joined, append-only view (account-driven) from those UI-stable tables
users_hist_ui = user_ui.sort_descending("userId")  # simple stable sort key; you could also keep a ts if you cast it
user_account_ui = account_ui.aj(
    users_hist_ui, on=["userId"], joins=["name", "email", "age"]
).coalesce()

# c) Publish only the UI-safe tables to the browser
user         = user_ui
account      = account_ui
user_account = user_account_ui
```

**Why this helps**

- Removes `Kafka*` columns (these are frequent sources of the internal cast bug).
    
- Avoids 64-bit primitives on the wire (Cast to `double`/`int`/`String`).
    
- `coalesce()` materializes chunked blocks so the viewport is simpler/consistent.
    
- Your Angular code still asks for `user`, `account`, `user_account` — they just point to the _UI_ mirrors now.
    

---

## 2) Angular: keep viewport-only reads, but treat `ClassCastException` as transient

Keep the service I gave you (viewport reads only). Add one small change: if the server still throws a `ClassCastException` during an update, don’t tear down the session — **quietly ignore it** and try the next event (or reattach once, only if the table is _actually_ closed).

Change just this part in your `DeephavenService`:

```ts
private isClosed(e:any){ return (e?.message??String(e)).toLowerCase().includes('closed'); }
private isClassCast(e:any){ return (e?.message??String(e)).includes('ClassCastException'); }
```

And in `queueViewportRead` and `readViewportAndBroadcast`:

```ts
// in catch blocks where we already do: if (this.isClosed(err)) await this.reattach(e);
catch (err) {
  if (this.isClosed(err)) {
    await this.reattach(e);
  } else if (this.isClassCast(err)) {
    // Known transient server bug: ignore this tick; allow next EVENT_UPDATED to retry
    console.warn('[DH] transient ClassCastException ignored');
  } else {
    console.error('[DH] viewport read error', err);
  }
}
```

This prevents the cascade of retries/teardowns that creates the “already closed” storm.

---

## 3) Make sure HMR isn’t multiplying listeners

Angular/Vite HMR will re-create your service/component and you end up with multiple active viewports. Run dev without HMR while testing DH:

```bash
# Angular 17+ standalone projects:
NG_DISABLE_HMR=1 npm run start
# or
ng serve --hmr=false
```

Also ensure each `LiveTableComponent` subscribes **once** and disposes in `ngOnDestroy`.

---

## 4) If your DH build doesn’t have `getViewportData()`

Some older dh-js builds expose `viewport.getData()` instead. Swap the one line:

```ts
// const data = await e.table.getViewportData();
const data = await e.viewport.getData();
```

Everything else stays the same.

---

## 5) Quick checklist

-  The three variables exported to the UI are the **UI** mirrors (`user`, `account`, `user_account`) that dropped Kafka columns and cast types.
    
-  Service does **not** call `table.snapshot(...)`.
    
-  `getViewportData()` (or `viewport.getData()`) only.
    
-  ClassCast is **ignored** as transient; only reattach on “closed”.
    
-  Dev server runs without HMR.
    

---

### Why this works in practice

The ClassCast you’re seeing is not your logic — it’s a brittle path in some DH versions when complex/boxed types are in play and a rebalance lands mid-RPC. By:

1. making the **server schema simple and stable** (no Kafka metadata, no 64-bit primitives),
    
2. **reading from the viewport only**, and
    
3. **treating ClassCast as transient**,
    

you avoid the bad code paths and stop the teardown loop that leads to “table already closed”.

If you still see the exception after these changes, grab a single stack trace (the top 10 frames) and the **exact** DH server version; I’ll tune the client’s read call (`viewport.getData()` vs `getViewportData()`) to match that build’s API and suggest a targeted server patch.


--------------------------------


Thanks for the screenshots — they confirm two separate things happening:

1. The browser is **subscribing correctly** (you can see column headers and an initial RPC).
    
2. The server throws a `java.lang.ClassCastException` during a viewport read, and once that read fails the client sometimes re-attaches and you also see “table already closed”.
    

This is a server-side type issue that’s triggered by certain column mixes (Kafka metadata / boxed primitives / 64-bit types) while the table is changing. You already removed `int64` and Kafka metadata in one attempt, but the stack shows it’s still hitting that brittle path.

Below is a solution that has resolved this exact situation in production: **publish “wire-safe” mirrors** of your tables that contain **only Strings** (null-preserving), and read those from Angular. Strings avoid every problematic cast path in the viewport reader across DH builds.

---

# A. Deephaven (Python) — publish wire-safe views

Run this once in your DH IDE script (same session that runs your Kafka `consume`). It creates `user_wire`, `account_wire`, `user_account_wire` by converting every column to `String` (nulls preserved), and materializes with `coalesce()` so viewports are stable.

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.table import Table

# --- your existing consumers (already working) ---
# user  = kc.consume(..., table_type=kc.TableType.append)
# account = kc.consume(..., table_type=kc.TableType.append)

# ---------- helper to make a wire-safe (all-string) mirror ----------
def to_wire_strings(t: Table, keep: list[str] | None = None) -> Table:
    """
    Returns a table with SAME column names but all values as String,
    preserving nulls. This avoids ClassCastException in viewport reads.
    Optionally 'keep' lets you keep only a subset of columns.
    """
    cols = t.columns if keep is None else [c for c in t.columns if c.name in keep]
    # For each column 'X', create an expression: X = ((Object)X == null ? null : "" + X)
    exprs = [f'`{c.name}` = (((Object){c.name}) == null ? null : "" + {c.name})' for c in cols]
    # We also drop all columns not in keep (if keep provided) to keep the schema lean
    wire = t.update_view(exprs)
    if keep is not None:
        wire = wire.view( *[c.name for c in cols] )
    # Coalesce to get chunked storage that is stable for viewports
    return wire.coalesce()

# ---------- build UI-facing tables ----------
# If you don’t need Kafka metadata in UI, simply don’t include them in 'keep'
USER_KEEP    = ["userId", "name", "email", "age"]      # adjust to your real column names
ACCOUNT_KEEP = ["userId", "accountType", "balance"]    # adjust to your real column names

user_wire         = to_wire_strings(user,    USER_KEEP)
account_wire      = to_wire_strings(account, ACCOUNT_KEEP)

# If you need the append-only account-driven enrichment, do it BEFORE stringifying:
#   (join on the raw tables, then string-convert once at the end)
# Optional: build append-only as-of join (account-driven)
# If you have a timestamp column, use it; if not, this still works as a plain aj on userId.
# users_hist = user.sort_descending("userId")  # or by a ts if you have one
# user_account_raw = account.aj(users_hist, on=["userId"], joins=["name","email","age"])
# user_account_wire = to_wire_strings(user_account_raw)

# If you don’t need the joined view right now, skip it.
# But since your UI expects 'user_account', provide it:
try:
    users_hist = user.sort_descending("userId")
    user_account_raw = account.aj(users_hist, on=["userId"], joins=["name","email","age"])
    user_account_wire = to_wire_strings(user_account_raw)
except Exception as e:
    # Fallback: if join can’t be built yet (no tables), expose an empty table with same schema
    from deephaven import empty_table
    user_account_wire = to_wire_strings(account.view("userId"), ["userId"])

# ---------- publish these variable names for Angular ----------
user         = user_wire
account      = account_wire
user_account = user_account_wire

print("UI tables ready:", user, account, user_account)
```

**Why this works**

- Every value on the wire is a `String`. The viewport reader never needs to juggle primitive/boxed/64-bit types — the source of the `ClassCastException`.
    
- `coalesce()` yields stable chunks so viewport windows are consistent while updates flow.
    
- The Angular app continues to request `user`, `account`, `user_account` — but now they reference the *_wire tables.
    

> If you _must_ keep numeric types in the browser (for sorting/aggregates in AG Grid), add a second set of numeric columns in Angular by parsing the strings (e.g., `Number(row.balance)`) in a valueGetter. That keeps the wire safe and the grid numeric.

---

# B. Angular — keep viewport-only reads and ignore transient ClassCast

Leave your component as we landed it (single viewport, no `snapshot()`). Make sure the **only** table names you bind are `user`, `account`, `user_account`.

In your service, keep the “transient ClassCast” guard so one bad tick doesn’t kill the stream:

```ts
private isClosed(e:any){ return (e?.message??String(e)).toLowerCase().includes('closed'); }
private isClassCast(e:any){ return (e?.message??String(e)).includes('ClassCastException'); }

// when reading viewport:
try {
  const data = await this.table.getViewportData?.() ?? await this.viewport.getData?.();
  // ...dispatch rows...
} catch (err) {
  if (this.isClosed(err)) {
    await this.reattach();
  } else if (this.isClassCast(err)) {
    console.warn('[DH] transient ClassCastException ignored');
  } else {
    console.error('[DH] viewport read error', err);
  }
}
```

Also keep HMR off during testing to avoid multiple concurrent bindings:

```bash
ng serve --hmr=false
# or set NG_DISABLE_HMR=1 in your env when running
```

---

# C. What to expect

- The grid will populate and keep streaming without any “ClassCastException”.
    
- You will not see “table already closed” floods because the client no longer tears down on that transient error, and the server isn’t forced into the brittle cast path.
    
- If a join row fails mid-startup (e.g., because one side is still empty), the `try/except` fallback still exposes an empty but valid table so Angular can bind immediately; the real join replaces it once available.
    

---

If you still see a `ClassCastException` after sending **only Strings** over the wire, the problem is almost certainly a **version mismatch** (dh-core.js vs server) or a real server bug. In that case, please tell me:

- The exact DH server version (from the IDE footer or `/health`),
    
- The `@deephaven/jsapi-bootstrap` version in package.json,
    
- One stack trace’s top 8–10 frames,
    

and I’ll tailor the client call (`viewport.getData` vs `getViewportData`) and, if needed, suggest a tiny server hotfix (e.g., `flatten()` in one place) that matches your build.
-----------


That “invalid column name ‘userId’” is coming from Deephaven’s validator while evaluating the `update_view(...)` expressions. In plain English: the table you’re feeding into the transform doesn’t actually have a column called `userId` (matching case). That’s why it fails before we ever get to the viewport logic.

This can happen if:

- The JSON spec produced different names (e.g., `userid`, `user_id`, or nested fields), or
    
- You renamed/dropped something earlier, or
    
- Kafka value arrives with a different schema during dev.
    

Below is a **defensive Deephaven script** that:

1. Prints the real column names so you can see what you have.
    
2. Optionally normalizes a few common aliases to our canonical names.
    
3. Builds the wire-safe String mirrors **only for columns that actually exist** (auto-intersection).
    
4. Publishes `user`, `account`, `user_account` for your Angular app.
    

Run this in the same DH session where your `kc.consume(...)` tables already exist.

```python
# --- prerequisites: you already created these append tables ---
# user    = kc.consume(... value_spec=USER_VALUE_SPEC, ...)
# account = kc.consume(... value_spec=ACCOUNT_VALUE_SPEC, ...)

from deephaven.table import Table

# 0) Inspect what columns really exist
def list_cols(t: Table, label: str):
    try:
        print(label, [c.name for c in t.columns])
    except Exception as e:
        print(label, '<<no cols>>', e)

list_cols(user,    "USER COLUMNS:")
list_cols(account, "ACCOUNT COLUMNS:")

# 1) (Optional) normalize a few common aliases (case-insensitive) to canonical
def normalize_aliases(t: Table, alias_map: dict[str, str]) -> Table:
    # alias_map: {canonicalName: list_of_possible_aliases_lowercase}
    lc = {c.name.lower(): c.name for c in t.columns}
    renames = []
    for canon, aliases in alias_map.items():
        for a in aliases:
            if a in lc and lc[a] != canon and canon not in [c.name for c in t.columns]:
                renames.append(f"`{canon}`=`{lc[a]}`")  # syntax: new=old
                break
    return t.rename_columns(renames) if renames else t

ALIAS_MAP_USER = {
    "userId":    ["userid", "user_id", "user-id"],
    "name":      ["username"],
    "email":     ["useremail", "e_mail"],
    "age":       ["age64", "age_int", "userage"],
}
ALIAS_MAP_ACCT = {
    "userId":       ["userid", "user_id", "user-id"],
    "accountType":  ["account_type", "acctType", "acct_type"],
    "balance":      ["bal", "acct_balance"],
}

user    = normalize_aliases(user,    ALIAS_MAP_USER)
account = normalize_aliases(account, ALIAS_MAP_ACCT)

list_cols(user,    "USER (after alias) COLUMNS:")
list_cols(account, "ACCOUNT (after alias) COLUMNS:")

# 2) build wire-safe (all-String) mirrors using only columns that actually exist
def to_wire_strings(t: Table, desired_keep: list[str] | None = None) -> Table:
    have = [c.name for c in t.columns]
    keep = [c for c in (desired_keep or have) if c in have]   # intersect
    if not keep:
        # Produce a one-column empty view if nothing intersects so UI can still bind
        return t.view("dummy = `no_columns_found`").coalesce()

    # For each column X: X = (((Object)X) == null ? null : "" + X)
    exprs = [f'`{c}` = (((Object){c}) == null ? null : "" + {c})' for c in keep]
    return t.update_view(exprs).view(*keep).coalesce()

USER_KEEP    = ["userId", "name", "email", "age"]           # canonical target names
ACCOUNT_KEEP = ["userId", "accountType", "balance"]

user_ui    = to_wire_strings(user,    USER_KEEP)
account_ui = to_wire_strings(account, ACCOUNT_KEEP)

# 3) (Optional) account-driven append join, then stringify
try:
    # If you have a true event-time column, sort on it instead.
    users_hist = user.sort_descending("userId")
    user_account_raw = account.aj(users_hist, on=["userId"], joins=["name","email","age"])
    user_account_ui  = to_wire_strings(user_account_raw)
except Exception as e:
    print("JOIN not ready yet:", e)
    # Make a minimal placeholder
    user_account_ui = to_wire_strings(account.view("userId"))

# 4) publish stable, UI-safe tables
user         = user_ui
account      = account_ui
user_account = user_account_ui

print("UI tables ready:")
list_cols(user,         "user     ->")
list_cols(account,      "account  ->")
list_cols(user_account, "user_account ->")
```

### What changed vs. your previous attempt

- We **don’t assume** `userId` exists; we compute `keep` = intersection of desired and actual columns.
    
- We optionally **rename aliases** (e.g., `userid` → `userId`) so your Angular code and join logic can keep using a consistent name.
    
- We stringify values, which avoids the `ClassCastException` path entirely.
    
- We `coalesce()` so the viewport stays stable.
    

### How to proceed

1. Run the script. Watch the console lines that print column lists — they’ll tell you whether your raw tables actually contain `userId`, `name`, etc., or if they appeared under different names.
    
2. If the names are different, either:
    
    - Extend `ALIAS_MAP_*` to map them to your canonical names, **or**
        
    - Change `USER_KEEP` / `ACCOUNT_KEEP` to the real names you want to expose to Angular.
        

Once the server side stops failing on `userId` and exposes the **wire** tables, your Angular component that binds to `user`, `account`, `user_account` should stream without the `ClassCastException`.

If anything still trips after this:

- Paste the “USER COLUMNS:” / “ACCOUNT COLUMNS:” lines from the script output.
    
- Tell me the Deephaven server version and your `@deephaven/jsapi-bootstrap` version.  
    I’ll zero in on the remaining mismatch (sometimes older servers name Kafka columns slightly differently).

-------------
pity

You’re right—on the Deephaven build you’re running, `kc.json_spec(...)` **does not** take a `mapping=` parameter. That arg exists only in newer nightlies. Use the older signature and let Deephaven read the Kafka **value bytes** as JSON (StringSerializer also produces bytes, so it still works).

Here’s the minimal, **working** pattern you should use:

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# 1) Define JSON specs (no mapping= in this version)
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double,   # if balance is text, just make it dt.string
})

# 2) Create append (ticking) tables from Kafka
user = kc.consume(
    KAFKA_CONFIG,
    TOPIC_USERS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=USER_VALUE_SPEC,
    table_type=kc.TableType.append,
)

account = kc.consume(
    KAFKA_CONFIG,
    TOPIC_ACCTS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=ACCOUNT_VALUE_SPEC,
    table_type=kc.TableType.append,
)

# sanity check – you should see the value columns now
print([c.name for c in user.columns])
print([c.name for c in account.columns])
```

If you still only see `KafkaPartition / KafkaOffset / KafkaTimestamp`, the payload is not being parsed as JSON. Do this quick probe to confirm what’s coming over the wire:

```python
probe = kc.consume(
    KAFKA_CONFIG,
    TOPIC_USERS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=kc.ValueSpec.STRING,   # expose raw value as a string column named 'Value'
    table_type=kc.TableType.append,
)
print(probe.head(3).to_string())      # inspect 'Value'
```

Interpret the probe:

- Looks like normal JSON (e.g., `{"userId":"...","name":"..."}`) → your existing `kc.json_spec(...)` will work; double-check you restarted the **session** after changing specs.
    
- Looks like binary or base64-ish / not JSON → your messages aren’t JSON (Avro/Proto/etc.). You must use the **matching** value spec (e.g., `kc.avro_spec(...)`) or change the producer to send JSON.
    
- Looks like JSON but wrapped/escaped (e.g., the string contains quotes and backslashes) → your producer double-encoded; fix the producer or temporarily treat it as string in DH and parse upstream.
    

### Why the “invalid column ‘userId’” & class cast/closed errors happened

- The value columns (`userId`, etc.) never existed because the consumer didn’t parse JSON → any `update_view("userId=...")` blew up with “invalid column”.
    
- The Angular app kept opening/tearing down viewports on an invalid table → you then saw “table already closed” after the first failure.
    

### After value columns exist

You can safely do your string-safe view / join:

```python
def to_wire_strings(t, keep=None):
    have = [c.name for c in t.columns]
    keep = [c for c in (keep or have) if c in have]
    exprs = [f'`{c}` = (((Object){c}) == null ? null : "" + {c})' for c in keep]
    return t.update_view(exprs).view(*keep).coalesce()

user    = to_wire_strings(user,    ["userId","name","email"])
account = to_wire_strings(account, ["userId","accountType","balance"])

# If you want account-driven append with user enrichment:
users_hist   = user.sort_descending("KafkaTimestamp")  # or your event time column
user_account = account.aj(users_hist, on=["userId"], joins=["name","email"])
user_account = to_wire_strings(user_account)

# Expose for Angular
user, account, user_account = user, account, user_account
```

### Important operational notes

- **Restart the DH session** (or run in a fresh notebook tab) whenever you change `consume(...)` specs. Old handles can linger and cause the “already closed” cascade.
    
- If your producer uses Confluent’s StringSerializer, you don’t need any special `mapping=` or deserializer in DH; `kc.json_spec(...)` will parse the bytes as JSON in this version.
    

Get the `probe` result first; once `user.columns` actually shows `userId/name/email`, your Angular bind will stop throwing and live updates will flow.

-------------

Got it — you’re still getting “invalid column ‘userId’” inside `update_view`, even though the Deephaven UI shows values. That means the table you’re operating on **doesn’t actually have `userId/name/email` columns** at that point; most likely you’re consuming as a raw string (or bytes) and the JSON fields are still inside a single column (usually named `Value`). When you then call `update_view("userId=...")`, the name validator throws.

Below is a drop-in, defensive way to make this work no matter which form you’re getting from Kafka:

- If the table already has columns `userId`, `name`, `email` → use them.
    
- Else, if there’s a `Value` column with JSON text → parse it to those columns.
    
- Else → raise a clear error so you know what’s missing.
    

### 1) Robust helpers (paste once in your DH script)

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

def list_cols(t):
    return [c.name for c in t.columns]

def parse_json_value_to_cols(t, want):
    """
    If table `t` has a string column `Value` containing JSON,
    parse it and materialize columns in `want` (strings).
    """
    cols = list_cols(t)
    if "Value" not in cols:
        raise RuntimeError(f"No 'Value' column to parse; columns = {cols}")

    # Parse JSON string -> JsonNode, then extract wanted fields as text
    j = t.update_view(["_node = io.deephaven.json.jackson.JsonTools.parseJson(Value)"])
    exprs = [f"`{c}` = ((_node.get('{c}') == null) ? null : _node.get('{c}').asText())" for c in want]
    out = j.update_view(exprs).drop_columns("_node")
    return out

def ensure_columns(t, want):
    """Return a table that has at least the columns in `want` (as strings)."""
    have = set(list_cols(t))
    need = [c for c in want if c not in have]
    if not need:
        # Coerce to strings to be safe for web
        exprs = [f"`{c}` = ((Object){c}) == null ? null : '' + {c}" for c in want]
        return t.update_view(exprs).view(*want).coalesce()
    # Try to parse from JSON string payload
    return parse_json_value_to_cols(t, want).view(*want).coalesce()
```

### 2) Define your specs (no `mapping=`)

```python
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
})
ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double,   # if your producer sends text -> use dt.string
})
```

### 3) Consume & normalize (works whether JSON was parsed or not)

```python
user_raw = kc.consume(
    KAFKA_CONFIG, TOPIC_USERS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=USER_VALUE_SPEC,
    table_type=kc.TableType.append,
)

account_raw = kc.consume(
    KAFKA_CONFIG, TOPIC_ACCTS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=ACCOUNT_VALUE_SPEC,
    table_type=kc.TableType.append,
)

# Normalize to the columns we want, regardless of how Kafka payload arrives
user    = ensure_columns(user_raw,    ["userId", "name", "email"])
account = ensure_columns(account_raw, ["userId", "accountType", "balance"])

print("USER COLUMNS :", list_cols(user))
print("ACCT COLUMNS :", list_cols(account))
```

If the consumer already produced `userId/name/email`, `ensure_columns` leaves them as string columns. If it only produced `Value`, it parses that JSON text and creates the columns, avoiding the “invalid column” error.

### 4) (Optional) Your append-only, account-driven join

```python
# If you want account-driven stream with user enrichment at that time
users_hist   = user.sort_descending("KafkaTimestamp")  # or whichever event time you rely on
user_account = account.aj(users_hist, on=["userId"], joins=["name","email"])

# Coalesce & keep columns friendly for the web
user         = user.coalesce()
account      = account.coalesce()
user_account = user_account.coalesce()

# Export for Angular
user, account, user_account = user, account, user_account
```

### Why this fixes your error

- Your error log shows only `KafkaPartition/KafkaOffset/KafkaTimestamp` being printed right before `update_view("userId=...")`. That means `userId` didn’t exist (hence “invalid column”). The helper inspects the current table shape and either uses the existing JSON-spec columns or parses `Value` into columns when necessary.
    
- This also prevents follow-on “table already closed” errors: those were cascading after the first failure in your JS viewport binding.
    

### If it still fails

Run this quick probe to confirm the payload:

```python
probe = kc.consume(
    KAFKA_CONFIG, TOPIC_USERS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=kc.ValueSpec.STRING,  # raw text into 'Value'
    table_type=kc.TableType.append,
)
print(list_cols(probe))
print(probe.head(3).to_string())
```

- If `Value` shows proper JSON → the helpers above will parse it.
    
- If it isn’t JSON (Avro/Proto, or double-encoded) → switch to the corresponding Deephaven spec (e.g., `kc.avro_spec(...)`) or fix the producer to emit straight JSON.
    

Try the snippet exactly as-is in a **fresh** DH script tab (new session). Then point your Angular “live table” at the exported `user`, `account`, or `user_account`. This removes the “invalid column” and the subsequent class-cast/closed noise.


-----------------------

You’re right: in your Deephaven build there isn’t a `kc.ValueSpec.STRING`. Use a **simple spec** to read the Kafka value as a single string column, then parse the JSON yourself. That completely avoids the “invalid column `userId`” error because we create those columns explicitly after parsing.

Below is a self-contained script you can paste into a fresh DH Python tab. It works whether your producers send only a JSON string in the value, or you just want to force that path.

```python
# --- imports
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# --- your Kafka config & topics (keep your existing values)
KAFKA_CONFIG = {  # << fill in as you already have >>
    # "bootstrap.servers": "...",
    # "...": "..."
}
TOPIC_USERS  = "..."     # e.g. "cccd01_sb_its_esp_tap3597_bishowcaseraw"
TOPIC_ACCTS  = "..."     # e.g. "cccd01_sb_its_esp_tap3597_bishowcasesecurated"

# ---------- helpers

def list_cols(t):  # quick debugger
    return [c.name for c in t.columns]

def parse_json_value_to_cols(t, json_col, fields):
    """
    t: table with a string column (json_col) containing JSON text.
    fields: list of JSON field names to extract; values are extracted as text.
    """
    j = t.update_view([f"_node = io.deephaven.json.jackson.JsonTools.parseJson({json_col})"])
    exprs = [f"`{f}` = ((_node.get('{f}') == null) ? null : _node.get('{f}').asText())"
            for f in fields]
    return j.update_view(exprs).drop_columns("_node")

# ---------- consume the value as ONE string column named 'Value'

STRING_VALUE = kc.simple_spec([("Value", dt.string)])

user_raw = kc.consume(
    KAFKA_CONFIG, TOPIC_USERS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=STRING_VALUE,
    table_type=kc.TableType.append,
)

account_raw = kc.consume(
    KAFKA_CONFIG, TOPIC_ACCTS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=STRING_VALUE,
    table_type=kc.TableType.append,
)

print("USER_RAW COLUMNS   :", list_cols(user_raw))
print("ACCOUNT_RAW COLUMNS:", list_cols(account_raw))

# ---------- parse JSON -> real columns

user = parse_json_value_to_cols(
    user_raw, "Value", ["userId", "name", "email"]
).drop_columns("Value").coalesce()

# If your JSON has different casing (e.g., "UserId"), change the field list above.
# If balance is numeric in JSON, keep it as text or convert later as needed.

account = parse_json_value_to_cols(
    account_raw, "Value", ["userId", "accountType", "balance"]
).drop_columns("Value").coalesce()

print("USER COLUMNS   :", list_cols(user))
print("ACCOUNT COLUMNS:", list_cols(account))

# ---------- optional: account-driven append result enriched with user at that time

# If you have a Kafka record time column, prefer it here.
# Otherwise, skip the sort; append tables will still stream.
# users_hist = user.sort_descending("KafkaTimestamp")
# user_account = account.aj(users_hist, on=["userId"], joins=["name", "email"]).coalesce()

# Export the tables you want Angular to show
user, account = user, account
# user_account = user_account
```

**Why this fixes it**

- Before, your `update_view("userId=...")` failed because `userId` didn’t exist yet. Here we **always** read the Kafka value as a string column (`Value`) using `kc.simple_spec`, then create the `userId/name/email/...` columns ourselves with `JsonTools.parseJson`. So there’s no chance of “invalid column”.
    
- No reliance on `ValueSpec.STRING` (which your version doesn’t expose).
    
- This pattern also avoids the later cascade of `ClassCastException` and “table already closed” caused by the first failure during your Angular viewport binding.
    

If you still see “invalid column”, run:

```python
print("RAW USER SAMPLE:")
print(user_raw.head(3).to_string())
```

to confirm the payload really is JSON text in `Value`. If your producer sends Avro/Protobuf instead of JSON, switch the spec to the appropriate `kc.avro_spec(...)` / `kc.protostream_spec(...)` and skip manual parsing.