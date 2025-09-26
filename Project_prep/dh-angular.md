Short answer: you **don’t** need to “host an API.” If your Deephaven server is already ingesting Kafka and producing a **live (ticking) table**, your Angular app can connect **directly** to the Deephaven server via the **JS API** (which uses a WebSocket under the hood) and stream updates in real-time. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))

If you’ve **disabled auth** (anonymous), you can skip login entirely. (Just know that anonymous access has no security.) ([Deephaven](https://deephaven.io/core/groovy/docs/how-to-guides/authentication/auth-anon/?utm_source=chatgpt.com "Enable anonymous authentication"))

---

## Minimal Angular setup (no auth, real-time streaming)

**Install**:

```bash
npm i @deephaven/jsapi-bootstrap @deephaven/jsapi-types
```

**`src/environments/environment.ts`**

```ts
export const environment = {
  DH_BASE_URL: 'http://localhost:10000', // ← your DH server
  TABLE_NAME: 'live_table'               // ← the published table variable created by your Kafka script
};
```

**`src/app/deephaven.service.ts`**

```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import type { CoreClient, IdeConnection, Table } from '@deephaven/jsapi-types';
import * as Bootstrap from '@deephaven/jsapi-bootstrap';
import { environment } from '../environments/environment';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private client!: CoreClient;
  private table!: Table;
  private rows$ = new BehaviorSubject<any[]>([]);
  private keys: string[] = [];

  getRows$(): Observable<any[]> { return this.rows$.asObservable(); }
  getKeys(): string[] { return this.keys; }

  /** Connect to DH and subscribe to an existing live table created by your Kafka ingester */
  async connectAndSubscribe(): Promise<void> {
    const baseUrl = environment.DH_BASE_URL;

    // Load the JS API that Deephaven serves; falls back to direct import if needed.
    const dh =
      (await (Bootstrap as any).loadApi?.({ baseUrl })) ??
      (await (Bootstrap as any).bootstrapApi?.({ baseUrl })) ??
      (await import(/* @vite-ignore */ `${baseUrl}/jsapi/dh-core.js`)).default;

    // Create client (no auth step if anonymous is enabled)
    this.client = new dh.CoreClient(baseUrl);

    // Open an IDE session to access the published table variable
    const ideConn: IdeConnection = await this.client.getAsIdeConnection();
    const ide = await ideConn.startSession('python');

    // IMPORTANT: TABLE_NAME must match the variable exposed by your Kafka script (e.g., live_table)
    this.table = await ide.getTable(environment.TABLE_NAME);

    // Use a viewport for efficient real-time streaming to the UI
    this.table.setViewport(0, 99); // first 100 rows (tweak as needed)

    const pushViewport = async () => {
      const vp = await this.table.getViewportData();
      const cols = this.table.columns;
      this.keys = cols.map((c: any) => c.name);
      const data = vp.rows.map((r: any) => {
        const obj: Record<string, any> = {};
        cols.forEach((c: any) => (obj[c.name] = r.get(c)));
        return obj;
      });
      this.rows$.next(data);
    };

    await pushViewport();
    this.table.addEventListener('update', pushViewport); // real-time deltas over WebSocket
  }
}
```

**`src/app/live-table.component.ts`**

```ts
import { Component } from '@angular/core';
import { DeephavenService } from './deephaven.service';

@Component({
  selector: 'app-live-table',
  template: `
    <button (click)="connect()" [disabled]="connected">
      {{ connected ? 'Connected' : 'Connect to Deephaven' }}
    </button>

    <table *ngIf="rows.length" style="margin-top:1rem; border-collapse:collapse;" border="1">
      <thead><tr><th *ngFor="let k of keys">{{ k }}</th></tr></thead>
      <tbody>
        <tr *ngFor="let r of rows">
          <td *ngFor="let k of keys">{{ r[k] }}</td>
        </tr>
      </tbody>
    </table>
  `,
})
export class LiveTableComponent {
  rows: any[] = [];
  keys: string[] = [];
  connected = false;

  constructor(private dh: DeephavenService) {}

  async connect() {
    await this.dh.connectAndSubscribe();
    this.connected = true;
    this.keys = this.dh.getKeys();
    this.dh.getRows$().subscribe(rows => (this.rows = rows));
  }
}
```

That’s it. No extra API to host, no polling, and it streams **live**.

---

## FAQs

- **Do I need WebSocket or HTTP?**  
    The JS API uses **WebSocket** internally for live/ticking tables; you don’t manage it yourself. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))
    
- **Do I need authentication?**  
    Not if you’ve enabled **anonymous** access (common in dev). Just be aware this provides **no security**. ([Deephaven](https://deephaven.io/core/groovy/docs/how-to-guides/authentication/auth-anon/?utm_source=chatgpt.com "Enable anonymous authentication"))
    
- **Will my Kafka tables be “live”?**  
    Yes. The Kafka ingester writes into **live/ticking** tables that you can subscribe to exactly like above. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/overview-kafka/?utm_source=chatgpt.com "Kafka Overview"))
    

If your Kafka script names the table something other than `live_table`, just set `TABLE_NAME` to that variable. If you hit CORS while importing `…/jsapi/dh-core.js`, add a dev proxy or host the Angular app on the same origin as Deephaven. ([Deephaven](https://deephaven.io/core/groovy/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))