


With Angular Material (MDC, v15+), the snack bar background is controlled on the **container** element  
`.mat-mdc-snack-bar-container`, not on a free-standing `.success-snackbar` class. Your `panelClass` is added **to the container**, so target it like this (in **`styles.scss`**, not component SCSS):

```scss
/* Works on Angular Material MDC (v15+) */
.mat-mdc-snack-bar-container.success-snackbar {
  /* Preferred: use MDC vars so it wins over default styles */
  --mdc-snackbar-container-color: #4caf50; /* green */
  --mdc-snackbar-supporting-text-color: #fff;
  --mdc-snackbar-button-color: #fff;

  /* Optional fallback if you’re on an odd mix of versions */
  background-color: #4caf50 !important;
  color: #fff !important;
}

.mat-mdc-snack-bar-container.error-snackbar {
  --mdc-snackbar-container-color: #f44336; /* red */
  --mdc-snackbar-supporting-text-color: #fff;
  --mdc-snackbar-button-color: #fff;

  background-color: #f44336 !important;
  color: #fff !important;
}
```

And keep the open calls like:

```ts
this.snackBar.open(`${row.tableName} tagged successfully!`, 'Close', {
  duration: 3000,
  panelClass: ['success-snackbar'],
});

this.snackBar.open('Failed to create tag. Please try again.', 'Close', {
  duration: 3000,
  panelClass: ['error-snackbar'],
});
```

**Double-check these common gotchas:**

1. `styles.scss` is listed under `angular.json > projects.app.architect.build.options.styles`.
    
2. You’re not defining these rules inside a component SCSS (encapsulation would block them).
    
3. If you’re on **legacy** (pre-MDC) Angular Material, the selector changes to  
    `.mat-snack-bar-container.success-snackbar { … }`.
    

If it still doesn’t change, open DevTools → inspect the snack bar → confirm the container has classes:  
`mat-mdc-snack-bar-container success-snackbar`. If you see `mat-snack-bar-container` instead, use the legacy selector.


-----------------------------------

No stress—let’s brute-force this and also debug what’s going on. Do these 3 things exactly:

# 1) Use selectors that match **both** legacy and MDC + target inner text

Put this **in `src/styles.scss`** (global). It covers:

- MDC (`.mat-mdc-snack-bar-container`)
    
- Legacy (`.mat-snack-bar-container`)
    
- Forces label/action text color too.
    

```scss
/* SUCCESS */
.cdk-overlay-container .mat-mdc-snack-bar-container.success-snackbar,
.cdk-overlay-container .mat-snack-bar-container.success-snackbar {
  background: #4caf50 !important;
  color: #fff !important;
}

/* make the message + action text white */
.cdk-overlay-container .mat-mdc-snack-bar-container.success-snackbar .mdc-snackbar__label,
.cdk-overlay-container .mat-mdc-snack-bar-container.success-snackbar .mat-mdc-snack-bar-action,
.cdk-overlay-container .mat-snack-bar-container.success-snackbar .mat-simple-snackbar,
.cdk-overlay-container .mat-snack-bar-container.success-snackbar .mat-button-wrapper {
  color: #fff !important;
}

/* ERROR */
.cdk-overlay-container .mat-mdc-snack-bar-container.error-snackbar,
.cdk-overlay-container .mat-snack-bar-container.error-snackbar {
  background: #f44336 !important;
  color: #fff !important;
}

.cdk-overlay-container .mat-mdc-snack-bar-container.error-snackbar .mdc-snackbar__label,
.cdk-overlay-container .mat-mdc-snack-bar-container.error-snackbar .mat-mdc-snack-bar-action,
.cdk-overlay-container .mat-snack-bar-container.error-snackbar .mat-simple-snackbar,
.cdk-overlay-container .mat-snack-bar-container.error-snackbar .mat-button-wrapper {
  color: #fff !important;
}
```

# 2) Pass the `panelClass` (array is safest)

```ts
this.snackBar.open('Tagged successfully!', 'Close', {
  duration: 3000,
  panelClass: ['success-snackbar'],
});

this.snackBar.open('Failed to create tag.', 'Close', {
  duration: 3000,
  panelClass: ['error-snackbar'],
});
```

# 3) Quick debug checklist (1 minute)

- **Inspect the DOM (F12)** while the toast is visible.
    
    - The container should be **inside `.cdk-overlay-container`** and have either:
        
        - `mat-mdc-snack-bar-container success-snackbar`, or
            
        - `mat-snack-bar-container success-snackbar`
            
    - If you only see `mat-snack-bar-container`, you’re on **legacy** (our CSS above already covers it).
        
- **Ensure global styles are loaded**: in `angular.json` → `projects → <app> → architect → build → options → styles` you should see `src/styles.scss`.
    
- You put the rules in **styles.scss** (not a component scss).
    
- You **fully restarted** `ng serve` after changing `angular.json` or adding the file the first time.
    

---

If this works (it should), you can later switch to the cleaner MDC-token version:

```scss
.mat-mdc-snack-bar-container.success-snackbar {
  --mdc-snackbar-container-color: #4caf50;
  --mdc-snackbar-supporting-text-color: #fff;
  --mdc-snackbar-button-color: #fff;
}
```

…but the “belt-and-suspenders” CSS above will override **any** theme/order issues and confirm the problem was just selector/specificity/version mismatch.

Tell me what you see in DevTools for the snack bar element (its class names), and I’ll tailor the final minimal CSS for your exact Material version.