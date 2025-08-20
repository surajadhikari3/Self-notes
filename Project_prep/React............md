


Perfect 👌 thanks for showing me your repo — I see you already have shared building blocks like:

- **`PaginationTable`** (shared table with paging, likely sortable/filterable)
    
- **`Button`** (generic button with variants & icons)
    
- **`Tag`** (for rendering schema tags nicely)
    
- **Type definitions** (`ColumnRow`, etc.)
    

So instead of reinventing anything, we’ll just wire these together.

Here’s an updated **DataSchemaPage.tsx** using only existing components, and falling back to simple generic ones where nothing exists yet:

```tsx
// src/features/publish/pages/DataSchemaPage.tsx

import React, { useState } from "react";
import PaginationTable from "@/shared/components/PaginationTable";
import Button from "@/shared/components/Button/Button";
import Tag from "@/shared/components/Tag/Tag"; // you already have this
import { ColumnRow } from "../pageTypes";

const DataSchemaPage: React.FC = () => {
  // TODO: Replace mock data with API fetch later
  const [rows, setRows] = useState<ColumnRow[]>([
    {
      fieldName: "id",
      displayName: "Identifier",
      description: "Unique record identifier",
      type: "string",
      tags: ["primary-key"]
    },
    {
      fieldName: "created_at",
      displayName: "Created At",
      description: "Timestamp of record creation",
      type: "timestamp",
      tags: ["system", "audit"]
    },
    {
      fieldName: "amount",
      displayName: "Transaction Amount",
      description: "The monetary value of the transaction",
      type: "decimal",
      tags: ["finance"]
    },
  ]);

  // search state (if PaginationTable doesn’t already support search)
  const [search, setSearch] = useState("");

  const filtered = rows.filter((r) =>
    [r.fieldName, r.displayName, r.description, r.type]
      .join(" ")
      .toLowerCase()
      .includes(search.toLowerCase())
  );

  // column config for PaginationTable
  const columns = [
    { key: "fieldName", label: "Field Name" },
    { key: "displayName", label: "Display Name" },
    { key: "description", label: "Description" },
    { key: "type", label: "Type" },
    {
      key: "tags",
      label: "Tags",
      render: (tags: string[] | null) =>
        tags?.map((tag) => <Tag key={tag} label={tag} />) || "—",
    },
  ];

  return (
    <div style={{ padding: "1rem" }}>
      <h2>Data Schema</h2>

      {/* Search bar */}
      <div style={{ marginBottom: "1rem", display: "flex", gap: "0.5rem" }}>
        <input
          type="text"
          placeholder="Search schema..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
          style={{
            flex: 1,
            padding: "0.5rem",
            borderRadius: "4px",
            border: "1px solid #ccc",
          }}
        />
        <Button
          onClick={() => console.log("TODO: open add field modal")}
          variant="filled"
        >
          + Add Field
        </Button>
      </div>

      {/* Table */}
      <PaginationTable
        data={filtered}
        columns={columns}
        pageSize={10}
      />
    </div>
  );
};

export default DataSchemaPage;
```

---

### 🔍 About Search

- If your `PaginationTable` **already supports `searchable`**, you can remove the `<input>` block and just pass `searchable` as a prop.
    
- If not, the above code does manual filtering.
    

---

✅ This page now:

- Uses your **`PaginationTable`** to render rows
    
- Uses your **`Tag`** for tags
    
- Uses your **`Button`** for the “Add Field” action
    
- Falls back to a plain `<input>` for search (only if not built-in)
    

---

Do you want me to open up your **PaginationTable.tsx** and adapt this so we know whether search is already built-in, or keep the manual search fallback?



-----------------------------


Totally—let’s keep the **same validation behavior** (block “Next” + toast) and also use your **existing shared components**: `PaginationTable`, `Button`, `FormTextField`, `FormSelectField`. The table doesn’t provide search, so we pre‑filter the rows before passing them in.

Below is a drop‑in **DataSchema.tsx** wired to your Context API (`pushRow`, `updateRowAt`, `removeRowAt`, `publishStepValidator`, `goNext`, `isNextBlocked`, `toast`, `hideToast`) and your types (`ColumnRow`). It reuses your shared UI pieces everywhere possible.

