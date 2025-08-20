


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

-------------------


takkkkkk


Got it — your table renders nothing because of two things:

1. **Wrong imports**: `PaginationTable`, `Button`, `FormTextField`, and `FormSelectField` are **named** exports in your repo. Importing them as defaults compiles (TS may not yell, depending on config) but at runtime you render `undefined`, so nothing shows.
    
2. **Columns shape**: must match your `types.ts` (`render(value, row, rowIndex)`, `sortAccessor(row)`), and the data objects must have keys that match `Column.key`.
    

Below is a **drop‑in `DataSchema.tsx`** that uses your shared components exactly as your `PaginationTable.tsx`/`types.ts` expect. It also seeds a few rows, filters as you type, and adds a row when you press **Enter** in the search box (no “Add field” button).

```tsx
import React, { useEffect, useMemo, useState } from "react";
import { usePublication } from "../context";

// 🔴 all named imports (match your codebase)
import { PaginationTable } from "@/shared/components/PaginationTable/PaginationTable";
import { Button } from "@/shared/components/Button/Button";
import { FormTextField } from "@/shared/components/FormFields/FormTextField";
import { FormSelectField } from "@/shared/components/FormFields/FormSelectField";
import type { Column } from "@/shared/type/types";

// Your row shape (fieldName, displayName, description, type, tags?)
type ColumnRow = {
  fieldName: string;
  displayName: string;
  description: string;
  type: string;
  tags: string[] | null;
};

// --- seed rows so the table shows immediately ---
const SEED_ROWS: ColumnRow[] = [
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

const TYPE_OPTIONS = [
  { label: "UUID", value: "UUID" },
  { label: "VARCHAR", value: "VARCHAR" },
  { label: "CHAR(11)", value: "CHAR(11)" },
  { label: "DATE", value: "DATE" },
  { label: "NUMBER", value: "NUMBER" },
];

const TAG_OPTIONS = [
  { label: "None", value: "None" },
  { label: "PII", value: "PII" },
  { label: "Sensitive", value: "Sensitive" },
];

export default function DataSchema() {
  const {
    form,
    patchDomain,
    pushRow,
    updateRowAt,
    removeRowAt,
    publishStepValidator,
    toast,
    hideToast,
  } = usePublication();

  const rows: ColumnRow[] = (form.dataSchema?.rows as ColumnRow[]) ?? [];
  const [query, setQuery] = useState("");

  // seed once
  useEffect(() => {
    if (!rows.length) {
      patchDomain("dataSchema", { rows: SEED_ROWS });
    }
  }, [rows.length, patchDomain]);

  // filter
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return rows;
    return rows.filter(
      (r) =>
        r.fieldName.toLowerCase().includes(q) ||
        r.displayName.toLowerCase().includes(q) ||
        r.description.toLowerCase().includes(q) ||
        (r.tags ?? []).some((t) => t.toLowerCase().includes(q))
    );
  }, [rows, query]);

  // context helpers (positional signatures)
  const selectRows = (f: any): ColumnRow[] | undefined => f.dataSchema?.rows;
  const writeRows = (f: any, next: ColumnRow[]) => {
    f.dataSchema ??= { rows: [] };
    f.dataSchema.rows = next;
  };

  const patchRow = (index: number, patch: Partial<ColumnRow>) =>
    updateRowAt<ColumnRow>(selectRows, writeRows, index, patch);

  const addRowFromSearch = () => {
    const name = query.trim();
    if (!name) return;
    const exists = rows.some(
      (r) => r.fieldName.trim().toLowerCase() === name.toLowerCase()
    );
    if (exists) {
      setQuery("");
      return;
    }
    pushRow<ColumnRow>(selectRows, writeRows, {
      fieldName: name,
      displayName: name
        .split("_")
        .map((s) => s.charAt(0).toUpperCase() + s.slice(1))
        .join(" "),
      description: "",
      type: "VARCHAR",
      tags: null,
    });
    setQuery("");
  };

  const deleteRow = (index: number) =>
    removeRowAt<ColumnRow>(selectRows, writeRows, index);

  const onSearchKeyDown: React.KeyboardEventHandler<HTMLInputElement> = (e) => {
    if (e.key === "Enter") addRowFromSearch();
  };

  // columns must match your shared `Column` type
  const columns: Column[] = [
    {
      key: "fieldName",
      header: "Field Name",
      sortable: true,
      sortAccessor: (row) => (row as ColumnRow).fieldName.toLowerCase(),
      render: (_value, row, rowIndex) => (
        <div data-row={rowIndex} data-col="fieldName">
          <FormTextField
            value={(row as ColumnRow).fieldName}
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
      sortAccessor: (row) => (row as ColumnRow).displayName.toLowerCase(),
      render: (_value, row, rowIndex) => (
        <FormTextField
          value={(row as ColumnRow).displayName}
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
      sortAccessor: (row) => (row as ColumnRow).description.toLowerCase(),
      render: (_value, row, rowIndex) => (
        <FormTextField
          value={(row as ColumnRow).description}
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
      sortAccessor: (row) => (row as ColumnRow).type.toLowerCase(),
      render: (_value, row, rowIndex) => (
        <FormSelectField
          value={(row as ColumnRow).type}
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
      sortAccessor: (row) =>
        (((row as ColumnRow).tags ?? ["None"])[0] || "None").toLowerCase(),
      render: (_value, row, rowIndex) => (
        <FormSelectField
          value={((row as ColumnRow).tags ?? ["None"])[0]}
          onChange={(v: string) =>
            patchRow(rowIndex, { tags: v === "None" ? null : [v] })
          }
          options={TAG_OPTIONS}
          placeholder="Select tag"
          disabled={false}
          name="tags"
          className=""
        />
      ),
    },
    {
      key: "actions",
      header: "",
      sortable: false,
      render: (_value, _row, rowIndex) => (
        <Button onClick={() => deleteRow(rowIndex)} variant="borderless" icon="">
          Remove
        </Button>
      ),
    },
  ];

  return (
    <div className="p-6">
      {toast?.visible && (
        <div className="mb-3 rounded border border-amber-300 bg-amber-50 p-2 text-sm">
          {toast.message}
          <Button onClick={hideToast} variant="borderless" icon="">
            Dismiss
          </Button>
        </div>
      )}

      <h2 className="text-xl" style={{ marginBottom: 8 }}>
        Data Schema
      </h2>

      {/* Search (press Enter to add) */}
      <div style={{ display: "flex", gap: 8, alignItems: "center", marginBottom: 12 }}>
        <input
          id="search-field"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          onKeyDown={onSearchKeyDown}
          placeholder="Search fields… (press Enter to add as a new field)"
          aria-label="Search fields"
          className="form-field-input"
          style={{ width: 360 }}
        />
        <span className="text-sm" style={{ color: "#676767" }}>
          {rows.length} total
        </span>
      </div>

      <PaginationTable columns={columns} data={filtered as Record<string, any>[]} initialPageSize={10} />
    </div>
  );
}
```

