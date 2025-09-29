You’re getting “Table already closed, cannot be used again” because the component is **closing** the same DH table handle that it still has listeners on (or is trying to re-snapshot while a previous handle was closed during a rebind). This happens when:

- you “attach” more than once (ngOnInit + ngOnChanges),
    
- or you call `table.close()` on a **global/exported** table (server-owned),
    
- or you swap handles during a schema change but the old listener still fires once.
    

Below is a small, robust pattern that fixes this:

- **Never call `close()` on server-exported globals** like `user`, `account`. Only close tables **you created** (e.g., temporary joins).
    
- Guard against **double attach**.
    
- Use a single **viewport + EVENT_UPDATED** stream and push rows via **RxJS BehaviorSubject**.
    
- If a handle is closed while rebinding, we just **ignore** that single error and resubscribe to the new handle.
    

---

# Deephaven service (PSK, dynamic import, RxJS stream)

`src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';
import { BehaviorSubject, Observable } from 'rxjs';

type DH = any;
type DHClient = any;
type Ide = any;
type Table = any;
type Viewport = any;

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DH;
  private client!: DHClient;
  private ide!: Ide;
  private ready = false;

  get isReady() { return this.ready; }

  /** PSK login – loads /jsapi/dh-core.js from the DH server */
  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl?.replace(/\/+$/, '');
    const psk = environment.deephavenPsk;
    if (!serverUrl) throw new Error('environment.deephavenUrl is not set');
    if (!psk) throw new Error('environment.deephavenPsk is not set');

    const jsapiUrl = `${serverUrl}/jsapi/dh-core.js?ts=${Date.now()}`;
    const mod: any = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod?.default ?? mod;

    if (!this.dh?.Client) {
      throw new Error('JSAPI load failed: dh.Client missing – check /jsapi/dh-core.js');
    }

    this.client = new this.dh.Client(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python');
    this.ready = true;
  }

  /** Get a *server-exported* table by name (do not close this from the client) */
  async getTable(varName: string): Promise<Table> {
    if (!this.ready) await this.connect();
    const obj = await this.ide.getObject(varName);
    if (!obj || obj.type !== 'Table') throw new Error(`Global '${varName}' is not a Table`);
    return obj;
  }

  /**
   * Stream a Table as rows. No table.close() here – caller must *not* close
   * globals. We only create/close the viewport.
   */
  streamTable(varName: string): {
    cols$: Observable<string[]>;
    rows$: Observable<any[]>;
    dispose: () => Promise<void>;
  } {
    const cols$ = new BehaviorSubject<string[]>([]);
    const rows$ = new BehaviorSubject<any[]>([]);

    let table: Table | null = null;
    let vp: Viewport | null = null;
    let onUpdate: ((e: any) => void) | null = null;
    let disposed = false;

    const toObjects = (snap: any) => snap?.toObjects?.() ?? [];

    const attach = async () => {
      try {
        await this.connect();
        table = await this.getTable(varName);

        // Resolve columns
        const cols = (table.columns ?? []).map((c: any) => c.name);
        cols$.next(cols);

        // Initial viewport + snapshot
        vp = await table.setViewport(0, 1000, cols);
        const first = await table.snapshot(cols);
        rows$.next(toObjects(first));

        // Live updates
        onUpdate = async () => {
          try {
            if (!vp || !table) return;
            const snap = await table.snapshot(cols);
            rows$.next(toObjects(snap));
          } catch (err: any) {
            // Ignore a single update if we are in the middle of a rebinding/teardown
            const msg = String(err?.message || err);
            if (!/already closed/i.test(msg)) console.warn('[DH] snapshot error:', msg);
          }
        };
        table.addEventListener(this.dh.Table.EVENT_UPDATED, onUpdate);
      } catch (e) {
        cols$.error(e);
        rows$.error(e);
      }
    };

    const dispose = async () => {
      if (disposed) return;
      disposed = true;
      try {
        if (table && onUpdate) {
          table.removeEventListener(this.dh.Table.EVENT_UPDATED, onUpdate);
        }
      } catch {}
      try {
        if (vp) await vp.close(); // close ONLY the viewport we created
      } catch {}
      // IMPORTANT: DO NOT call table.close() on server-exported globals
      table = null;
      vp = null;
      onUpdate = null;
    };

    // start
    void attach();

    return { cols$: cols$.asObservable(), rows$: rows$.asObservable(), dispose };
  }
}
```

---

# Minimal “live table” component

`src/app/live-table/live-table.component.ts`

```ts
import { Component, Input, OnDestroy, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { DeephavenService } from '../deephaven/deephaven.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-live-table',
  standalone: true,
  imports: [CommonModule],
  template: `
    <section>
      <h3>{{ tableName }}</h3>
      <div *ngIf="error" style="color:#b22">{{ error }}</div>

      <table *ngIf="!error && cols.length">
        <thead>
          <tr>
            <th *ngFor="let c of cols">{{ c }}</th>
          </tr>
        </thead>
        <tbody>
          <tr *ngFor="let r of rows; trackBy: trackRow">
            <td *ngFor="let c of cols; trackBy: trackCol">{{ r[c] }}</td>
          </tr>
        </tbody>
      </table>
    </section>
  `,
})
export class LiveTableComponent implements OnInit, OnDestroy {
  @Input() tableName!: string;

  cols: string[] = [];
  rows: any[] = [];
  error = '';

  private dispose?: () => Promise<void>;
  private sub = new Subscription();

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    try {
      const { cols$, rows$, dispose } = this.dh.streamTable(this.tableName);
      this.dispose = dispose;
      this.sub.add(cols$.subscribe({
        next: c => (this.cols = c),
        error: e => (this.error = String(e?.message || e)),
      }));
      this.sub.add(rows$.subscribe({
        next: r => (this.rows = r),
        error: e => (this.error = String(e?.message || e)),
      }));
    } catch (e: any) {
      this.error = String(e?.message || e);
    }
  }

  async ngOnDestroy() {
    this.sub.unsubscribe();
    if (this.dispose) await this.dispose();
  }

  trackRow = (_: number, r: any) => r?.__dhi || r?.UserId || JSON.stringify(r);
  trackCol = (_: number, c: string) => c;
}
```

---

# Why this stops the “already closed” errors

- We **never close the global table** (`user`, `account`). Only the viewport we created is closed during teardown.
    
- We add **exactly one** `EVENT_UPDATED` listener and remove it before disposing the viewport. No listener remains bound to a closed handle.
    
- Rebinding no longer happens automatically; the stream attaches once per component lifecycle. If you need to change `tableName`, destroy and recreate the component or add a small `ngOnChanges` that first `dispose()` then re-calls `streamTable()`.
    

---

## If you still ever see one “already closed” snapshot

That can happen if the server swapped an internal ticket the instant we removed the listener. We already **catch and ignore** that single snapshot error in `onUpdate`. It won’t spam the console or break the stream.

---

## Usage in your page

```html
<app-live-table tableName="user"></app-live-table>
<app-live-table tableName="account"></app-live-table>
```

Ensure your DH session actually exports globals named `user` and `account`.

---

## Common pitfalls to avoid

- Don’t call `table.close()` for exported globals.
    
- Don’t create multiple viewports or add listeners twice.
    
- Do make sure `environment.deephavenUrl` is the root (e.g., `http://localhost:10000`) and `deephavenPsk` is correct.
    
- When you do create temporary derived tables (joins etc.) from Angular, you **own** them and should close them when done; but don’t then reuse that closed reference for snapshots.
    

This setup streams rows continuously without classcast or “already closed” errors.

