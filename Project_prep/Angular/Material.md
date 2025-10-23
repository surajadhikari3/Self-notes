


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