### Quick checklist

- **Imports** must be:  
    `import { PaginationTable } from '.../PaginationTable'`  
    `import { Button } from '.../Button'`  
    `import { FormTextField } from '.../FormTextField'`  
    `import { FormSelectField } from '.../FormSelectField'`
    
- `Column.render` signature is `(value, row, rowIndex) => ReactNode`.
    
- `sortAccessor` signature is `(row) => string | number`.
    
- Data objects have keys: `fieldName`, `displayName`, `description`, `type`, `tags`.
    

With those fixed, the table will render the prepopulated rows, filter live, and let you add new fields by pressing Enter in the search box.

-------------------------

gems

Awesome — here’s the **complete, drop‑in update**: one React file and one SCSS module. It uses your existing `PaginationTable` + `Column` types, the green design tokens from your `scss-vars.scss`, **no add/remove/edit UI**, and validation only passes when at least one row is selected. The search bar spans the full width and there’s no add button.

---

### `src/features/publish/steps/DataSchema.module.scss`

```scss
@import "@/styles/scss-vars";

/* PAGE */
.data-schema {
  padding: $spacing-6;

  h2 {
    margin-bottom: $spacing-2;
    color: $dark-green;
    font-size: $font-h3-size;
    line-height: $font-h3-line-height;
    letter-spacing: $font-h3-letter-spacing;
  }

  /* FULL‑WIDTH SEARCH (no button) */
  .search {
    margin-bottom: $spacing-3;

    input {
      width: 100%;
      height: $sizing-ctrl-1;     /* 40px */
      padding: 0 $spacing-2;
      border: 1px solid $input-stroke;
      border-radius: 8px;
      background: $white;
      color: $black;
      font-size: $font-body-regular-size;
      line-height: $font-body-regular-line-height;

      &::placeholder { color: $hint-text; }

      &:focus {
        outline: 2px solid $primary-green-tint; /* focus halo */
        border-color: $primary-text-green;
        box-shadow: 0 0 0 1px $primary-text-green inset;
      }
    }
  }

  /* TABLE (scoped to existing PaginationTable class names) */
  .multisort-table-container {
    table { width: 100%; border-collapse: separate; border-spacing: 0; }

    thead {
      .header-cell {
        padding: $spacing-2;
        font-weight: 600;
        color: $dark-green;
        background: $primary-green-tint;
        border-bottom: 1px solid $light-divider;
      }

      .sort-arrows {
        margin-left: $spacing-1;

        .arrow-up, .arrow-down {
          display: inline-flex;
          vertical-align: middle;

          svg { width: 12px; height: 12px; }
        }

        .active {
          color: $primary-text-green;
          fill: currentColor;
        }
      }
    }

    tbody {
      td {
        padding: $spacing-2;
        border-bottom: 1px solid $light-divider;
      }
      .cell-ellipsis { color: $black; }
    }

    .pagination-controls {
      color: $black;

      select {
        height: $sizing-ctrl-1;
        border: 1px solid $input-stroke;
        border-radius: 8px;
        padding: 0 $spacing-2;

        &:focus {
          outline: 2px solid $primary-green-tint;
          border-color: $primary-text-green;
        }
      }

      .next-button,
      .pagination-center button {
        border: 1px solid $primary-text-green;
        color: $primary-text-green;
        background: $white;
        border-radius: 9999px;
        padding: 0 $spacing-2;
        height: 28px;

        &.active { background: $primary-text-green; color: $white; }
        &:disabled { opacity: .5; cursor: not-allowed; }
      }
    }
  }

  /* Checkbox selection in TD green (modern browsers) */
  input[type="checkbox"] { accent-color: $primary-text-green; }
}
```