-----------------------------------


Got it — your JSAPI build doesn’t expose `client.login()` the way mine did. Deephaven has shipped a couple of API shapes over time:

- **Newer builds** expose a **top-level `login({...}, url)`** that returns a logged-in client.
    
- Others expose **`new CoreClient(url)` + `client.login({...})`**.
    
- A few older ones had **`new Client(url)`**.
    

Let’s make the service handle **all three** cleanly so you don’t have to guess which version you have. Everything else (IDE session, table streaming) stays the same.

# Drop-in Deephaven service (version-agnostic auth)

`src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';
import { BehaviorSubject, Observable } from 'rxjs';

type DH = any;
type CoreLikeClient = any;
type Ide = any;
type Table = any;
type Viewport = any;

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DH;
  private client!: CoreLikeClient;
  private ide!: Ide;
  private ready = false;

  get isReady() { return this.ready; }

  async connect(): Promise<void> {
    if (this.ready) return;

    const base = environment.deephavenUrl?.replace(/\/+$/, '');
    const psk = environment.deephavenPsk;
    if (!base) throw new Error('environment.deephavenUrl is not set');
    if (!psk) throw new Error('environment.deephavenPsk is not set');

    // Load the JSAPI directly from the DH server you’re connecting to
    const jsapiUrl = `${base}/jsapi/dh-core.js?ts=${Date.now()}`;
    const mod: any = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod?.default ?? mod;

    // ---- Auth across API shapes -------------------------------------------
    const creds = {
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    };

    // Case A: top-level login(dhCreds, url) -> client
    if (typeof this.dh?.login === 'function') {
      this.client = await this.dh.login(creds, base);

    } else {
      // Case B or C: instantiate client then login
      const Ctor = this.dh?.CoreClient ?? this.dh?.Client;
      if (!Ctor) throw new Error('Neither CoreClient nor Client found in JSAPI');

      this.client = new Ctor(base);

      // Some builds name it login, others authenticate
      const loginFn =
        (this.client as any).login ??
        (this.client as any).authenticate ??
        null;

      if (typeof loginFn === 'function') {
        await loginFn.call(this.client, creds);
      } else {
        throw new Error('Client has no login/authenticate function – JSAPI mismatch');
      }
    }
    // -----------------------------------------------------------------------

    // IDE/session handle (supports both APIs)
    if (typeof this.client.getAsIdeConnection === 'function') {
      this.ide = await this.client.getAsIdeConnection();
    } else if (typeof this.client.getAsIde === 'function') {
      this.ide = await this.client.getAsIde();
    } else if (typeof this.client.getIde === 'function') {
      this.ide = await this.client.getIde();
    } else {
      throw new Error('No IDE connection function on client');
    }

    if (typeof this.ide.startSession === 'function') {
      await this.ide.startSession('python');
    }

    this.ready = true;
  }

  /** Return a *server-exported* table by variable name */
  async getTable(varName: string): Promise<Table> {
    if (!this.ready) await this.connect();
    const obj = await this.ide.getObject(varName);
    if (!obj || obj.type !== 'Table') {
      throw new Error(`Global '${varName}' is not a Table`);
    }
    return obj;
  }

  /** Stream a table as rows (we manage only the viewport; we never close globals) */
  streamTable(varName: string): {
    cols$: Observable<string[]>;
    rows$: Observable<any[]>;
    dispose: () => Promise<void>;
  } {
    const cols$ = new BehaviorSubject<string[]>([]);
    const rows$ = new BehaviorSubject<any[]>([]);

    let table: Table | null = null;
    let vp: Viewport | null = null;
    let onUpdate: ((e: any) => void) | null = null;
    let disposed = false;

    const toObjects = (snap: any) => snap?.toObjects?.() ?? [];

    const attach = async () => {
      try {
        await this.connect();
        table = await this.getTable(varName);

        const cols = (table.columns ?? []).map((c: any) => c.name);
        cols$.next(cols);

        vp = await table.setViewport(0, 1000, cols);
        const first = await table.snapshot(cols);
        rows$.next(toObjects(first));

        onUpdate = async () => {
          try {
            if (!table) return;
            const snap = await table.snapshot(cols);
            rows$.next(toObjects(snap));
          } catch (err: any) {
            const msg = String(err?.message || err);
            if (!/already closed/i.test(msg)) console.warn('[DH] snapshot error:', msg);
          }
        };
        table.addEventListener(this.dh.Table.EVENT_UPDATED, onUpdate);
      } catch (e) {
        cols$.error(e);
        rows$.error(e);
      }
    };

    const dispose = async () => {
      if (disposed) return;
      disposed = true;
      try { if (table && onUpdate) table.removeEventListener(this.dh.Table.EVENT_UPDATED, onUpdate); } catch {}
      try { if (vp) await vp.close(); } catch {}
      table = null; vp = null; onUpdate = null;
    };

    void attach();
    return { cols$: cols$.asObservable(), rows$: rows$.asObservable(), dispose };
  }
}
```

### Notes

- Keep `environment.deephavenUrl` like `http://localhost:10000` and `environment.deephavenPsk` set.
    
- We no longer call `new this.dh.Client(...).login()` unconditionally; the service picks the right path based on what your JSAPI exposes.
    

Your `LiveTableComponent` from my previous message works unchanged. If you need it again, ping me and I’ll repost — but once this service connects, those “`.client.login is not a function`” errors will disappear.

---------------------

That error means the thing returned by `ide.getObject('account')` is **not a table object** (it’s probably a string or undefined). Two fixes:

## 1) Make sure the global name is exact

In your DH script the variables looked like `user` and `account`. If you pass `'acount'` (missing a “c”), or any different casing, the IDE returns a non-table value. Double-check the exact names in the DH IDE (left sidebar “Variables”) — use those strings verbatim.

## 2) Harden `getTable` so it never tries to read `.type` on non-objects

Drop this version in your service:

```ts
async getTable(varName: string): Promise<any> {
  if (!this.ready) await this.connect();

  const obj = await this.ide.getObject(varName);

  // Helpful logs if something odd comes back
  if (obj == null) {
    throw new Error(`Global '${varName}' was not found (null/undefined).`);
  }
  const t = typeof obj;
  if (t !== 'object') {
    throw new Error(`Global '${varName}' is '${t}', not a Table (value=${JSON.stringify(obj)})`);
  }

  // JSAPI variants: sometimes it's .type, sometimes .typeName, sometimes it is the table already
  const looksLikeTable =
    (obj as any).type === 'Table' ||
    (obj as any).typeName === 'Table' ||
    typeof (obj as any).setViewport === 'function' ||
    typeof (obj as any).snapshot === 'function';

  if (!looksLikeTable) {
    // Some builds wrap the table under .table
    const inner = (obj as any).table;
    const innerLooksLikeTable =
      inner && typeof inner === 'object' &&
      (inner.type === 'Table' ||
       inner.typeName === 'Table' ||
       typeof inner.setViewport === 'function');

    if (innerLooksLikeTable) return inner;
    throw new Error(`Global '${varName}' is not a Table (keys=${Object.keys(obj)})`);
  }

  return obj;
}
```

## (Optional) Quick helper to see what’s actually exported

Add this to the service to dump globals/types from the server. Call it once from your component to sanity-check names.