```tsx
// src/features/publish/steps/DataSchema.tsx
import React, { useEffect, useMemo, useState } from 'react';
import PaginationTable from '@/shared/components/PaginationTable';
import type { Column } from '@/shared/components/PaginationTable/type/types';
import Button from '@/shared/components/Button/Button';
import { FormTextField } from '@/shared/components/FormFields/FormTextField';
import { FormSelectField } from '@/shared/components/FormFields/FormSelectField';
import { usePublication } from '../context';
import type { ColumnRow } from '../pageTypes';

// ---------- validation ----------
type ValidationResult =
  | { ok: true }
  | { ok: false; message?: string; focusSelector?: string };

function validateDataSchema(rows?: ColumnRow[]): ValidationResult {
  const list = rows ?? [];

  if (list.length === 0) {
    return { ok: false, message: 'Add at least one field.', focusSelector: '#add-field' };
  }

  // required columns
  const badIdx = list.findIndex(
    r => !r.fieldName?.trim() || !r.displayName?.trim() || !r.type?.trim()
  );
  if (badIdx >= 0) {
    return {
      ok: false,
      message: `Complete required columns in row ${badIdx + 1}.`,
      focusSelector: `[data-row="${badIdx}"] [data-col="fieldName"]`,
    };
  }

  // duplicate field names
  const seen = new Set<string>();
  for (const r of list) {
    const k = r.fieldName.trim().toLowerCase();
    if (seen.has(k)) return { ok: false, message: `Duplicate field name: ${r.fieldName}` };
    seen.add(k);
  }

  return { ok: true };
}

// adjust to your real lists
const TYPE_OPTIONS = [
  { label: 'UUID', value: 'UUID' },
  { label: 'VARCHAR', value: 'VARCHAR' },
  { label: 'CHAR', value: 'CHAR' },
  { label: 'DATE', value: 'DATE' },
  { label: 'NUMBER', value: 'NUMBER' },
];

const TAG_OPTIONS = [
  { label: 'PII', value: 'PII' },
  { label: 'Sensitive', value: 'Sensitive' },
  { label: 'None', value: 'None' },
];

export default function DataSchema() {
  const {
    form,
    pushRow,
    updateRowAt,
    removeRowAt,
    publishStepValidator,
    isNextBlocked,
    goNext,
    toast,
    hideToast,
  } = usePublication();

  const rows = form.dataSchema?.rows ?? [];
  const [query, setQuery] = useState('');

  // register validation for this step (provider triggers this on Next)
  useEffect(() => publishStepValidator(() => validateDataSchema(rows)), [rows, publishStepValidator]);

  // --- Search (PaginationTable doesn’t include this) ---
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return rows;
    return rows.filter(r =>
      r.fieldName.toLowerCase().includes(q) ||
      r.displayName.toLowerCase().includes(q) ||
      (r.description ?? '').toLowerCase().includes(q) ||
      (r.tags ?? []).some(t => t.toLowerCase().includes(q))
    );
  }, [rows, query]);

  // helpers using your generic row APIs
  const patchRow = (index: number, patch: Partial<ColumnRow>) =>
    updateRowAt<ColumnRow>({
      selector: f => f.dataSchema!.rows ?? [],
      writer: (next, arr) => {
        next.dataSchema ??= { rows: [] };
        next.dataSchema.rows = arr;
      },
      index,
      patch,
    });

  const addRow = () =>
    pushRow<ColumnRow>({
      selector: f => f.dataSchema!.rows ?? [],
      writer: (next, arr) => {
        next.dataSchema ??= { rows: [] };
        next.dataSchema.rows = arr;
      },
      row: { fieldName: '', displayName: '', description: '', type: 'VARCHAR', tags: null },
    });

  const deleteRow = (index: number) =>
    removeRowAt<ColumnRow>({
      selector: f => f.dataSchema!.rows ?? [],
      writer: (next, arr) => {
        next.dataSchema ??= { rows: [] };
        next.dataSchema.rows = arr;
      },
      index,
    });

  // --- Columns for shared PaginationTable ---
  const columns: Column[] = [
    {
      key: 'fieldName',
      header: 'Field Name',
      sortable: true,
      sortAccessor: r => r.fieldName?.toLowerCase?.() ?? '',
      render: (_val, row, rowIndex) => (
        <div data-row={rowIndex} data-col="fieldName">
          <FormTextField
            value={row.fieldName}
            onChange={(v) => patchRow(rowIndex, { fieldName: v })}
            placeholder="e.g., transaction_id"
            disabled={false}
            name="fieldName"
            className=""
            icon=""
          />
        </div>
      ),
    },
    {
      key: 'displayName',
      header: 'Display Name',
      sortable: true,
      render: (_val, row, rowIndex) => (
        <FormTextField
          value={row.displayName}
          onChange={(v) => patchRow(rowIndex, { displayName: v })}
          placeholder="e.g., Transaction ID"
          disabled={false}
          name="displayName"
          className=""
          icon=""
        />
      ),
    },
    {
      key: 'description',
      header: 'Description',
      sortable: true,
      render: (_val, row, rowIndex) => (
        <FormTextField
          value={row.description}
          onChange={(v) => patchRow(rowIndex, { description: v })}
          placeholder="Describe the column…"
          disabled={false}
          name="description"
          className=""
          icon=""
        />
      ),
    },
    {
      key: 'type',
      header: 'Type',
      sortable: true,
      sortAccessor: r => r.type,
      render: (_val, row, rowIndex) => (
        <FormSelectField
          value={row.type}
          onChange={(v) => patchRow(rowIndex, { type: v })}
          options={TYPE_OPTIONS}
          placeholder="Select type"
          disabled={false}
          name="type"
          className=""
        />
      ),
    },
    {
      key: 'tags',
      header: 'Tags',
      sortable: true,
      sortAccessor: r => (r.tags?.[0] ?? 'None'),
      render: (_val, row, rowIndex) => (
        <FormSelectField
          value={(row.tags?.[0]) ?? 'None'}
          onChange={(v) => patchRow(rowIndex, { tags: v === 'None' ? null : [v] })}
          options={TAG_OPTIONS}
          placeholder="Select tag"
          disabled={false}
          name="tags"
          className=""
        />
      ),
    },
    {
      key: 'actions',
      header: ' ',
      sortable: false,
      render: (_val, _row, rowIndex) => (
        <Button variant="borderless" onClick={() => deleteRow(rowIndex)}>
          Remove
        </Button>
      ),
    },
  ];

  return (
    <div className="p-6">
      <h2 className="text-xl font-semibold mb-2">Data Schema</h2>
      <p className="text-sm text-gray-600 mb-4">
        Specify how the dataset should be classified and governed.
      </p>

      {toast?.visible && (
        <div className="mb-3 rounded border border-amber-300 bg-amber-50 p-2 text-sm">
          {toast.message}
          <Button variant="borderless" onClick={hideToast}>Dismiss</Button>
        </div>
      )}

      <div className="flex gap-3 items-center mb-3">
        {/* search */}
        <div style={{ minWidth: 280 }}>
          <FormTextField
            value={query}
            onChange={setQuery}
            placeholder="Search fields…"
            disabled={false}
            name="schemaSearch"
            className=""
            icon=""
          />
        </div>

        {/* add field */}
        <Button id="add-field" variant="filled" onClick={addRow}>
          + Add field
        </Button>

        <span className="text-sm text-gray-500">{rows.length} total</span>
      </div>

      <PaginationTable
        columns={columns}
        data={filtered}
        initialPageSize={10}
      />

      <div className="mt-6 flex justify-end gap-2">
        <Button variant="standard" onClick={() => history.back()}>Back</Button>
        <Button variant="filled" onClick={goNext} >
          Next
        </Button>
      </div>
    </div>
  );
}
```