---

### `src/features/publish/steps/DataSchema.tsx`

```tsx
import React, { useEffect, useMemo, useState } from "react";
import styles from "./DataSchema.module.scss";
import { usePublication } from "../context";
import { PaginationTable } from "@/shared/components/PaginationTable/PaginationTable";
import type { Column } from "@/shared/type/types";

/** Row model for this page only */
type ColumnRow = {
  fieldName: string;
  displayName: string;
  description: string;
  type: "UUID" | "VARCHAR" | "CHAR(11)" | "DATE" | "NUMBER" | string;
  tags: string[] | null;              // e.g., ["PII"] or null
  selected?: boolean;                 // for selection-only behavior
};

/** Seed a few rows if the form is empty */
const DEFAULT_ROWS: ColumnRow[] = [
  {
    fieldName: "transaction_id",
    displayName: "Transaction ID",
    description: "Unique identifier for each transaction.",
    type: "UUID",
    tags: null,
    selected: false,
  },
  {
    fieldName: "client_name",
    displayName: "Client Name",
    description: "Full name of the client involved.",
    type: "VARCHAR",
    tags: ["PII"],
    selected: false,
  },
  {
    fieldName: "client_ssn",
    displayName: "SSN",
    description: "Client's Social Security Number.",
    type: "CHAR(11)",
    tags: ["PII"],
    selected: false,
  },
  {
    fieldName: "security_ticker",
    displayName: "Sec Ticker",
    description: "Ticker symbol of the trade.",
    type: "VARCHAR",
    tags: null,
    selected: false,
  },
  {
    fieldName: "trade_date",
    displayName: "Trade Date",
    description: "Date when the trade was executed.",
    type: "DATE",
    tags: null,
    selected: false,
  },
];

export default function DataSchema() {
  const {
    form,
    setDomain,
    publishStepValidator,
    toast,
    hideToast,
  } = usePublication();

  // Ensure the domain exists with some seed rows
  useEffect(() => {
    const existing = form?.dataSchema?.rows as ColumnRow[] | undefined;
    if (!existing || existing.length === 0) {
      setDomain("dataSchema", { rows: DEFAULT_ROWS });
    }
  }, [form?.dataSchema?.rows, setDomain]);

  const rows: ColumnRow[] = (form?.dataSchema?.rows as ColumnRow[]) ?? [];

  // FULL‑WIDTH SEARCH
  const [query, setQuery] = useState("");
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return rows;
    return rows.filter((r) =>
      [
        r.fieldName,
        r.displayName,
        r.description,
        r.type,
        ...(r.tags ?? []),
      ]
        .filter(Boolean)
        .join(" ")
        .toLowerCase()
        .includes(q)
    );
  }, [rows, query]);

  // Toggle row selection (no add/remove/edit)
  const toggleRow = (rowIndex: number) => {
    const next = rows.map((r, i) =>
      i === rowIndex ? { ...r, selected: !r.selected } : r
    );
    setDomain("dataSchema", { rows: next });
  };

  // Validation: at least one row must be selected to proceed
  useEffect(() => {
    publishStepValidator(() => {
      const anySelected = rows.some((r) => r.selected);
      return anySelected
        ? { ok: true }
        : {
            ok: false,
            message: "Select at least one field to continue.",
            focusSelector: "#search-field",
          };
    });
    // unregister handled by provider when step unmounts
  }, [rows, publishStepValidator]);

  // Columns for the shared PaginationTable
  const columns: Column[] = [
    {
      key: "selected",
      header: "",
      sortable: false,
      render: (_v: any, row: Record<string, any>, rowIndex: number) => (
        <input
          type="checkbox"
          checked={Boolean((row as ColumnRow).selected)}
          onChange={() => toggleRow(rowIndex)}
          aria-label={`Select row ${rowIndex + 1}`}
        />
      ),
    },
    {
      key: "fieldName",
      header: "Field Name",
      sortable: true,
      sortAccessor: (r) => String((r as ColumnRow).fieldName || "").toLowerCase(),
      render: (v: any) => <span className="body-link-text">{v}</span>,
    },
    {
      key: "displayName",
      header: "Display Name",
      sortable: true,
      sortAccessor: (r) =>
        String((r as ColumnRow).displayName || "").toLowerCase(),
      render: (v: any) => <span>{v}</span>,
    },
    {
      key: "description",
      header: "Description",
      sortable: true,
      sortAccessor: (r) =>
        String((r as ColumnRow).description || "").toLowerCase(),
      render: (v: any) => <span>{v}</span>,
    },
    {
      key: "type",
      header: "Type",
      sortable: true,
      sortAccessor: (r) => String((r as ColumnRow).type || "").toLowerCase(),
      render: (v: any) => <span>{v}</span>,
    },
    {
      key: "tags",
      header: "Tags",
      sortable: true,
      sortAccessor: (r) =>
        ((r as ColumnRow).tags?.[0] ?? "None").toString().toLowerCase(),
      render: (_v: any, row: Record<string, any>) => {
        const tag = ((row as ColumnRow).tags?.[0] ?? "None") as string;
        return (
          <span>
            {tag === "PII" ? (
              // Use your green toggle visual if desired; here plain text
              <span>PII</span>
            ) : (
              "None"
            )}
          </span>
        );
      },
    },
  ];

  return (
    <div className={styles["data-schema"]}>
      {toast?.visible && (
        <div className="mb-3 rounded border border-amber-300 bg-amber-50 p-2 text-sm">
          {toast.message}
          <button onClick={hideToast} className="ml-2 underline">
            Dismiss
          </button>
        </div>
      )}

      <h2>Data Schema</h2>

      <div className={styles.search}>
        <input
          id="search-field"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search fields…"
          aria-label="Search fields"
        />
      </div>

      <PaginationTable
        columns={columns}
        data={filtered as Record<string, any>[]}
        initialPageSize={10}
      />
    </div>
  );
}
```