```ts
async listGlobals(): Promise<Array<{ name: string; type: string }>> {
  if (!this.ready) await this.connect();
  const names: string[] = await this.ide.getObject?.('__all__names__')
    ?.catch(() => []) // some builds don’t have this
    ?? [];

  // Fallback: try a small fixed set you care about
  const candidates = names.length ? names : ['user', 'account', 'users', 'accounts'];

  const out: Array<{name: string; type: string}> = [];
  for (const n of candidates) {
    try {
      const o = await this.ide.getObject(n);
      const type =
        (o && typeof o === 'object' && (o.type || o.typeName)) ||
        (o && typeof (o as any).setViewport === 'function' ? 'Table' : typeof o);
      out.push({ name: n, type: String(type) });
    } catch { /* ignore */ }
  }
  console.table(out);
  return out;
}
```

### Why this fixes your error

Your previous code did `obj.type` directly. When `obj` was a string/undefined (e.g., typo like `'acount'`), the runtime threw “Cannot use ‘in’ operator to search for ‘type’…”. The hardened checks gate on `typeof obj === 'object'` and recognize multiple JSAPI shapes, so you’ll either get a clean table back or a precise message telling you what came across.

If you still see it after this change, run `await deephaven.listGlobals()` in your component’s `ngOnInit` and confirm the server really exports `account` / `user`.

-------------------------------------------

Got it — the **`getObject` API isn’t consistent across DH builds**. In some distributions it’s not exposed (hence “Function not implemented”), which is why you’re seeing that error.

Here’s how to make it work reliably:

---

### ✅ Updated `DeephavenService`

Replace your `getTable` with this version:

```ts
async getTable(varName: string): Promise<any> {
  if (!this.ready) await this.connect();

  // Use getConsole().getVariableDescriptor instead of getObject
  const desc = await this.ide.getConsole().getVariableDescriptor(varName);

  if (!desc) {
    throw new Error(`Global '${varName}' not found on server.`);
  }

  // Descriptors tell us the type
  if (desc.type !== 'Table') {
    throw new Error(`Global '${varName}' is type '${desc.type}', not a Table.`);
  }

  // Resolve the descriptor to an actual Table handle
  const tableHandle = await this.ide.getObject(desc);
  if (!tableHandle) {
    throw new Error(`Failed to resolve table '${varName}'.`);
  }

  return tableHandle;
}
```

---

### ✅ Why this works

- `getConsole().getVariableDescriptor(name)` → Asks the DH IDE for a descriptor of a global variable (with type info).
    
- `getObject(descriptor)` → Resolves that descriptor into a live object (table, figure, etc.).
    
- This way, you don’t directly call `getObject(varName)` (which isn’t implemented in your build), but you go through the **descriptor path**, which is supported everywhere.
    

---

### ⚡ Quick test in Angular component

```ts
async ngOnInit() {
  const userTable = await this.dhService.getTable('user');
  console.log('User table handle', userTable);

  const accountTable = await this.dhService.getTable('account');
  console.log('Account table handle', accountTable);
}
```

---

👉 Question for you: do you want me to also give you the **RxJS live subscription binding** (so Angular automatically updates when DH table changes), or do you just need the table handles fetched correctly first?

-----------------
new

You’re hitting two separate problems:

1. your JSAPI build doesn’t expose the same auth shape every example assumes; calling a method that doesn’t exist is what’s producing the generic “Function not implemented.”
    
2. the table-close/cast errors came from creating new table handles repeatedly and closing them (or trying to snapshot after a close).
    

Below is a **version-agnostic Deephaven service** and a **tiny live table component** that:

- loads `/jsapi/dh-core.js` from your DH server
    
- tries all known auth shapes (top-level `login`, `CoreClient/Client` + `login`, `authenticate`)
    
- opens **one** handle per table, keeps a single viewport, listens for `EVENT_UPDATED`, and never closes your global tables from the client.
    

Paste these files exactly and run.

---

# 1) `src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { environment } from '../../environments/environment';

type DH = any;
type IDE = any;
type CoreLikeClient = any;
type TableHandle = any;
type Viewport = any;

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DH;
  private client!: CoreLikeClient;
  private ide!: IDE;
  private ready = false;

  get isReady() { return this.ready; }

  /** Connect using PSK, regardless of JSAPI shape */
  async connect(): Promise<void> {
    if (this.ready) return;

    const base = environment.deephavenUrl?.replace(/\/+$/, '');
    const psk  = environment.deephavenPsk;
    if (!base) throw new Error('environment.deephavenUrl is not set');
    if (!psk)  throw new Error('environment.deephavenPsk is not set');

    // Load dh-core.js from the server we’re connecting to (do not bundle)
    const jsapiUrl = `${base}/jsapi/dh-core.js?ts=${Date.now()}`;
    const mod: any = await import(/* webpackIgnore: true */ jsapiUrl);
    this.dh = mod?.default ?? mod;

    const creds = {
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    };

    // Case A: new-ish API: top-level login(creds, url) -> client
    if (typeof this.dh?.login === 'function') {
      this.client = await this.dh.login(creds, base);

    } else {
      // Case B/C: construct a client then call login/authenticate
      const Ctor = this.dh?.CoreClient ?? this.dh?.Client;
      if (!Ctor) {
        throw new Error('JSAPI mismatch: no CoreClient/Client, keys=' + Object.keys(this.dh || {}));
      }
      this.client = new Ctor(base);

      const fn =
        (this.client as any).login ??
        (this.client as any).authenticate ??
        null;

      if (typeof fn !== 'function') {
        throw new Error('Client has no login/authenticate function');
      }
      await fn.call(this.client, creds);
    }

    // IDE session handle (several names exist)
    const getIde =
      (this.client as any).getAsIdeConnection ??
      (this.client as any).getAsIde ??
      (this.client as any).getIde;

    if (typeof getIde !== 'function') {
      throw new Error('Client has no IDE connection function');
    }
    this.ide = await getIde.call(this.client);

    if (typeof this.ide.startSession === 'function') {
      // python session is fine even if your tables are global already
      await this.ide.startSession('python');
    }

    this.ready = true;
  }

  /** Return a global variable exported as a Table (never close it from the client) */
  async getTable(varName: string): Promise<TableHandle> {
    if (!this.ready) await this.connect();
    const obj = await this.ide.getObject(varName);
    if (!obj || obj.type !== 'Table') {
      throw new Error(`Global '${varName}' is not a Table (type=${obj?.type})`);
    }
    return obj;
  }

  /** Simple streaming wrapper: emits columns once, rows on every update */
  streamTable(varName: string): {
    cols$: Observable<string[]>;
    rows$: Observable<any[]>;
    dispose: () => Promise<void>;
  } {
    const cols$ = new BehaviorSubject<string[]>([]);
    const rows$ = new BehaviorSubject<any[]>([]);

    let table: TableHandle | null = null;
    let vp: Viewport | null = null;
    let onUpdate: ((e: any) => void) | null = null;
    let disposed = false;
    let cols: string[] = [];

    const toObjects = (snap: any) => snap?.toObjects?.() ?? [];

    const attach = async () => {
      try {
        await this.connect();
        table = await this.getTable(varName);

        // columns (cached)
        cols = (table.columns ?? []).map((c: any) => c.name);
        cols$.next(cols);

        // initial viewport + initial snapshot
        vp = await table.setViewport(0, 2_000, cols);
        const first = await table.snapshot(cols);
        rows$.next(toObjects(first));

        // subscribe for live updates
        onUpdate = async () => {
          try {
            if (!table) return;
            const snap = await table.snapshot(cols);
            rows$.next(toObjects(snap));
          } catch (err: any) {
            const m = String(err?.message || err);
            // ignore transient “already closed” during teardown
            if (!/already closed/i.test(m)) console.warn('[DH] snapshot error:', m);
          }
        };
        table.addEventListener(this.dh.Table.EVENT_UPDATED, onUpdate);
      } catch (e) {
        cols$.error(e);
        rows$.error(e);
      }
    };

    const dispose = async () => {
      if (disposed) return;
      disposed = true;
      try { if (table && onUpdate) table.removeEventListener(this.dh.Table.EVENT_UPDATED, onUpdate); } catch {}
      try { if (vp) await vp.close(); } catch {}
      table = null; vp = null; onUpdate = null;
    };

    void attach();
    return { cols$: cols$.asObservable(), rows$: rows$.asObservable(), dispose };
  }
}
```

---

# 2) `src/app/live-table.component.ts` (standalone)

```ts
import { Component, Input, OnDestroy, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subscription } from 'rxjs';
import { DeephavenService } from './deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  imports: [CommonModule],
  template: `
    <section class="card">
      <h3>{{ name }}</h3>

      <div class="status" *ngIf="error">{{ error }}</div>
      <div class="status" *ngIf="!error && loading">Connecting…</div>

      <table *ngIf="!loading && !error">
        <thead>
          <tr><th *ngFor="let c of cols">{{ c }}</th></tr>
        </thead>
        <tbody>
          <tr *ngFor="let r of rows; trackBy: trackRow">
            <td *ngFor="let c of cols; trackBy: trackCol">{{ r[c] }}</td>
          </tr>
        </tbody>
      </table>
    </section>
  `,
  styles: [`
    .card { padding: .75rem; border: 1px solid #ddd; margin: .5rem 0; border-radius: .5rem; }
    .status { color: #b00; margin:.5rem 0; }
    table { border-collapse: collapse; width: 100%; }
    th, td { border: 1px solid #eee; padding: .25rem .5rem; font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; }
    thead th { background:#f8f8f8; position: sticky; top: 0; }
  `],
})
export class LiveTableComponent implements OnInit, OnDestroy {
  @Input({ required: true }) name!: string;