### What you get

- **Validation** identical to what we discussed: the page registers `publishStepValidator(() => validateDataSchema(rows))`; your provider blocks **Next** and shows the toast.
    
- **Shared components** everywhere: `PaginationTable`, `Button`, `FormTextField`, `FormSelectField`.
    
- **Search** implemented by pre‑filtering rows (since the table itself doesn’t include search).
    
- **Inline edit** inside table cells, writing to `form.dataSchema.rows` through your generic context helpers.
    

If you also want the **“Items per page / 1–13”** footer like in your mock, set `initialPageSize={10}` (already done) and let your `PaginationTable` render its pager; it matches the shared sample in your repo.


---------------------------------------------


You’ve got two separate issues:

1. **`FormTextField` imported with `import type`**  
    → Components must be regular imports (runtime values), not type-only.
    
2. **`updateRowAt / pushRow / removeRowAt` “Expected 4/3 args”**  
    → Your context functions take **positional arguments**, not an options object.
    

Below is a **working DataSchema.tsx** that fixes both and uses only your shared components (`PaginationTable`, `Button`, `FormTextField`, `FormSelectField`) + your Context helpers.

---

### DataSchema.tsx (drop‑in)

```tsx
import React, { useEffect, useMemo, useState } from "react";
import { usePublication } from "../context";
import type { ColumnRow } from "../pageTypes";

// ✅ components must be regular imports (NOT "import type")
import Button from "@/shared/components/Button/Button";
import FormTextField from "@/shared/components/FormFields/FormTextField";
import FormSelectField from "@/shared/components/FormFields/FormSelectField";
import PaginationTable from "@/shared/components/PaginationTable/PaginationTable";
import type { Column } from "@/shared/type/types";

// ---- simple validator ----
type ValidationResult =
  | { ok: true }
  | { ok: false; message?: string; focusSelector?: string };

function validateDataSchema(rows?: ColumnRow[]): ValidationResult {
  const list = rows ?? [];
  if (!list.length) return { ok: false, message: "Add at least one field.", focusSelector: "#add-field" };
  const badIdx = list.findIndex(r => !r.fieldName?.trim() || !r.displayName?.trim() || !r.type?.trim());
  if (badIdx >= 0) {
    return {
      ok: false,
      message: `Complete required columns in row ${badIdx + 1}.`,
      focusSelector: `[data-row="${badIdx}"] [data-col="fieldName"]`,
    };
  }
  const seen = new Set<string>();
  for (const r of list) {
    const k = r.fieldName.trim().toLowerCase();
    if (seen.has(k)) return { ok: false, message: `Duplicate field name: ${r.fieldName}` };
    seen.add(k);
  }
  return { ok: true };
}

const TYPE_OPTIONS = [
  { label: "UUID", value: "UUID" },
  { label: "VARCHAR", value: "VARCHAR" },
  { label: "CHAR", value: "CHAR" },
  { label: "DATE", value: "DATE" },
  { label: "NUMBER", value: "NUMBER" },
];

export default function DataSchema() {
  const {
    form,
    pushRow,
    updateRowAt,
    removeRowAt,
    publishStepValidator,
    isNextBlocked,
    goNext,
    toast,
    hideToast,
  } = usePublication();

  // use the correct key: dataSchema
  const rows = form.dataSchema?.rows ?? [];
  const [query, setQuery] = useState("");

  // register validator
  useEffect(() => publishStepValidator(() => validateDataSchema(rows)), [rows, publishStepValidator]);

  // --- Search (table doesn't include it) ---
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return rows;
    return rows.filter(r =>
      r.fieldName.toLowerCase().includes(q) ||
      r.displayName.toLowerCase().includes(q) ||
      (r.description ?? "").toLowerCase().includes(q) ||
      (r.tags ?? []).some(t => t.toLowerCase().includes(q))
    );
  }, [rows, query]);

  // ---- shared selector/writer (positional args!) ----
  const selectRows = (f: any): ColumnRow[] | undefined => f.dataSchema?.rows;
  const writeRows = (f: any, next: ColumnRow[]) => {
    f.dataSchema ??= { rows: [] };
    f.dataSchema.rows = next;
  };

  // helpers using positional signatures from context
  const patchRow = (index: number, patch: Partial<ColumnRow>) =>
    updateRowAt<ColumnRow>(selectRows, writeRows, index, patch);

  const addRow = () =>
    pushRow<ColumnRow>(selectRows, writeRows, {
      fieldName: "",
      displayName: "",
      description: "",
      type: "VARCHAR",
      tags: null,
    });

  const deleteRow = (index: number) =>
    removeRowAt<ColumnRow>(selectRows, writeRows, index);

  // ---- columns for PaginationTable ----
  const columns: Column[] = [
    {
      key: "fieldName",
      header: "Field Name",
      sortable: true,
      sortAccessor: (r: ColumnRow) => r.fieldName?.toLowerCase?.() ?? "",
      render: (_v: any, row: ColumnRow, rowIndex: number) => (
        <div data-row={rowIndex} data-col="fieldName">
          <FormTextField
            value={row.fieldName}
            onChange={(v: string) => patchRow(rowIndex, { fieldName: v })}
            placeholder="e.g., transaction_id"
            disabled={false}
            name="fieldName"
            className=""
            icon=""
          />
        </div>
      ),
    },
    {
      key: "displayName",
      header: "Display Name",
      sortable: true,
      render: (_v: any, row: ColumnRow, rowIndex: number) => (
        <FormTextField
          value={row.displayName}
          onChange={(v: string) => patchRow(rowIndex, { displayName: v })}
          placeholder="e.g., Transaction ID"
          disabled={false}
          name="displayName"
          className=""
          icon=""
        />
      ),
    },
    {
      key: "description",
      header: "Description",
      sortable: true,
      render: (_v: any, row: ColumnRow, rowIndex: number) => (
        <FormTextField
          value={row.description}
          onChange={(v: string) => patchRow(rowIndex, { description: v })}
          placeholder="Field description"
          disabled={false}
          name="description"
          className=""
          icon=""
        />
      ),
    },
    {
      key: "type",
      header: "Type",
      sortable: true,
      sortAccessor: (r: ColumnRow) => r.type,
      render: (_v: any, row: ColumnRow, rowIndex: number) => (
        <FormSelectField
          value={row.type}
          onChange={(v: string) => patchRow(rowIndex, { type: v })}
          options={TYPE_OPTIONS}
          placeholder="Select type"
          disabled={false}
          name="type"
          className=""
        />
      ),
    },
    {
      key: "tags",
      header: "Tags",
      sortable: true,
      sortAccessor: (r: ColumnRow) => (r.tags?.[0] ?? "None"),
      render: (_v: any, row: ColumnRow, rowIndex: number) => (
        <FormSelectField
          value={(row.tags?.[0]) ?? "None"}
          onChange={(v: string) =>
            patchRow(rowIndex, { tags: v === "None" ? null : [v] })
          }
          options={[
            { label: "None", value: "None" },
            { label: "PII", value: "PII" },
            { label: "Sensitive", value: "Sensitive" },
          ]}
          placeholder="Select tag"
          disabled={false}
          name="tags"
          className=""
        />
      ),
    },
    {
      key: "actions",
      header: " ",
      sortable: false,
      render: (_v: any, _row: ColumnRow, rowIndex: number) => (
        <Button onClick={() => deleteRow(rowIndex)} variant="borderless" icon="">
          Remove
        </Button>
      ),
    },
  ];

  return (
    <div className="p-6">
      <h2 className="text-xl" style={{ marginBottom: 8 }}>Data Schema</h2>

      {toast?.visible && (
        <div className="mb-3 rounded border border-amber-300 bg-amber-50 p-2 text-sm">
          {toast.message}
          <Button onClick={hideToast} variant="borderless" icon="">
            Dismiss
          </Button>
        </div>
      )}

      <div style={{ display: "flex", gap: 8, alignItems: "center", marginBottom: 12 }}>
        <input
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search fields…"
          aria-label="Search fields"
          className="form-field-input"
          style={{ width: 320 }}
        />
        <Button id="add-field" onClick={addRow} variant="filled" icon="">
          + Add field
        </Button>
        <span className="text-sm" style={{ color: "#676767" }}>{rows.length} total</span>
      </div>

      <PaginationTable
        columns={columns}
        data={filtered}
        initialPageSize={10}
      />

      <div style={{ marginTop: 16, display: "flex", justifyContent: "flex-end", gap: 8 }}>
        <Button onClick={() => history.back()} variant="standard" icon="">
          Back
        </Button>
        <Button onClick={goNext} variant="filled" icon="" >
          Next
        </Button>
      </div>
    </div>
  );
}
```

