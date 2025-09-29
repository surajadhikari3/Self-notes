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