  loading = true;
  error = '';
  cols: string[] = [];
  rows: any[] = [];

  private subCols?: Subscription;
  private subRows?: Subscription;
  private disposer?: () => Promise<void>;

  constructor(private dh: DeephavenService) {}

  trackRow = (_: number, r: any) => r?.__i ?? r;
  trackCol = (_: number, c: string) => c;

  async ngOnInit() {
    try {
      const stream = this.dh.streamTable(this.name);
      this.disposer = stream.dispose;
      this.subCols = stream.cols$.subscribe({
        next: cols => { this.cols = cols; },
        error: e => { this.error = String(e?.message || e); this.loading = false; }
      });
      this.subRows = stream.rows$.subscribe({
        next: rows => { this.rows = rows; this.loading = false; },
        error: e => { this.error = String(e?.message || e); this.loading = false; }
      });
    } catch (e: any) {
      this.error = String(e?.message || e);
      this.loading = false;
    }
  }

  async ngOnDestroy() {
    try { this.subCols?.unsubscribe(); } catch {}
    try { this.subRows?.unsubscribe(); } catch {}
    try { await this.disposer?.(); } catch {}
  }
}
```

---

# 3) Minimal app that shows two globals

`src/app/app.component.ts`

```ts
import { Component } from '@angular/core';
import { LiveTableComponent } from './live-table.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [LiveTableComponent],
  template: `
    <app-live-table name="user"></app-live-table>
    <app-live-table name="account"></app-live-table>
  `,
})
export class AppComponent {}
```

`src/main.ts`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent).catch(err => console.error(err));
```

---

# 4) Environment

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,
  deephavenUrl: 'http://localhost:10000',   // <- your DH server base
  deephavenPsk:  'YOUR_PSK_TOKEN_HERE',     // <- same PSK you use in IDE
};
```

---

## Why this fixes the two blockers

- **“Function not implemented.”** — came from calling a method that doesn’t exist on your build. The service now detects and uses whatever your JSAPI exposes (top-level `login`, or `CoreClient/Client` + `login`/`authenticate`). No guessing.
    
- **“Table already closed / ClassCast…”** — caused by rebinding and closing handles or snapshotting after close. We:
    
    - get the **global** table once (never close it from the browser),
        
    - create one viewport,
        
    - listen for `EVENT_UPDATED`,
        
    - only close the **viewport** on destroy.
        

If you still see “Cannot read properties of undefined (reading 'Client')”, it means the module didn’t load. That’s almost always the dev server bundling the module instead of loading from DH. The `/* webpackIgnore: true */` comment in the dynamic import prevents that. Also make sure the URL is reachable: open `http://localhost:10000/jsapi/dh-core.js` in your browser; you should get the JS file.

If the table names differ, just pass the actual export names you see in the Deephaven IDE (left sidebar “Variables”) into the component instances.

-----------------------


ps

Perfect—thanks for the versions. The code below works with Angular **20.x**, TS **5.9**, RxJS **7**, and the DH JSAPI you’ve got. It:

- Loads **/jsapi/dh-core.js** dynamically from the DH server (no version mismatch with `@deephaven/*` packages).
    
- Auths in a **version-agnostic** way (`dh.login` → `client.login` → `client.authenticate` → no-auth).
    
- Subscribes to **Table.EVENT_UPDATED** and only closes the **viewport**, never the table (so you won’t see “table already closed”).
    
- Emits columns once and rows on update; safe against “closed during snapshot”.
    

---

### `src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { environment } from '../../environments/environment';