**What you get**

- Green theme everywhere (from your tokens).
    
- Big full‑width search (no button).
    
- Table shows read‑only values.
    
- First column is a checkbox for **select** only.
    
- **Next** validation is wired via `publishStepValidator`: it only passes when at least one row is selected.
    
- If your form is empty, it auto‑seeds with the 5 sample rows so you can search immediately.
    

If your import aliases differ, adjust the two `@/...` imports to the relative paths you use in this repo.

------------

makaaa


Perfect—let’s switch the page to **single‑select** (no checkboxes), make the **search bar full‑width with an icon + proper padding**, show a **PII tag icon** in the Tags column, and gate **Next** on a selection.

Below are drop‑in changes only in your page file + one small SCSS module. No change to the shared `PaginationTable`.

---

### 1) Styles – `DataSchema.module.scss`

```scss
/* src/features/publish/steps/DataSchema.module.scss */
@import "@/styles/scss-vars";

.data-schema {
  padding: $spacing-6;

  .search {
    position: relative;
    margin-bottom: $spacing-3;

    /* not too oval: 8px radius, generous paddings */
    input {
      width: 100%;
      height: $sizing-ctrl-1;        /* 40px */
      padding: 0 $spacing-3 0 ($spacing-6);  /* space for icon */
      border: 1px solid $input-stroke;
      border-radius: 8px;
      background: $white;
      color: $black;
      font-size: $font-body-regular-size;

      &::placeholder { color: $hint-text; }
      &:focus {
        outline: 2px solid $primary-green-tint;
        border-color: $primary-text-green;
        box-shadow: 0 0 0 1px $primary-text-green inset;
      }
    }

    .icon {
      position: absolute; left: $spacing-2; top: 50%;
      transform: translateY(-50%);
      width: 18px; height: 18px;
      pointer-events: none;
      color: $dark-green;
    }
  }

  /* selected row look: apply on every cell so whole row appears highlighted */
  .selectedCell {
    background: $primary-green-tint;     /* subtle green */
  }

  /* Tag pill shared look */
  .tagPill {
    display: inline-flex; align-items: center; gap: $spacing-1;
    height: 24px; padding: 0 $spacing-2;
    border-radius: 9999px;
    border: 1px solid $input-stroke;
    background: $white; color: $black; font-size: 12px;

    &.pii {
      border-color: $primary-text-green;
      color: $primary-text-green;
    }

    .tagIcon {
      width: 14px; height: 14px;
      display: inline-block;
    }
  }
}
```

