That flood of `ClassCastException` means your frontend is still calling methods (snapshot / setViewport) on a table/viewport that Deephaven just closed (typically because the Python cell was rerun or the global got replaced). Multiple concurrent retries then make it worse.

Fix it by making the component **self-healing and de-duped**:

- Treat “closed” as a terminal state for the current handle.
    
- Stop all listeners immediately.
    
- Re-acquire a fresh handle (by name), reopen the viewport, and resume.
    
- Ensure only **one** reattach happens at a time (no storms).
    
- Ignore late events from the old handle (generation token).
    

Below is a drop-in replacement for your `LiveTableComponent` that implements that. Your `DeephavenService` from the last step is fine — keep PSK login as-is.

---

### `src/app/live-table/live-table.component.ts` (robust, self-healing)

```ts
import {
  Component, Input, OnInit, OnDestroy,
  ChangeDetectionStrategy, ChangeDetectorRef
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { DeephavenService, DhTable } from '../deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  imports: [CommonModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
  <section class="card">
    <header class="bar">
      <h3><span class="mono">{{ tableName }}</span></h3>
      <div class="status">
        <span *ngIf="error" class="error">{{ error }}</span>
        <span *ngIf="!error && loading">Connecting…</span>
      </div>
    </header>

    <div class="grid" *ngIf="!error">
      <div class="thead" *ngIf="cols.length">
        <div class="th" *ngFor="let c of cols; trackBy: trackCol">{{ c }}</div>
      </div>

      <div class="tbody" (scroll)="onScroll($event)">
        <div class="tr" *ngFor="let r of rows; trackBy: trackRow">
          <div class="td" *ngFor="let v of r; trackBy: trackCell">{{ v }}</div>
        </div>
      </div>
    </div>

    <footer class="pager" *ngIf="!error">
      <button (click)="pageFirst()" [disabled]="offset === 0">First</button>
      <button (click)="pagePrev()"  [disabled]="offset === 0">Prev</button>
      <span class="mono">rows {{ offset }}–{{ offset + pageSize - 1 }}</span>
      <button (click)="pageNext()">Next</button>
      <button (click)="refresh()">Refresh</button>
    </footer>
  </section>
  `,
  styles: [`
    :host { display:block }
    .card { border:1px solid #2a2a2a; border-radius:.75rem; background:#0f0f0f; color:#fff; }
    .bar { display:flex; justify-content:space-between; align-items:center; padding:.75rem 1rem; border-bottom:1px solid #242424 }
    .mono{ font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace }
    .grid{ display:grid; grid-template-rows: auto 1fr; gap:.5rem; }
    .thead, .tr { display:grid; grid-auto-flow:column; grid-auto-columns:minmax(140px, 1fr) }
    .th { position:sticky; top:0; background:#111; font-weight:600; padding:.5rem; z-index:1; }
    .tbody{ max-height:40dvh; overflow:auto; }
    .td { padding:.25rem .5rem; border-top:1px solid #242424; white-space:nowrap; overflow:hidden; text-overflow:ellipsis }
    .pager{ display:flex; gap:.5rem; align-items:center; padding:.5rem 1rem; border-top:1px solid #242424 }
    .error{ color:#ff6b6b }
  `],
})
export class LiveTableComponent implements OnInit, OnDestroy {
  /** name of DH globals() table, e.g. "user_account" */
  @Input({ required: true }) tableName!: string;
  @Input() pageSize = 300;
  @Input() initialOffset = 0;

  loading = false;
  error: string | null = null;

  cols: string[] = [];
  rows: any[] = [];
  offset = 0;

  // DH handles
  private table: DhTable | null = null;
  private viewport: any;

  // listeners
  private onUpdateHandler?: () => void;
  private onSchemaHandler?: () => void;

  // self-heal flags
  private generation = 0;
  private reconnecting = false;
  private destroyed = false;

  constructor(private dh: DeephavenService, private cdr: ChangeDetectorRef) {}

  async ngOnInit(): Promise<void> {
    this.offset = this.initialOffset;
    await this.attach(true);
  }
  async ngOnDestroy(): Promise<void> { this.destroyed = true; await this.detach(); }

  // ---------- attach/detach ----------