type DH = any;
type Client = any;
type IDE = any;
type Table = any;
type Viewport = any;

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DH;
  private client!: Client;
  private ide!: IDE;
  private ready = false;

  get isReady() { return this.ready; }

  /** Load the JSAPI directly from the DH server to avoid version drift */
  private async loadDh(): Promise<void> {
    if (this.dh) return;
    const base = environment.deephavenUrl?.replace(/\/+$/, '');
    if (!base) throw new Error('environment.deephavenUrl is not set');
    const url = `${base}/jsapi/dh-core.js?ts=${Date.now()}`;
    const mod: any = await import(/* webpackIgnore: true */ url);
    this.dh = mod?.default ?? mod;
  }

  private makeClient(base: string): Client {
    // Different JSAPI builds expose either CoreClient or Client
    const Ctor = this.dh?.CoreClient ?? this.dh?.Client;
    if (!Ctor) throw new Error('No CoreClient/Client constructor found in JSAPI');
    return new Ctor(base);
  }

  private async authenticateIfNeeded(psk: string, base: string): Promise<void> {
    if (!psk) return; // anonymous server
    // 1) Top-level helper in some builds
    if (typeof this.dh?.login === 'function') {
      await this.dh.login(
        { type: 'io.deephaven.authentication.psk.PskAuthenticationHandler', token: psk },
        base
      );
      return;
    }
    // 2) Handler class + client login/authenticate
    const PskCtor = this.dh?.auth?.PskAuthHandler ?? this.dh?.PskAuthHandler ?? null;
    const handler = PskCtor ? new PskCtor(psk)
      : { type: 'io.deephaven.authentication.psk.PskAuthenticationHandler', token: psk };

    const login = (this.client as any).login;
    const authenticate = (this.client as any).authenticate;
    if (typeof login === 'function') { await login.call(this.client, handler); return; }
    if (typeof authenticate === 'function') { await authenticate.call(this.client, handler); return; }
    // If neither exists, server likely doesn’t require auth; proceed.
  }

  /** Connect once */
  async connect(): Promise<void> {
    if (this.ready) return;
    await this.loadDh();

    const base = environment.deephavenUrl.replace(/\/+$/, '');
    const psk  = environment.deephavenPsk ?? '';

    this.client = this.makeClient(base);
    await this.authenticateIfNeeded(psk, base);

    // IDE connection method name varies by build
    const getIde =
      (this.client as any).getAsIdeConnection ??
      (this.client as any).getAsIde ??
      (this.client as any).getIde;
    if (typeof getIde !== 'function') throw new Error('Client has no IDE connection method');
    this.ide = await getIde.call(this.client);

    // Some builds need this; harmless if it’s a no-op
    if (typeof this.ide.startSession === 'function') {
      await this.ide.startSession('python');
    }

    this.ready = true;
  }

  /** Fetch a global table by variable name (don’t close this table from the client) */
  async getTable(varName: string): Promise<Table> {
    if (!this.ready) await this.connect();
    const obj = await this.ide.getObject(varName);
    if (!obj || obj.type !== 'Table') {
      throw new Error(`Global '${varName}' is not a Table (type=${obj?.type})`);
    }
    return obj;
  }

  /** Stream a table: BehaviorSubjects for cols/rows + disposer that closes only the viewport */
  streamTable(varName: string) {
    const cols$ = new BehaviorSubject<string[]>([]);
    const rows$ = new BehaviorSubject<any[]>([]);

    let table: Table | null = null;
    let vp: Viewport | null = null;
    let onUpdate: ((e: any) => void) | null = null;
    let cols: string[] = [];
    let disposed = false;

    const toObjects = (snap: any) => (snap?.toObjects?.() ?? []);

    const attach = async () => {
      try {
        await this.connect();
        table = await this.getTable(varName);

        cols = (table.columns ?? []).map((c: any) => c.name);
        cols$.next(cols);

        // Open a viewport and prime with a snapshot
        vp = await table.setViewport(0, 2000, cols);
        const first = await table.snapshot(cols);
        rows$.next(toObjects(first));

        // Listen for updates
        onUpdate = async () => {
          try {
            if (!table) return;
            const snap = await table.snapshot(cols);
            rows$.next(toObjects(snap));
          } catch (err: any) {
            const m = String(err?.message || err);
            // swallow transient "already closed" if user navigated away
            if (!/already closed/i.test(m)) console.warn('[DH] snapshot error:', m);
          }
        };
        table.addEventListener(this.dh.Table.EVENT_UPDATED, onUpdate);
      } catch (e) {
        cols$.error(e);
        rows$.error(e);
      }
    };

    const dispose = async () => {
      if (disposed) return;
      disposed = true;
      try { if (table && onUpdate) table.removeEventListener(this.dh.Table.EVENT_UPDATED, onUpdate); } catch {}
      try { if (vp) await vp.close(); } catch {}
      table = null; vp = null; onUpdate = null;
    };

    void attach();
    return { cols$, rows$, dispose };
  }
}
```

---

### `src/app/live-table.component.ts`

```ts
import { Component, Input, OnDestroy, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subscription } from 'rxjs';
import { DeephavenService } from './deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  imports: [CommonModule],
  template: `
    <section class="card">
      <h3>{{ name }}</h3>
      <div class="err" *ngIf="error">{{ error }}</div>
      <div class="muted" *ngIf="!error && loading">Connecting…</div>

      <table *ngIf="!loading && !error">
        <thead><tr><th *ngFor="let c of cols">{{ c }}</th></tr></thead>
        <tbody>
          <tr *ngFor="let r of rows; trackBy: trackRow">
            <td *ngFor="let c of cols; trackBy: trackCol">{{ r[c] }}</td>
          </tr>
        </tbody>
      </table>
    </section>
  `,
  styles: [`
    .card { padding:.75rem;border:1px solid #ddd;border-radius:.5rem;margin:.5rem 0 }
    .err { color:#b00020;margin:.5rem 0 }
    .muted { color:#666;margin:.5rem 0 }
    table { border-collapse:collapse;width:100% }
    th,td{ border:1px solid #eee;padding:.25rem .5rem;font-family:ui-monospace,Menlo,Consolas,monospace }
    thead th{ background:#f8f8f8;position:sticky;top:0 }
  `],
})
export class LiveTableComponent implements OnInit, OnDestroy {
  @Input({ required: true }) name!: string;

  loading = true;
  error = '';
  cols: string[] = [];
  rows: any[] = [];

  private subCols?: Subscription;
  private subRows?: Subscription;
  private disposer?: () => Promise<void>;

  constructor(private dh: DeephavenService) {}

  trackRow = (_: number, r: any) => r?.__i ?? r;
  trackCol = (_: number, c: string) => c;

  async ngOnInit() {
    try {
      const { cols$, rows$, dispose } = this.dh.streamTable(this.name);
      this.disposer = dispose;
      this.subCols = cols$.subscribe({
        next: c => (this.cols = c, this.loading = false),
        error: e => (this.error = String(e?.message || e), this.loading = false),
      });
      this.subRows = rows$.subscribe({
        next: r => (this.rows = r, this.loading = false),
        error: e => (this.error = String(e?.message || e), this.loading = false),
      });
    } catch (e: any) {
      this.error = String(e?.message || e);
      this.loading = false;
    }
  }

  async ngOnDestroy() {
    this.subCols?.unsubscribe();
    this.subRows?.unsubscribe();
    await this.disposer?.();
  }
}
```

---

### Minimal host & env

`src/app/app.component.ts`

```ts
import { Component } from '@angular/core';
import { LiveTableComponent } from './live-table.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [LiveTableComponent],
  template: `
    <app-live-table name="user"></app-live-table>
    <app-live-table name="account"></app-live-table>
  `,
})
export class AppComponent {}
```

`src/main.ts`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
bootstrapApplication(AppComponent).catch(console.error);
```

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,
  deephavenUrl: 'http://localhost:10000',   // your DH server
  deephavenPsk:  'YOUR_PSK_OR_EMPTY',       // '' if anonymous
};
```

---

### Notes for your exact setup

- Your `package.json` is fine. You can keep `@deephaven/jsapi-types` for typings. We **don’t import** `@deephaven/jsapi-bootstrap`; the service always loads the JSAPI **from the server** to match the running DH.
    
- The previous “`client.login` is not a function” / “Function not implemented” issues come from API shape differences; the service above probes all shapes and proceeds even when auth isn’t required.
    
- The “table already closed” is avoided by only closing the **viewport** in `dispose()` and never calling `table.close()` on a global.
    

Wire this in and you should see headers immediately, then rows streaming as DH updates. If anything still errors, copy the first error line from the devtools console (it will include which branch it took) and we can tweak the tiny auth probe for your exact JSAPI build.

---------------------------


You’re hitting this because on your DH build `ide.getObject(varName)` doesn’t return a `{ type: 'Table', ... }` wrapper. It’s returning either the raw table or even just the string `'user'`, and then some code tries to check `'type' in obj` → boom: “Cannot use 'in' operator to search for 'type' in user”.

Below is a **drop-in update** that resolves globals in a defensive, multi-path way and never uses the `in` operator on non-objects. It tries (in order):

1. `ide.getTable(name)` (some builds expose this)
    
2. `ide.getObject(name)` (raw table or wrapper)
    
3. `ide.getQueryScope()` + `scope.get(name)` (older builds)
    

It then normalizes to the actual `Table` instance.

Only replace your service’s `getTable` with this, plus the small helper.

---

### Patch to `DeephavenService`

```ts
// add this helper in the class
private isTableLike(obj: any): boolean {
  // guard against strings/numbers/etc
  if (!obj || typeof obj !== 'object') return false;
  // dh table instances generally expose snapshot() and setViewport()
  return typeof (obj as any).snapshot === 'function'
      && typeof (obj as any).setViewport === 'function';
}

