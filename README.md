# Matlistor

**Automatic cleaning, labels and portioning of special diets — for the Kriminalvården kitchen.**

A single static HTML file. All processing happens in the browser on the local computer.
No backend, no database, no server-side component.

`100% client-side` · `zero network (offline build)` · `zero PII` · `single-file HTML`

---

## What it does

From the daily meal-order list (Matbeställningslista, PDF) it produces:

- **Rensad lista** — the special-diet list cleaned by fixed rules
- **Etiketter** — one label per person (ward, diet, number)
- **Summa per kost** — totals per diet type
- **Fläsk-räkning** — pork count per ward
- **Portionering** — how much to cook per ward

It reads **no personal names** — only registration number, ward and diet.

---

## Security & privacy

- Everything runs in the browser — **data never leaves the computer**.
- **No names** are read or stored.
- **No database, no cookies, no logging, no outbound traffic.**
- Processing lives only in volatile memory; closing the tab clears it.

```
PDF list (local)
   -> Browser memory: pdf.js reads only X-coordinates + regex
   -> Name column discarded
   -> jsPDF builds the output PDFs
   -> Close the tab = memory cleared
```

---

## Deployment

| Option | Notes |
|---|---|
| **GitHub Pages** (current) | Free, but external. |
| **In-house** | Put the single HTML file on your own web server, or open it locally from disk. No database, nothing to operate. |
| **Offline build** | All libraries embedded in the HTML (`index.html` offline version): **zero external requests**, works on a fully isolated / firewalled network. Recommended for internal hosting. |

### Libraries

| Library | Version | Purpose | Standard build loads from |
|---|---|---|---|
| pdf.js | 3.11.174 | reads the PDF | cdnjs (or inlined in offline build) |
| jsPDF | 2.5.1 | builds the output PDFs | cdnjs (or inlined) |
| jspdf-autotable | 3.8.2 | tables in the output | cdnjs (or inlined) |
| pdf.worker | 3.11.174 | pdf.js worker thread | cdnjs (or inlined as a Blob) |

> **CSP note (internal server):** the offline build needs `worker-src blob:`
> (the pdf.js worker runs from a Blob URL) and inline scripts (`script-src` for
> the embedded libraries), or serve the file without a restrictive CSP.
> Opened directly from disk (`file://`) no CSP applies.

---

## How the data is read

| Input | Logic | Output |
|---|---|---|
| PDF text items with X-coordinates | split columns at the Specialkost boundary | left = reg.no + ward; right = diet; **name discarded** |
| ward token split by pdf.js (spaces) | regex tolerant of spaces, then strip spaces | normalised ward, e.g. `12C2/12C2-11:1` |
| raw diet text | `cleanSpec()` removal / keep / flag rules | cleaned diet, or row flagged for check |
| diet present but no ward (after wards start) | safety check against the leading block | row flagged `AVD SAKNAS` (red) |
| portion adjustment string | global regex: `=` fixed, `+`/`-` delta | per-ward portion count (fixed gets no extra) |

---

## Key code

### Position-based reading
The list is read by X-coordinate, so the name column (between number+ward and the
diet column) is never touched. pdf.js can split the ward into pieces with spaces,
so the match tolerates spaces and strips them afterwards.

```js
const leftText = sorted.filter(o => o.x < boundary)
                       .map(o => o.str).join(" ");
const im = leftText.match(/(\d{4}\/\d+)/);               // reg.no
const am = leftText.match(                               // ward (tolerates spaces)
  /([A-Za-zÅÄÖåäö0-9]+\s*\/\s*[A-Za-zÅÄÖåäö0-9\s\-]*?:\s*\d+)/);
cur = {
  insk: im ? im[1] : "",
  avd:  am ? am[1].replace(/\s+/g, "") : "",             // remove spaces
  spec: rightText                                        // the NAME is never read
};
```

### Automatic label sizing
Ward names with letters (72RO, 12B2, 12C2 …) start two points smaller and shrink
until they fit; pure numeric wards keep full size.

```js
const special = /[A-Za-zÅÄÖåäö]/.test(it.ward || "");
let ws = special ? WARD - 2 : WARD;
while (ws > 8 && doc.getStringUnitWidth(it.ward || "") * ws * PT > WARD_MAXW) {
  ws -= 1;                       // shrink until the name fits
}
```

### Safety check: missing ward
A person with a diet but no ward — appearing after wards have started — is flagged
red. The leading block without a ward (at the very top) is left as-is.

```js
const firstAvd = cleaned.findIndex(p => p.avd);
if (firstAvd >= 0) {
  cleaned.forEach((p, i) => {
    if (i > firstAvd && !p.avd && p.text !== "" && p.flag !== "review")
      p.flag = "noavd";          // diet but no ward -> flag red
  });
}
```

### Portioning adjustments
Finds **all** adjustments on a line regardless of separator (space, comma, newline).
`=` sets a fixed count (no extra added); `+`/`-` change the count and extra applies.

```js
const RX = /([Hh]?\s*\d+[A-Za-zÅÄÖåäö]*\d*)\s*(=|\+|-)\s*(\d+)/g;
let m;
while ((m = RX.exec(line))) {       // all adjustments, not just the first
  const k = normW(m[1]), op = m[2], n = +m[3];
  if (op === "=") map[k] = { type: "fixed", val: n };          // fixed: no extra
  else            map[k] = { type: "delta", val: (op === "-" ? -1 : 1) * n };
}
```

---

## Cleaning rules

- Always removed: `Ej Fläsk`, `Ramadan`, `Ungdom`.
- Removed only when alone: `Nötter`, `Mandel`, `Laktos`, `Ej Blodmat`.
- Allergens and short "Annat" foods get a `-` in front.
- Instructions (e.g. Diabetes, heart issues) stay as text.
- `Nötter`/`Mandel` mixed with other content → row flagged `KONTROLLERA`, untouched.
- **Rows are never deleted** — when in doubt, the row is kept.

---

## Version & contact

Current version: **v1.28**. Shown at the bottom of the tool and in the portioning PDF.

Built and maintained by **Simone Frosali**, Kitchen, Kriminalvården Västervik.