---

### 2) Page logic – **single select + icons + validation**

```tsx
// src/features/publish/steps/DataSchema.tsx
import React, { useEffect, useMemo, useState } from "react";
import { usePublication } from "../context";
import PaginationTable from "@/shared/components/PaginationTable/PaginationTable";
import type { Column } from "@/shared/type/types";
import styles from "./DataSchema.module.scss";

// tiny inline icons (fallback if you don't have assets)
const SearchIcon = () => (
  <svg className={styles.icon} viewBox="0 0 24 24" fill="none">
    <circle cx="11" cy="11" r="7" stroke="currentColor" strokeWidth="2"/>
    <path d="M20 20L16.5 16.5" stroke="currentColor" strokeWidth="2" />
  </svg>
);
const PiiIcon = () => (
  <svg className={styles.tagIcon} viewBox="0 0 24 24" fill="none">
    <path d="M12 2l9 5v10l-9 5-9-5V7l9-5Z" stroke="currentColor" strokeWidth="2"/>
    <circle cx="12" cy="12" r="3" fill="currentColor"/>
  </svg>
);

// ---- validation helper (only selection matters) ----
type ValidationResult = { ok: true } | { ok: false; message: string; focusSelector?: string };

// (you can keep your previous validate function; just ensure it only checks selection)
function validateSelection(selectedIndex: number | null): ValidationResult {
  if (selectedIndex === null) {
    return { ok: false, message: "Select a field to continue.", focusSelector: "#search-field" };
  }
  return { ok: true };
}

export default function DataSchema() {
  const {
    form,
    publishStepValidator,
    isNextBlocked,    // stays intact
    goNext,
    toast, hideToast,
  } = usePublication();

  // source rows (prepopulated somewhere earlier or by you)
  const rows = form.schema?.rows ?? [];

  // ---- search state ----
  const [query, setQuery] = useState("");

  // ---- single selection state (row index in *rows*) ----
  const [selectedIndex, setSelectedIndex] = useState<number | null>(null);

  // register validator (fires when host provider asks)
  useEffect(() => publishStepValidator(() => validateSelection(selectedIndex)), [publishStepValidator, selectedIndex]);

  // filter and carry original index to allow selection by Enter
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return rows.map((r, i) => ({ row: r, index: i }));
    return rows
      .map((r, i) => ({ row: r, index: i }))
      .filter(({ row }) =>
        row.fieldName?.toLowerCase().includes(q) ||
        row.displayName?.toLowerCase().includes(q) ||
        row.description?.toLowerCase().includes(q) ||
        (row.tags ?? []).some((t: string) => t.toLowerCase().includes(q))
      );
  }, [rows, query]);

  // Enter to select first result
  const handleSearchKeyDown: React.KeyboardEventHandler<HTMLInputElement> = (e) => {
    if (e.key === "Enter" && filtered.length > 0) {
      setSelectedIndex(filtered[0].index);
    }
  };

  // helper: cell class when selected
  const cellClass = (rowIndex: number) =>
    rowIndex === selectedIndex ? styles.selectedCell : undefined;

  // ---- columns (read-only; click = select) ----
  const columns: Column[] = [
    {
      key: "fieldName",
      header: "Field Name",
      sortable: true,
      sortAccessor: (r) => (r.fieldName ?? "").toLowerCase(),
      render: (_v, row, rowIndex) => (
        <div className={cellClass(rowIndex)} onClick={() => setSelectedIndex(rowIndex)}>
          <a className="body-link-text">{row.fieldName}</a>
        </div>
      ),
    },
    {
      key: "displayName",
      header: "Display Name",
      sortable: true,
      render: (_v, row, rowIndex) => (
        <div className={cellClass(rowIndex)} onClick={() => setSelectedIndex(rowIndex)}>
          {row.displayName}
        </div>
      ),
    },
    {
      key: "description",
      header: "Description",
      sortable: true,
      render: (_v, row, rowIndex) => (
        <div className={cellClass(rowIndex)} onClick={() => setSelectedIndex(rowIndex)}>
          {row.description}
        </div>
      ),
    },
    {
      key: "type",
      header: "Type",
      sortable: true,
      sortAccessor: (r) => r.type ?? "",
      render: (_v, row, rowIndex) => (
        <div className={cellClass(rowIndex)} onClick={() => setSelectedIndex(rowIndex)}>
          {row.type}
        </div>
      ),
    },
    {
      key: "tags",
      header: "Tags",
      sortable: true,
      sortAccessor: (r) => (r.tags?.[0] ?? "None"),
      render: (_v, row, rowIndex) => {
        const t = row.tags?.[0] ?? "None";
        const isPII = t.toLowerCase() === "pii";
        return (
          <div className={cellClass(rowIndex)} onClick={() => setSelectedIndex(rowIndex)}>
            <span className={` ${styles.tagPill} ${isPII ? styles.pii : ""}`}>
              {isPII && <PiiIcon />}
              {t}
            </span>
          </div>
        );
      },
    },
  ];

  return (
    <div className={styles["data-schema"]}>
      {toast?.visible && (
        <div className="mb-3 rounded border border-amber-300 bg-amber-50 p-2 text-sm">
          {toast.message}
          <button onClick={hideToast} className="ml-2 underline">Dismiss</button>
        </div>
      )}

      <div className={styles.search}>
        <SearchIcon />
        <input
          id="search-field"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          onKeyDown={handleSearchKeyDown}
          placeholder="Search fields…"
          aria-label="Search fields"
        />
      </div>

      <PaginationTable
        columns={columns}
        data={filtered.map(({ row }) => row) as Record<string, any>[]}
        initialPageSize={10}
      />
    </div>
  );
}
```

