


Here you go—drop these into your icons file and import as named exports. They match the format in your screenshot (`export const … = (props: SVGProps<SVGSVGElement>) => (...)`) and use `currentColor`, so you can color them via CSS.

```tsx
import * as React from "react";
import { SVGProps } from "react";

/** Single user (outline) */
export const SingleUserIcon = (props: SVGProps<SVGSVGElement>) => (
  <svg
    width="19"
    height="19"
    viewBox="0 0 24 24"
    fill="none"
    xmlns="http://www.w3.org/2000/svg"
    {...props}
  >
    <g stroke="currentColor" strokeWidth={1.8} strokeLinecap="round" strokeLinejoin="round">
      <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2" />
      <circle cx="12" cy="7" r="4" />
    </g>
  </svg>
);

/** Double users / group (outline) */
export const DoubleUserIcon = (props: SVGProps<SVGSVGElement>) => (
  <svg
    width="19"
    height="19"
    viewBox="0 0 24 24"
    fill="none"
    xmlns="http://www.w3.org/2000/svg"
    {...props}
  >
    <g stroke="currentColor" strokeWidth={1.8} strokeLinecap="round" strokeLinejoin="round">
      {/* front/left user */}
      <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2" />
      <circle cx="9" cy="7" r="4" />
      {/* back/right user */}
      <path d="M23 21v-2a4 4 0 0 0-3-3.87" />
      <path d="M16 3.13a4 4 0 1 1 0 7.75" />
    </g>
  </svg>
);
```

Usage:

```tsx
// Example
// <SingleUserIcon className="text-green-700" />
// <DoubleUserIcon style={{ color: '#0f8f4e' }} />
```

If you prefer filled silhouettes instead of outlines, I can provide a `fill="currentColor"` version too.