  private async attach(first = false): Promise<void> {
    const myGen = ++this.generation;
    this.loading = true; if (first) this.error = null; this.cdr.markForCheck();

    try {
      await this.dh.connect();
      const dhns = this.dh.getDhNs();

      // get fresh handle
      const t = await this.dh.getTableHandle(this.tableName);
      if (this.destroyed || myGen !== this.generation) return;

      this.table = t;
      this.cols = (this.table.columns ?? []).map((c: any) => c.name);

      // open viewport
      this.viewport = await this.table.setViewport(
        this.offset, this.offset + this.pageSize - 1, this.cols
      );

      // initial snapshot
      await this.refresh();

      // subscribe to updates
      const EVU = dhns?.Table?.EVENT_UPDATED ?? 'update';
      const EVS = dhns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed';

      this.onUpdateHandler = async () => {
        if (this.destroyed || myGen !== this.generation) return;
        await this.refresh();
      };
      this.onSchemaHandler = async () => {
        if (this.destroyed || myGen !== this.generation) return;
        await this.onSchemaChanged();
      };

      this.table.addEventListener(EVU, this.onUpdateHandler);
      this.table.addEventListener(EVS, this.onSchemaHandler);

      this.loading = false; this.cdr.markForCheck();
    } catch (e: any) {
      if (this.destroyed || myGen !== this.generation) return;
      this.loading = false;
      this.error = e?.message ?? String(e);
      this.cdr.markForCheck();
    }
  }

  private async detach(): Promise<void> {
    try {
      const dhns = this.dh.getDhNs();
      const EVU = dhns?.Table?.EVENT_UPDATED ?? 'update';
      const EVS = dhns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed';
      if (this.table && this.onUpdateHandler) this.table.removeEventListener?.(EVU, this.onUpdateHandler);
      if (this.table && this.onSchemaHandler) this.table.removeEventListener?.(EVS, this.onSchemaHandler);
    } catch {}
    this.onUpdateHandler = undefined;
    this.onSchemaHandler = undefined;
    try { await this.viewport?.close?.(); } catch {}
    // Never close the table (server global)
    this.viewport = null;
    this.table = null;
  }

  // ---------- helpers ----------

  private isClosedError(e: any): boolean {
    const msg = (e?.message ?? String(e)).toLowerCase();
    return msg.includes('closed') || msg.includes('table already closed');
  }

  private async handleClosedAndReattach(): Promise<void> {
    if (this.reconnecting || this.destroyed) return;
    this.reconnecting = true;
    try {
      await this.detach();
      // small delay to avoid hot-looping while server is swapping globals
      await new Promise(r => setTimeout(r, 150));
      await this.attach();
    } finally {
      this.reconnecting = false;
    }
  }

  // ---------- data ----------

  async refresh(): Promise<void> {
    if (!this.table) return;
    try {
      const snap = await this.table.snapshot(this.cols);
      const objs = snap.toObjects?.() ?? snap;
      this.rows = (objs as any[]).map(o =>
        Array.isArray(o) ? o : this.cols.map(c => (o as any)[c])
      );
      this.cdr.markForCheck();
    } catch (e: any) {
      if (this.isClosedError(e)) {
        // stop the flood of ClassCastException by switching generation once
        await this.handleClosedAndReattach();
      } else {
        this.error = e?.message ?? String(e);
        this.cdr.markForCheck();
      }
    }
  }

  private async onSchemaChanged(): Promise<void> {
    if (!this.table) return;
    try {
      this.cols = (this.table.columns ?? []).map((c: any) => c.name);
      await this.table.setViewport(this.offset, this.offset + this.pageSize - 1, this.cols);
      await this.refresh();
    } catch (e) {
      if (this.isClosedError(e)) {
        await this.handleClosedAndReattach();
      }
    }
  }

  // ---------- paging & scroll ----------
  async pageFirst() { this.offset = 0; await this.onPage(); }
  async pagePrev()  { this.offset = Math.max(0, this.offset - this.pageSize); await this.onPage(); }
  async pageNext()  { this.offset = this.offset + this.pageSize; await this.onPage(); }
  private async onPage() {
    if (!this.table) return;
    try {
      await this.table.setViewport(this.offset, this.offset + this.pageSize - 1, this.cols);
      await this.refresh();
    } catch (e) {
      if (this.isClosedError(e)) {
        await this.handleClosedAndReattach();
      }
    }
  }

  onScroll(ev: Event) {
    const el = ev.target as HTMLElement;
    if (el.scrollTop + el.clientHeight >= el.scrollHeight - 8) this.pageNext();
  }