**What changed**

- **No checkboxes**; clicking any cell selects that row. The selected row is highlighted via `.selectedCell`.
    
- **Validation**: blocks “Next” until one row is selected (`selectedIndex !== null`). This plugs into your existing `publishStepValidator`.
    
- **Search bar**: full width, left search icon, sensible padding, 8px radius.
    
- **Tags**: pill with a **PII icon** when the first tag is `PII`. You can swap the inline SVGs for your own assets later.
    

If you want the Enter key to **confirm selection and auto‑advance** (call `goNext()` when valid), add:

```ts
if (e.key === "Enter" && filtered.length > 0) {
  setSelectedIndex(filtered[0].index);
  const res = validateSelection(filtered[0].index);
  if (res.ok && !isNextBlocked) goNext();
}
```

This keeps everything within your existing components and style system (green tokens).

-----------------

god


Got it 👍 Thanks for flagging. Two issues likely happened:

1. **Data not showing** → because I wrapped your `rows` into `{row, index}` objects for filtering and then passed the wrong shape into `PaginationTable`.  
    → Fix: always pass the original array of objects (`rows`) into the table, not the wrappers.
    
2. **Validation not firing** → your host app expects `publishStepValidator` to register a validator. In my last code I built a local `validateSelection` but didn’t hook it properly into `form` or `goNext`.
    

