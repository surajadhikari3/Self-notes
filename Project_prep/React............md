


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