**Key changes to fix your errors:**

- ✅ Use **regular imports** for components (no `import type`).
    
- ✅ Call context helpers with **positional args**:
    
    - `pushRow(selectRows, writeRows, row)`
        
    - `updateRowAt(selectRows, writeRows, index, patch)`
        
    - `removeRowAt(selectRows, writeRows, index)`
        

If anything still complains, paste the exact export lines for `FormTextField`, `FormSelectField`, and `Button` (default vs named), and I’ll adjust the imports to match.



---------------------------------



Got it—let’s keep it **very simple**:

- No CRUD in the table.
    
- No `FormSelectField`.
    
- A **search box** that filters from a small **seed list** (local `useState`).
    
- Clicking a result **adds** that field to the Data Schema (if not already added).
    
- Table is **read‑only** (plus an optional Remove button so users can undo a choice).
    
- **Validation**: if nothing is selected, the step’s validator fails and your wrapper will keep **Next** disabled.
    

Here’s a drop‑in `DataSchema.tsx` using only your shared pieces (`PaginationTable`, `Button`) and your context API (`patchDomain`, `publishStepValidator`). It also uses the **positional** selector/writer only for the _remove_ case; add is done via `patchDomain` (simpler and fewer moving parts).

```tsx
import React, { useEffect, useMemo, useState } from "react";
import { usePublication } from "../context"; // adjust relative path
import PaginationTable from "@/shared/components/PaginationTable/PaginationTable";
import type { Column } from "@/shared/type/types";
import Button from "@/shared/components/Button/Button";
import type { ColumnRow } from "../pageTypes";

/** --- Seed catalog (local) ---------------------------------- */
const SEED_CATALOG: ColumnRow[] = [
  {
    fieldName: "transaction_id",
    displayName: "Transaction ID",
    description: "Unique identifier for each transaction",
    type: "UUID",
    tags: null,
  },
  {
    fieldName: "client_name",
    displayName: "Client Name",
    description: "Full name of the client investor",
    type: "VARCHAR",
    tags: ["PII"],
  },
  {
    fieldName: "client_ssn",
    displayName: "SSN",
    description: "Client's Social Security Number",
    type: "CHAR(11)",
    tags: ["PII"],
  },
  {
    fieldName: "security_ticker",
    displayName: "Sec Ticker",
    description: "Ticker symbol of the trade",
    type: "VARCHAR",
    tags: null,
  },
  {
    fieldName: "trade_date",
    displayName: "Trade Date",
    description: "Date when the trade was executed",
    type: "DATE",
    tags: null,
  },
];

export default function DataSchema() {
  const {
    form,
    patchDomain,
    publishStepValidator,
    removeRowAt,
  } = usePublication();

  // Selected rows come from form state
  const rows = form.dataSchema?.rows ?? [];

  // local catalog + search query
  const [catalog] = useState<ColumnRow[]>(SEED_CATALOG);
  const [query, setQuery] = useState("");

  /** --- Validation: require at least one selected row -------- */
  useEffect(() => {
    publishStepValidator(() => {
      if (!rows.length) {
        return {
          ok: false,
          message: "Select at least one field for the data schema.",
          focusSelector: "#schema-search",
        };
      }
      return { ok: true };
    });
  }, [rows, publishStepValidator]);

  /** --- Derived: search results (exclude already selected) --- */
  const results = useMemo(() => {
    const q = query.trim().toLowerCase();
    const selected = new Set(rows.map(r => r.fieldName.toLowerCase()));
    return catalog.filter((c) => {
      if (selected.has(c.fieldName.toLowerCase())) return false;
      if (!q) return true;
      return (
        c.fieldName.toLowerCase().includes(q) ||
        c.displayName.toLowerCase().includes(q) ||
        (c.description ?? "").toLowerCase().includes(q) ||
        (c.tags ?? []).some(t => t.toLowerCase().includes(q))
      );
    });
  }, [catalog, rows, query]);

  /** --- Add from search selection ---------------------------- */
  const addFromCatalog = (item: ColumnRow) => {
    const exists = rows.some(
      (r) => r.fieldName.toLowerCase() === item.fieldName.toLowerCase()
    );
    if (exists) return;
    patchDomain("dataSchema", { rows: [...rows, item] });
    setQuery(""); // clear after pick
  };

  /** --- Optional: remove selected (simple undo) -------------- */
  const removeSelected = (rowIndex: number) => {
    // use positional helpers for removal
    const selectRows = (f: any): ColumnRow[] | undefined => f.dataSchema?.rows;
    const writeRows = (f: any, next: ColumnRow[]) => {
      f.dataSchema ??= { rows: [] };
      f.dataSchema.rows = next;
    };
    removeRowAt<ColumnRow>(selectRows, writeRows, rowIndex);
  };

  /** --- Table columns: read-only display --------------------- */
  const columns: Column[] = [
    {
      key: "fieldName",
      header: "Field Name",
      sortable: true,
      sortAccessor: (r) => (r as ColumnRow).fieldName?.toLowerCase() ?? "",
      render: (_v, row) => <span className="body-link-text">{(row as ColumnRow).fieldName}</span>,
    },
    {
      key: "displayName",
      header: "Display Name",
      sortable: true,
      sortAccessor: (r) => (r as ColumnRow).displayName?.toLowerCase() ?? "",
      render: (_v, row) => <span>{(row as ColumnRow).displayName}</span>,
    },
    {
      key: "description",
      header: "Description",
      sortable: true,
      sortAccessor: (r) => (r as ColumnRow).description?.toLowerCase() ?? "",
      render: (_v, row) => <span>{(row as ColumnRow).description}</span>,
    },
    {
      key: "type",
      header: "Type",
      sortable: true,
      sortAccessor: (r) => (r as ColumnRow).type?.toLowerCase() ?? "",
      render: (_v, row) => <span>{(row as ColumnRow).type}</span>,
    },
    {
      key: "tags",
      header: "Tags",
      sortable: true,
      sortAccessor: (r) => ((r as ColumnRow).tags?.[0] ?? "None").toLowerCase(),
      render: (_v, row) => <span>{(row as ColumnRow).tags?.[0] ?? "None"}</span>,
    },
    {
      key: "actions",
      header: "",
      sortable: false,
      render: (_v, _row, rowIndex) => (
        <Button onClick={() => removeSelected(rowIndex)} variant="borderless" icon="">
          Remove
        </Button>
      ),
    },
  ];

  return (
    <div className="p-6">
      <h2 className="text-xl" style={{ marginBottom: 8 }}>Data Schema</h2>

      {/* Search box (no add button). Pick from results list to add. */}
      <div style={{ display: "flex", flexDirection: "column", gap: 8, marginBottom: 12 }}>
        <input
          id="schema-search"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search fields… then click a result to add"
          aria-label="Search fields"
          className="form-field-input"
          style={{ width: 360 }}
        />

        {/* lightweight suggestion list */}
        {query.trim() && results.length > 0 && (
          <div
            className="rounded border"
            style={{ width: 360, maxHeight: 200, overflow: "auto", background: "white" }}
          >
            {results.map((r) => (
              <div
                key={r.fieldName}
                onClick={() => addFromCatalog(r)}
                className="p-2 hover:bg-gray-100 cursor-pointer"
                title={`Add "${r.fieldName}"`}
              >
                <div style={{ fontWeight: 600 }}>{r.fieldName}</div>
                <div style={{ fontSize: 12, color: "#616161" }}>{r.displayName} • {r.type} • {(r.tags?.[0] ?? "None")}</div>
              </div>
            ))}
          </div>
        )}
      </div>

      <PaginationTable columns={columns} data={rows} initialPageSize={10} />
    </div>
  );
}
```

### Notes

- **No `FormSelectField`**, no in‑table editing: the table shows what’s been chosen.
    
- Users **type**, **pick** from the suggestion list, and it’s **added** to `dataSchema.rows`.
    
- If the user picks nothing, the **validator** fails → your wrapper keeps **Next** disabled (as in your flow).
    
- Everything else uses your existing shared pieces and context. If you don’t want the **Remove** action, just delete the last “actions” column.