  // ---------- trackBys ----------
  trackCol = (_: number, c: string) => c;
  trackRow = (_: number, r: any[]) => r?.[0] ?? _;
  trackCell = (_: number, _v: any) => _;
}
```

**What changed vs. your previous component**

- `generation` token prevents late events from an old table from touching state.
    
- `reconnecting` flag guarantees only **one** reattach in flight.
    
- Any “closed” error from `snapshot`/`setViewport`/schema update triggers `handleClosedAndReattach()`, which:
    
    - detaches listeners immediately,
        
    - waits 150ms (lets the server finish swapping),
        
    - re-acquires a **fresh** handle by name and re-subscribes.
        

That stops the ClassCastException storm you’re seeing and keeps the grid live even when the DH script is re-run.

---

### Double-check the server script

Make sure your Python defines stable globals **once** per session (or with `if 'user_raw' not in globals():` guards). Re-running is fine; the component above will reattach cleanly. But avoid creating dozens of new variables or closing the globals in a loop.

---

If you still see a burst after a cell re-run, it should drop to a single reconnect cycle (one short “Connecting…” blip) and then resume streaming.


Got it — let’s avoid any `if … in globals()` guards and any helpers. Here’s a **single top-level cell** that Deephaven will execute cleanly. It:

- **closes** any previous variables if they exist (no errors if they don’t),
    
- **recreates** the Kafka consumers every run,
    
- builds the account-driven **as-of** join (`aj` with `ts`),
    
- leaves three globals: `user`, `account`, `user_account`.
    

Paste this in one DH Python cell and run it as-is.

```python
# -------------------- Imports --------------------
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# -------------------- Optional: clean up any prior run --------------------
for _n in ("user_account", "users_hist", "user", "account", "user_raw", "account_raw"):
    try:
        globals()[_n].close()        # close old table if present
    except Exception:
        pass
    finally:
        globals().pop(_n, None)      # and remove the name

# -------------------- Kafka config (plain strings only) --------------------
TOPIC_USERS   = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
TOPIC_ACCTS   = "ccd01_sb_its_esp_tap3507_bishowcasescurated"

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "auto.offset.reset": "earliest",
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url":
        "https://fdsit.rastest.tdbank.ca/as/token.oauth2",
    # add your other OAuth properties here as single Python strings
    # e.g. client id/secret, scopes, etc.
}

# -------------------- JSON value specs (must be kc.json_spec) --------------------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
    "age":    dt.int64,   # use dt.int32 if that’s what your topic carries
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double,
})

# -------------------- Create Kafka append tables (unconditional) --------------------
user_raw = kc.consume(
    KAFKA_CONFIG,
    TOPIC_USERS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=USER_VALUE_SPEC,
    table_type=kc.TableType.append,
)

account_raw = kc.consume(
    KAFKA_CONFIG,
    TOPIC_ACCTS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=ACCOUNT_VALUE_SPEC,
    table_type=kc.TableType.append,
)

# -------------------- Normalize timestamps --------------------
# Kafka consumers expose KafkaTimestamp; rename to 'ts' for the as-of key.
user    = user_raw.rename_columns(["ts = KafkaTimestamp"])
account = account_raw.rename_columns(["ts = KafkaTimestamp"])

# -------------------- Account-driven as-of join --------------------
# New row only when *account* ticks; enrich with latest user fields as-of that ts.
users_hist   = user.sort_descending("ts")
user_account = account.aj(users_hist, on=["userId", "ts"], joins=["name", "email", "age"])

# Optional: simple readiness flag
PIPELINE_READY = True

# (Optional sanity)
# print("sizes:", user.size(), account.size(), user_account.size())
# print("joined cols:", [c.name for c in user_account.columns])
```

### Why this will work in your DH session

- **No functions, no conditionals** around the consumers — just top-level Python like DH’s docs/examples.
    
- Safe **cleanup first**, then unconditional recreate, so there’s never a half-constructed handle.
    
- Using only **supported types**: `kc.json_spec(...)`, `kc.KeyValueSpec.IGNORE`, `kc.TableType.append`, string config.
    
- The **as-of** join (`aj`) is exactly your requirement: append a new joined row when an account event arrives, pulling user fields as-of that event time.
    

Your Angular component that opens a **viewport** and listens for **EVENT_UPDATED** will now tick automatically. Since we recreate tables on every run, keep the self-healing logic in the component (reattach if it sees “closed”), and you won’t get the ClassCastException storms anymore.