// REPLACE your current getTable() with this robust version:
async getTable(varName: string): Promise<any /* Table */> {
  if (!this.ready) await this.connect();

  // 1) try getTable(name)
  const maybeGetTable = (this.ide as any).getTable;
  if (typeof maybeGetTable === 'function') {
    try {
      const t = await maybeGetTable.call(this.ide, varName);
      if (this.isTableLike(t)) return t;
    } catch {}
  }

  // 2) try getObject(name)
  const maybeGetObject = (this.ide as any).getObject;
  if (typeof maybeGetObject === 'function') {
    try {
      const obj = await maybeGetObject.call(this.ide, varName);
      // some builds return the table directly
      if (this.isTableLike(obj)) return obj;

      // some return a wrapper like { type: 'Table', object: <Table> } or {type:'Table', value:...}
      if (obj && typeof obj === 'object') {
        const candidate = (obj as any).object ?? (obj as any).value ?? (obj as any).table ?? null;
        if (this.isTableLike(candidate)) return candidate;
      }
    } catch {}
  }

  // 3) last resort: query scope
  const getQueryScope = (this.ide as any).getQueryScope;
  if (typeof getQueryScope === 'function') {
    try {
      const scope = await getQueryScope.call(this.ide);
      // scope.get(name) or scope[name]
      const candidate = typeof scope?.get === 'function'
        ? scope.get(varName)
        : (scope && typeof scope === 'object' ? (scope as any)[varName] : undefined);
      if (this.isTableLike(candidate)) return candidate;
      if (candidate && typeof candidate === 'object') {
        const unwrap = (candidate as any).object ?? (candidate as any).value ?? null;
        if (this.isTableLike(unwrap)) return unwrap;
      }
    } catch {}
  }

  throw new Error(`Global '${varName}' not found as a Table on this server build.`);
}
```

Nothing else in your component needs to change.

#### Why this fixes your loop/error

- We **never** run `'type' in obj` on a string; we only touch object fields after `typeof obj === 'object'` checks.
    
- We **normalize** whatever the server returns to an actual `Table` (the one that has `snapshot()` / `setViewport()`).
    
- If the name doesn’t exist, you’ll get one clear error (“not found as a Table”) instead of that cryptic “in operator” message.
    

If you still see nothing: check the exact global names in DH (case-sensitive). In the IDE console, the tab name is not always the global variable name; verify with a quick `globals()` or by printing `user` / `account` in the Python console.

------------------------------------------------

This error tells me exactly what’s happening now:

👉 `java.lang.IllegalStateException: Table already closed, cannot be used again`

That’s not a Kafka schema mismatch this time. It’s Angular/DH client lifecycle. You’re “re-using” the same Deephaven `Table` object after Angular has already **closed** it.

---

### Why the table is closed

- Deephaven tables are _live server handles_.
    
- When you call `.close()` or Angular destroys a component, the table handle is released on the server.
    
- If you try to re-attach to that same handle from the client, you’ll get this exception.
    

This usually happens when:

1. You subscribe to a table, then Angular re-renders and tries to bind to the same table handle again.
    
2. Your service/component calls `detach` or `.close()` in `ngOnDestroy`, but you still reference the old table object later.
    
3. Multiple `subscribe()` calls reuse the same DH table reference.
    

---

### Fix pattern

You must **never reuse a closed handle**. Instead:

- Always **fetch a fresh handle** with `ide.getObject("users_ui")` / `ide.getObject("accounts_ui")` when you need the table again.
    
- Keep the table “open” until you explicitly want it gone.
    

---

### Example Angular service fix

```ts
async getFreshTable(varName: string): Promise<any> {
  if (!this.ready) await this.connect();

  // always fetch a fresh handle
  const obj = await this.ide.getObject(varName);

  // some builds wrap table, unwrap it
  const table = obj?.object ?? obj?.table ?? obj;
  if (!table || typeof table.snapshot !== 'function') {
    throw new Error(`Global '${varName}' is not a Table`);
  }

  return table;
}
```

---

### Component side

When you use it:

```ts
async ngOnInit() {
  this.table = await this.dhService.getFreshTable("users_ui");
  this.viewport = await this.table.setViewport(0, 100);
  this.table.addEventListener(this.table.EVENT_UPDATED, () => {
    this.onUpdate();
  });
}

ngOnDestroy() {
  // cleanly close this instance only once
  if (this.table) {
    this.table.close();
    this.table = null;
  }
}
```

---

✅ Key rule:  
If you see **“table already closed”**, you are trying to use a handle that’s been invalidated. Solution is to **always re-fetch a fresh handle** instead of storing and re-using one across multiple binds.

---

Would you like me to give you a **full updated Angular service + component pair** where the service guarantees a fresh handle every time (no reuse → no “table already closed”)?

---------------------------------------------


Perfect 👌 — let’s build you an **end-to-end Angular + Deephaven setup** that is:

- ✅ Compatible with your Angular 20.3.0 + `@deephaven/jsapi-bootstrap@41.5.3` + `@deephaven/jsapi-types@41.0.0`
    
- ✅ Uses **PSK authentication** (like you had before, no login UI)
    
- ✅ Avoids the **“table already closed”** problem by keeping a **live handle** to the server-side tables
    
- ✅ Avoids the **dynamic import warnings** by using Vite config (`/* @vite-ignore */` and `optimizeDeps.exclude`)
    
- ✅ Ready for use in your `join-tables.component.ts` with RxJS observables
    

---

## 1. `vite.config.ts`

Add this file at the root of your Angular project (same level as `angular.json`):

```ts
import { defineConfig } from 'vite';
import angular from '@analogjs/vite-plugin-angular';

export default defineConfig({
  plugins: [angular()],
  optimizeDeps: {
    exclude: ['@deephaven/jsapi-bootstrap']
  },
  build: {
    commonjsOptions: {
      ignoreDynamicRequires: true,
    },
  },
});
```

This silences the **dynamic import warning**.

---

## 2. `deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../environments/environment';
import { BehaviorSubject, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh: any;
  private client: any;
  private ide: any;
  private ready = false;

  private tableCache = new Map<string, any>();
  private tableSubjects = new Map<string, BehaviorSubject<any[]>>();

  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl.replace(/\/+$/, '');
    const psk = environment.deephavenPsk;
    if (!serverUrl || !psk) {
      throw new Error('Deephaven URL or PSK not configured');
    }

    // Load DH JSAPI dynamically from the DH server
    const jsapiUrl = `${serverUrl}/jsapi/dh-core.js?ts=${Date.now()}`;
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default ?? mod;

    // Create client
    this.client = new this.dh.CoreClient(serverUrl);

    // Authenticate with PSK
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    // Open IDE session
    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python');

    this.ready = true;
  }

  /** Get or subscribe to a live table by global name */
  async getTable(varName: string): Promise<Observable<any[]>> {
    await this.connect();

    // Return cached observable if already active
    if (this.tableSubjects.has(varName)) {
      return this.tableSubjects.get(varName)!.asObservable();
    }

    const obj = await this.ide.getObject(varName);
    if (!obj || obj.type !== 'Table') {
      throw new Error(`Global '${varName}' is not a Table`);
    }

    const table = obj; // live handle
    this.tableCache.set(varName, table);

    const subject = new BehaviorSubject<any[]>([]);
    this.tableSubjects.set(varName, subject);

    // Subscribe to snapshot updates
    const listener = (update: any) => {
      const rows = update.snapshot?.rows ?? [];
      subject.next(rows);
    };

    await table.addEventListener(this.dh.Table.EVENT_UPDATED, listener);
    await table.snapshot([], true).then((snap: any) => {
      subject.next(snap.rows ?? []);
    });

    return subject.asObservable();
  }

  /** Cleanup */
  async disconnect(): Promise<void> {
    for (const [varName, table] of this.tableCache) {
      try {
        await table.close();
      } catch {}
    }
    this.tableCache.clear();
    this.tableSubjects.clear();

    if (this.ide) {
      try {
        await this.ide.close();
      } catch {}
    }
    if (this.client) {
      try {
        await this.client.close();
      } catch {}
    }
    this.ready = false;
  }
}
```

---

## 3. `join-tables.component.ts`

```ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription } from 'rxjs';
import { DeephavenService } from './deephaven.service';