Let me show you a corrected version that **(a)** shows your data again, and **(b)** runs validation to block “Next” until one row is selected.

---

### ✅ Corrected `DataSchema.tsx`

```tsx
import React, { useEffect, useMemo, useState } from "react";
import { usePublication } from "../context";
import PaginationTable from "@/shared/components/PaginationTable/PaginationTable";
import type { Column } from "@/shared/type/types";
import styles from "./DataSchema.module.scss";

const SearchIcon = () => (
  <svg className={styles.icon} viewBox="0 0 24 24" fill="none">
    <circle cx="11" cy="11" r="7" stroke="currentColor" strokeWidth="2"/>
    <path d="M20 20L16.5 16.5" stroke="currentColor" strokeWidth="2" />
  </svg>
);
const PiiIcon = () => (
  <svg className={styles.tagIcon} viewBox="0 0 24 24" fill="none">
    <path d="M12 2l9 5v10l-9 5-9-5V7l9-5Z" stroke="currentColor" strokeWidth="2"/>
    <circle cx="12" cy="12" r="3" fill="currentColor"/>
  </svg>
);

type ValidationResult = { ok: true } | { ok: false; message: string; focusSelector?: string };

export default function DataSchema() {
  const {
    form,
    publishStepValidator,
    toast,
    hideToast,
  } = usePublication();

  // --- rows come from form or mock data ---
  const rows = form.schema?.rows ?? [
    {
      fieldName: "transaction_id",
      displayName: "Transaction ID",
      description: "Unique identifier for each transaction",
      type: "UUID",
      tags: [],
    },
    {
      fieldName: "client_name",
      displayName: "Client Name",
      description: "Full name of the client involved",
      type: "VARCHAR",
      tags: ["PII"],
    },
    {
      fieldName: "trade_date",
      displayName: "Trade Date",
      description: "Date when the trade was executed",
      type: "DATE",
      tags: [],
    },
  ];

  // --- search ---
  const [query, setQuery] = useState("");

  // --- single selection ---
  const [selectedIndex, setSelectedIndex] = useState<number | null>(null);

  // --- filter logic ---
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return rows;
    return rows.filter(r =>
      r.fieldName.toLowerCase().includes(q) ||
      r.displayName.toLowerCase().includes(q) ||
      r.description.toLowerCase().includes(q) ||
      (r.tags ?? []).some((t: string) => t.toLowerCase().includes(q))
    );
  }, [rows, query]);

  // --- validation: register with publication provider ---
  useEffect(() => {
    publishStepValidator((): ValidationResult => {
      if (selectedIndex === null) {
        return { ok: false, message: "Please select a field before continuing.", focusSelector: "#search-field" };
      }
      return { ok: true };
    });
  }, [publishStepValidator, selectedIndex]);

  // --- columns ---
  const columns: Column[] = [
    {
      key: "fieldName",
      header: "Field Name",
      sortable: true,
      render: (_v, row, i) => (
        <div
          className={i === selectedIndex ? styles.selectedCell : ""}
          onClick={() => setSelectedIndex(i)}
        >
          <a className="body-link-text">{row.fieldName}</a>
        </div>
      ),
    },
    {
      key: "displayName",
      header: "Display Name",
      render: (_v, row, i) => (
        <div
          className={i === selectedIndex ? styles.selectedCell : ""}
          onClick={() => setSelectedIndex(i)}
        >
          {row.displayName}
        </div>
      ),
    },
    {
      key: "description",
      header: "Description",
      render: (_v, row, i) => (
        <div
          className={i === selectedIndex ? styles.selectedCell : ""}
          onClick={() => setSelectedIndex(i)}
        >
          {row.description}
        </div>
      ),
    },
    {
      key: "type",
      header: "Type",
      render: (_v, row, i) => (
        <div
          className={i === selectedIndex ? styles.selectedCell : ""}
          onClick={() => setSelectedIndex(i)}
        >
          {row.type}
        </div>
      ),
    },
    {
      key: "tags",
      header: "Tags",
      render: (_v, row, i) => {
        const t = row.tags?.[0] ?? "None";
        const isPII = t.toLowerCase() === "pii";
        return (
          <div
            className={i === selectedIndex ? styles.selectedCell : ""}
            onClick={() => setSelectedIndex(i)}
          >
            <span className={`${styles.tagPill} ${isPII ? styles.pii : ""}`}>
              {isPII && <PiiIcon />}
              {t}
            </span>
          </div>
        );
      },
    },
  ];

  return (
    <div className={styles["data-schema"]}>
      {toast?.visible && (
        <div className="mb-3 rounded border border-amber-300 bg-amber-50 p-2 text-sm">
          {toast.message}
          <button onClick={hideToast} className="ml-2 underline">Dismiss</button>
        </div>
      )}

      <div className={styles.search}>
        <SearchIcon />
        <input
          id="search-field"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search fields…"
        />
      </div>

      <PaginationTable
        columns={columns}
        data={filtered}
        initialPageSize={10}
      />
    </div>
  );
}
```

