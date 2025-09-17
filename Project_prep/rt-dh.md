
Perfect—keep the layout fixed at **719px** and make every card fill its cell, with the **body stretching to the remaining height** and turning **scrollable** when content grows. The critical bits are: use a grid for the 3 columns, and set `min-height: 0` on every ancestor between the fixed-height layout and the scrollable body.

### 1) `DashboardPage.module.scss` (replace/adjust)

```scss
/* fixed-height, three columns */
.layout {
  height: 719px;                     /* fixed */
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: 1fr;               /* every row's cards are same height */
  gap: 20px;
  padding: 10px;

  /* allow children to shrink within the fixed height */
  min-height: 0;
}

/* each card = header + scrollable body */
.dashboardCard {
  display: flex;
  flex-direction: column;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-sizing: border-box;

  /* MUST be here so the body can shrink/scroll */
  min-height: 0;
  overflow: hidden;                  /* keeps rounded corners when body scrolls */
}

.cardHeader {
  flex: 0 0 auto;
  padding: 12px 14px;
  border-bottom: 1px solid #eee;
  display: flex; align-items: center; justify-content: space-between;
}

/* this area must fill the card's remaining height */
.cardBody {
  flex: 1 1 auto;                    /* take remaining vertical space */
  min-height: 0;                     /* CRUCIAL: allows shrinking in 719px */
  overflow: auto;                    /* scroll if content exceeds height */
  padding: 10px;
  display: flex; flex-direction: column; gap: 16px;
}

/* optional inner wrapper when you map items or mount tables */
.cardContent {
  flex: 1 1 auto;
  min-height: 0;
  overflow: auto;                    /* inner scroll if you want; keep or drop */
  display: flex; flex-direction: column; gap: 16px;
}

.statusMargin { margin-bottom: 16px; }
```

### 2) `DashboardCard.tsx` (simple header/body split)

```tsx
import React from "react";
import styles from "./DashboardPage.module.scss";

type Action = { actionLabel: string; onActionClick: () => void; icon?: React.ReactNode };

export const DashboardCard: React.FC<{
  title: string;
  actions?: Action[];
  rightSideAction?: React.ReactNode;
  children?: React.ReactNode;
}> = ({ title, actions = [], rightSideAction, children }) => (
  <section className={styles.dashboardCard} aria-label={title}>
    <header className={styles.cardHeader}>
      <h3 style={{ margin: 0, fontSize: 16, fontWeight: 600 }}>{title}</h3>
      <div style={{ display: "flex", gap: 10, alignItems: "center" }}>
        {actions.map((a, i) => (
          <button key={i} onClick={a.onActionClick} style={{ padding: "6px 10px", borderRadius: 6 }}>
            {a.icon} {a.actionLabel}
          </button>
        ))}
        {rightSideAction}
      </div>
    </header>

    {/* Fills remaining height and scrolls when needed */}
    <div className={styles.cardBody}>{children}</div>
  </section>
);
```

### 3) `DashboardPage.tsx` (ensure inner wrappers can shrink)

```tsx
export const DashboardPage: React.FC = () => (
  <div className={styles.layout}>
    {/* Ingestion */}
    <DashboardCard
      title="Ingestion"
      actions={[
        { actionLabel: "Ingest Data", onActionClick: handleIngestClick },
        { actionLabel: "View All", onActionClick: handleIngestClick },
      ]}
    >
      <div className={styles.cardContent}>
        {ingestionCardData.map((card, i) => (
          <div key={i} className={styles.statusMargin}>
            <StatusCard
              title={card.title}
              content={<></>}
              subtitle={card.subtitle}
              rightSideAction={<span>{card.status}</span>}
            />
          </div>
        ))}
      </div>
    </DashboardCard>

    {/* Datasets */}
    <DashboardCard
      title="Datasets"
      actions={[
        { actionLabel: "Register data", onActionClick: handleRegisterClick },
        { actionLabel: "View All", onActionClick: handleViewAllDatasets },
      ]}
    >
      {/* Wrapper must be allowed to shrink */}
      <div className={styles.cardContent}>
        <div style={{ flex: 1, minHeight: 0, display: "flex", flexDirection: "column" }}>
          <PaginationTable
            columns={columns}
            data={datasetTableData}
            initialPageSize={datasetTableData.length}
            isLoading={false}
            emptyMessage="No datasets available."
            enablePagination={false}
            enableRowSeparator={true}
            /* if your table component accepts style: */
            style={{ flex: 1, minHeight: 0 }}
          />
        </div>
      </div>
    </DashboardCard>

    {/* Status */}
    <DashboardCard title="Status" rightSideAction={<img src={checkStatus} alt="All Operational" />}>
      <div className={styles.cardContent}>
        <div style={{ display: "flex", flexWrap: "wrap", gap: 20, minHeight: 0 }}>
          {statusCardData.map((c, i) => (
            <div key={i} style={{ flex: "1 1 calc(50% - 20px)", minWidth: 260, minHeight: 0 }}>
              <StatusCard title={c.title} content={<></>} subtitle={c.subtitle} rightSideAction={<img src={checkMark} alt="" />} />
            </div>
          ))}
        </div>
      </div>
    </DashboardCard>

    {/* Incident Log */}
    <DashboardCard title="Incident Log" actions={[{ actionLabel: "View all", onActionClick: () => {} }]}>
      <div className={styles.cardContent}>
        {incidentLogCardData.map((c, i) => (
          <div key={i} className={styles.statusMargin}>
            <StatusCard
              title={c.title}
              content={c.content}
              subtitle={
                <>
                  {c.icon && <img src={c.icon} alt="" />} <span>{c.subtitle}</span>
                </>
              }
              rightSideAction={<img src={expandView} alt="" />}
            />
          </div>
        ))}
      </div>
    </DashboardCard>
  </div>
);
```

#### Why this fixes your screenshot issue

- The outer `.layout` is **fixed at 719px**; using CSS **grid** makes each column/row cell the same height.
    
- Each `.dashboardCard` is a **flex column** with `min-height: 0`, so its **`.cardBody` can shrink** to fit within 719px.
    
- `.cardBody` uses `flex: 1` + `overflow: auto`, so it **fills the card** and **scrolls only when needed**.
    
- Any inner wrapper (like tables/lists) gets `flex: 1; min-height: 0;` to avoid “pushing” the card taller than the layout.
    

Drop these changes in and your cards will stretch to the full fixed layout height and become neatly scrollable when content grows—without breaking your 3-column fixed dashboard.