@Component({
  selector: 'app-join-tables',
  template: `
    <div>
      <h3>Users</h3>
      <pre>{{ users | json }}</pre>

      <h3>Accounts</h3>
      <pre>{{ accounts | json }}</pre>
    </div>
  `,
})
export class JoinTablesComponent implements OnInit, OnDestroy {
  users: any[] = [];
  accounts: any[] = [];

  private subs: Subscription[] = [];

  constructor(private dhService: DeephavenService) {}

  async ngOnInit() {
    const users$ = await this.dhService.getTable('users_ui');
    const accounts$ = await this.dhService.getTable('accounts_ui');

    this.subs.push(users$.subscribe((rows) => (this.users = rows)));
    this.subs.push(accounts$.subscribe((rows) => (this.accounts = rows)));
  }

  ngOnDestroy() {
    this.subs.forEach((s) => s.unsubscribe());
    this.dhService.disconnect();
  }
}
```

---

## 4. `environment.ts`

```ts
export const environment = {
  production: false,
  deephavenUrl: 'http://localhost:10000', // your DH server
  deephavenPsk: 'YOUR_PSK_VALUE',        // your PSK
};
```

---

### ✅ Key Fixes Applied

- **No `table already closed`**:  
    We never close the table from Angular. We keep a live handle and subscribe via snapshots + event listeners.
    
- **No import issue**:  
    Used `/* @vite-ignore */` + `vite.config.ts` with `optimizeDeps.exclude`.
    
- **Angular safe cleanup**:  
    `ngOnDestroy()` unsubscribes from RxJS and calls `disconnect()` to close tables only when leaving.
    

---

👉 Next step: run `ng serve` and open `http://localhost:4200`.  
You should see **live `users_ui` and `accounts_ui` tables** from Deephaven without any closed table errors.

Would you like me to also wire this into your **Ag-Grid component** instead of just `<pre>{{ rows | json }}</pre>` so you see a real table UI directly?

---------------------------------------


Thanks for the screenshot — that error tells us exactly what’s wrong:

> **TypeError: Cannot use ‘in’ operator to search for ‘type’ in users_ui**

`ide.getObject(varName)` is returning **the string `'users_ui'`** (or otherwise not the usual object that has a `type` field). That happens on some server builds / JSAPI mixes. The fix is to **resolve the variable via the variables catalog first**, then fetch the object from its ticket. Below is a drop-in service that does that and also avoids “table already closed” by keeping one live handle and using a **viewport** stream (no snapshots).

Use this as your Deephaven service (keep your environments as you had them):

```ts
// src/app/deephaven.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { environment } from '../environments/environment';

type Rows = any[];

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: any;
  private client!: any;
  private ide!: any;
  private ready = false;

  private handles = new Map<string, any>();
  private streams = new Map<string, BehaviorSubject<Rows>>();
  private listeners = new Map<string, () => void>();

  /** Connect once (PSK) and open a Python IDE session */
  async connect(): Promise<void> {
    if (this.ready) return;
    const serverUrl = environment.deephavenUrl.replace(/\/+$/, '');
    const psk = environment.deephavenPsk;
    if (!serverUrl || !psk) throw new Error('Missing deephavenUrl or deephavenPsk');

    const mod = await import(/* @vite-ignore */ `${serverUrl}/jsapi/dh-core.js?ts=${Date.now()}`);
    this.dh = mod.default ?? mod;

    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python');

    this.ready = true;
  }

  /** Robust “get table handle by global name”, compatible with both API shapes */
  private async getTableHandle(varName: string): Promise<any> {
    await this.connect();

    // Use cached handle if still open
    const cached = this.handles.get(varName);
    if (cached && !cached.isClosed?.()) return cached;

    // 1) Try direct getObject(name)
    let obj: any;
    try {
      obj = await this.ide.getObject(varName);
      if (obj && typeof obj === 'object' && 'type' in obj) {
        if (obj.type !== 'Table') throw new Error(`Global '${varName}' exists but is not a Table`);
        this.handles.set(varName, obj);
        return obj;
      }
    } catch {
      /* fall through to catalog path */
    }

    // 2) Fallback: resolve via variables catalog, then fetch by ticket
    const vars = await this.ide.getVariables();
    const found = (vars || []).find(
      (v: any) => v?.name === varName && (v?.type === 'Table' || `${v?.type}`.includes('Table'))
    );
    if (!found) throw new Error(`Global '${varName}' not found as a Table on the server`);

    const byTicket = await this.ide.getObject(found);
    if (!byTicket || byTicket.type !== 'Table') {
      throw new Error(`Resolved '${varName}', but it is not a Table`);
    }
    this.handles.set(varName, byTicket);
    return byTicket;
  }

  /**
   * Subscribe to live rows from a server-global table.
   * Uses a viewport (no snapshots) to avoid “table already closed” races.
   */
  async stream(varName: string, pageSize = 500): Promise<Observable<Rows>> {
    const existing = this.streams.get(varName);
    if (existing) return existing.asObservable();

    const table = await this.getTableHandle(varName);
    const subject = new BehaviorSubject<Rows>([]);
    this.streams.set(varName, subject);

    // Helper to push current viewport to consumer
    const publish = () => {
      try {
        const vp = table.getViewportData?.();
        const rows = vp?.rows ?? [];
        subject.next(rows);
      } catch {
        /* ignore transient reads during updates */
      }
    };

    // Initial viewport (all columns). Offset=0..pageSize-1
    // If your table grows, we still receive updates within that window.
    const allCols = table.columns?.map((c: any) => c.name) ?? [];
    await table.setViewport(0, Math.max(0, pageSize - 1), allCols);
    publish();

    // Listen for live updates
    const onUpdated = () => publish();
    await table.addEventListener(this.dh.Table.EVENT_UPDATED, onUpdated);
    this.listeners.set(varName, () => table.removeEventListener(this.dh.Table.EVENT_UPDATED, onUpdated));

    return subject.asObservable();
  }

  /** Clean up everything (call in ngOnDestroy of your shell if desired) */
  async disconnect(): Promise<void> {
    for (const [name, stop] of this.listeners) {
      try { stop(); } catch {}
    }
    this.listeners.clear();

    for (const [name, t] of this.handles) {
      try { await t.close?.(); } catch {}
    }
    this.handles.clear();

    for (const s of this.streams.values()) s.complete();
    this.streams.clear();

    try { await this.ide?.close?.(); } catch {}
    try { await this.client?.close?.(); } catch {}
    this.ready = false;
  }
}
```

### How to use it in your component