---

### 🔑 Fixes

- **Data**: we pass `filtered` (plain row objects) to `PaginationTable`. Now table rows render.
    
- **Validation**: inside `useEffect`, `publishStepValidator` is given a closure that checks `selectedIndex`. If none selected → shows message and blocks **Next**.
    
- **Mock data**: if `form.schema?.rows` is empty, three rows are pre-populated so you can see something.
    

---

👉 Do you want me to also hook the **toast** (error popup) so that when validation fails on “Next”, it shows `toast.message` automatically? Or should it only block silently like in your screenshot?


-----------------------------


rows:


Perfect 👍 I’ll give you **15+ mock rows** so that pagination kicks in and you can test scrolling/search properly.  
Here’s just the `rows` array you can drop into the component:

```ts
const rows = form.schema?.rows ?? [
  {
    fieldName: "transaction_id",
    displayName: "Transaction ID",
    description: "Unique identifier for each transaction",
    type: "UUID",
    tags: [],
  },
  {
    fieldName: "client_name",
    displayName: "Client Name",
    description: "Full name of the client",
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
    tags: [],
  },
  {
    fieldName: "trade_date",
    displayName: "Trade Date",
    description: "Date when the trade was executed",
    type: "DATE",
    tags: [],
  },
  {
    fieldName: "trade_amount",
    displayName: "Trade Amount",
    description: "Value of the trade",
    type: "DECIMAL(12,2)",
    tags: [],
  },
  {
    fieldName: "currency",
    displayName: "Currency",
    description: "Currency of the transaction",
    type: "CHAR(3)",
    tags: [],
  },
  {
    fieldName: "broker_id",
    displayName: "Broker ID",
    description: "Identifier of executing broker",
    type: "VARCHAR",
    tags: [],
  },
  {
    fieldName: "broker_name",
    displayName: "Broker Name",
    description: "Full name of broker",
    type: "VARCHAR",
    tags: [],
  },
  {
    fieldName: "account_id",
    displayName: "Account ID",
    description: "Internal account identifier",
    type: "UUID",
    tags: [],
  },
  {
    fieldName: "account_type",
    displayName: "Account Type",
    description: "Retail, Institutional, or Other",
    type: "VARCHAR",
    tags: [],
  },
  {
    fieldName: "country",
    displayName: "Country",
    description: "Country of client residence",
    type: "VARCHAR",
    tags: [],
  },
  {
    fieldName: "email",
    displayName: "Email",
    description: "Contact email of client",
    type: "VARCHAR",
    tags: ["PII"],
  },
  {
    fieldName: "phone_number",
    displayName: "Phone Number",
    description: "Client contact phone",
    type: "VARCHAR",
    tags: ["PII"],
  },
  {
    fieldName: "created_at",
    displayName: "Created At",
    description: "Record creation timestamp",
    type: "TIMESTAMP",
    tags: [],
  },
  {
    fieldName: "updated_at",
    displayName: "Updated At",
    description: "Record update timestamp",
    type: "TIMESTAMP",
    tags: [],
  },
];
```

---

This will give you **16 fields total** so your table will paginate past page 1.  
➡️ You’ll get a good spread of `UUID`, `VARCHAR`, `CHAR`, `DECIMAL`, `DATE`, and `TIMESTAMP` plus a few **PII tags** so you can see the icon render.

---

Do you also want me to **group PII fields visually** (like coloring the whole row or column differently), or keep it just as the pill + icon?