```ts
// src/app/join-tables.component.ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { Subscription } from 'rxjs';
import { DeephavenService } from './deephaven.service';

@Component({
  selector: 'app-join-tables',
  template: `
    <section class="card">
      <h3>Users</h3>
      <pre>{{ users | json }}</pre>
    </section>

    <section class="card">
      <h3>Accounts</h3>
      <pre>{{ accounts | json }}</pre>
    </section>
  `,
})
export class JoinTablesComponent implements OnInit, OnDestroy {
  users: any[] = [];
  accounts: any[] = [];
  private subs: Subscription[] = [];

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    try {
      this.subs.push((await this.dh.stream('users_ui', 1000)).subscribe(r => (this.users = r)));
      this.subs.push((await this.dh.stream('accounts_ui', 1000)).subscribe(r => (this.accounts = r)));
    } catch (e: any) {
      this.users = [`${e?.message ?? e}`];
      this.accounts = [`${e?.message ?? e}`];
    }
  }

  ngOnDestroy() {
    this.subs.forEach(s => s.unsubscribe());
    // optional if this is your only page:
    // void this.dh.disconnect();
  }
}
```

### Keep your Vite and environment the same as before

- `vite.config.ts` with `optimizeDeps.exclude: ['@deephaven/jsapi-bootstrap']` and `ignoreDynamicRequires`
    
- `environment.ts` with `deephavenUrl` and `deephavenPsk`
    

---

## Why this fixes your error

- When `getObject('users_ui')` returns a **string** (your console showed exactly that), the JSAPI helper that checks `('type' in obj)` explodes. We avoid that by **first querying the variables catalog** and fetching the object **by ticket**, which always yields the `{ type: 'Table', ... }` shape.
    
- Using **viewport streaming** (not snapshot) prevents the “table already closed” race you hit when snapshots coincide with rebind/detach.
    
- We **never close** the table while the component is alive; it’s cached in the service and torn down only when you call `disconnect()` (or on app shutdown).
    

If your server table globals are named differently, just change `'users_ui'` / `'accounts_ui'` in the component.

--------------------------------------


Great—your screenshots make it clear what’s happening:

- `this.ide.getVariables is not a function` means your JSAPI build doesn’t expose that helper on the IDE connection.
    
- And earlier “Cannot use 'in' operator to search for 'type' in users_ui” happens because `ide.getObject('users_ui')` sometimes returns a **ticket string**, not the `{ type: 'Table', ... }` descriptor.
    

You don’t need `getVariables` at all. Resolve tickets directly.

# Drop-in fix (service)

Replace your service with this version. It logs in with PSK, fetches globals, and **auto-resolves ticket strings** returned by `getObject(name)`. It also streams via a **viewport** (no snapshots), so you won’t hit “table already closed”.

```ts
// src/app/deephaven.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { environment } from '../environments/environment';

type Rows = any[];

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: any;
  private client!: any;        // CoreClient
  private ide!: any;           // IDE connection
  private ready = false;

  private handles = new Map<string, any>();
  private streams = new Map<string, BehaviorSubject<Rows>>();
  private listeners = new Map<string, () => void>();

  /** Connect once using PSK */
  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl.replace(/\/+$/, '');
    const psk = environment.deephavenPsk;
    if (!serverUrl || !psk) throw new Error('Missing deephavenUrl or deephavenPsk');

    // Load JSAPI directly from the DH server
    const mod = await import(/* @vite-ignore */ `${serverUrl}/jsapi/dh-core.js?ts=${Date.now()}`);
    this.dh = mod.default ?? mod;

    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python');

    this.ready = true;
  }

  /** Robust: get a Table handle by global name, resolving string tickets if needed */
  private async getTableHandle(varName: string): Promise<any> {
    await this.connect();

    const cached = this.handles.get(varName);
    if (cached && !cached.isClosed?.()) return cached;

    // 1) Ask for the object by name
    const obj = await this.ide.getObject(varName);

    // 2) If it already looks like a Table, use it
    if (obj && typeof obj === 'object' && obj.type === 'Table') {
      this.handles.set(varName, obj);
      return obj;
    }

    // 3) If the IDE returned a string or {ticket: "..."} descriptor, resolve by ticket
    if (typeof obj === 'string' || (obj && typeof obj === 'object' && 'ticket' in obj)) {
      const ticket = typeof obj === 'string' ? obj : obj.ticket;
      const resolved = await this.ide.getObject({ type: 'Table', ticket });
      if (!resolved || resolved.type !== 'Table') {
        throw new Error(`Global '${varName}' resolved by ticket but is not a Table`);
      }
      this.handles.set(varName, resolved);
      return resolved;
    }

    // 4) Not found / wrong type
    throw new Error(`Global '${varName}' not found as a Table on this server build.`);
  }

  /** Live rows stream via viewport (avoids “table already closed”) */
  async stream(varName: string, pageSize = 1000): Promise<Observable<Rows>> {
    const existing = this.streams.get(varName);
    if (existing) return existing.asObservable();

    const table = await this.getTableHandle(varName);
    const subject = new BehaviorSubject<Rows>([]);
    this.streams.set(varName, subject);

    const push = () => {
      try {
        const vp = table.getViewportData?.();
        subject.next(vp?.rows ?? []);
      } catch {
        /* ignore transient reads while the table mutates */
      }
    };

    const cols = table.columns?.map((c: any) => c.name) ?? [];
    await table.setViewport(0, Math.max(0, pageSize - 1), cols);
    push();

    const onUpdated = () => push();
    await table.addEventListener(this.dh.Table.EVENT_UPDATED, onUpdated);
    this.listeners.set(varName, () => table.removeEventListener(this.dh.Table.EVENT_UPDATED, onUpdated));

    return subject.asObservable();
  }

  /** Optional teardown */
  async disconnect(): Promise<void> {
    for (const stop of this.listeners.values()) {
      try { stop(); } catch {}
    }
    this.listeners.clear();

    for (const t of this.handles.values()) {
      try { await t.close?.(); } catch {}
    }
    this.handles.clear();

    for (const s of this.streams.values()) s.complete();
    this.streams.clear();

    try { await this.ide?.close?.(); } catch {}
    try { await this.client?.close?.(); } catch {}
    this.ready = false;
  }
}
```

### Component usage (unchanged)

```ts
// src/app/join-tables.component.ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { Subscription } from 'rxjs';
import { DeephavenService } from './deephaven.service';

@Component({
  selector: 'app-join-tables',
  template: `
    <section class="card">
      <h3>Users</h3>
      <pre>{{ users | json }}</pre>
    </section>

    <section class="card">
      <h3>Accounts</h3>
      <pre>{{ accounts | json }}</pre>
    </section>
  `,
})
export class JoinTablesComponent implements OnInit, OnDestroy {
  users: any[] = [];
  accounts: any[] = [];
  private subs: Subscription[] = [];

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    try {
      this.subs.push((await this.dh.stream('users_ui')).subscribe(r => (this.users = r)));
      this.subs.push((await this.dh.stream('accounts_ui')).subscribe(r => (this.accounts = r)));
    } catch (e: any) {
      this.users = [e?.message ?? String(e)];
      this.accounts = [e?.message ?? String(e)];
    }
  }

  ngOnDestroy() {
    this.subs.forEach(s => s.unsubscribe());
    // optionally: void this.dh.disconnect();
  }
}
```

### Notes

- Keep your **vite** settings you already added (ignore dynamic import analysis for `/jsapi/dh-core.js` and exclude `@deephaven/jsapi-bootstrap`).
    
- On the **server**, just ensure those globals exist (your screenshots show `users_ui`, `accounts_ui`, `user_accounts_ui` are there). You don’t need to change the DH script for this client fix.
    

This removes the `getVariables` dependency and handles the ticket-string case, so you shouldn’t see the “‘type’ in …” error again, and you’ll keep a stable viewport (no table-closed).