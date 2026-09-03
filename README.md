# Dukan — دفتر المحل

Offline store manager for iPhone. Arabic / English / French, prices in DZD.
Tracks stock, sales, profit, product photos and barcodes.

No server, no login, no account. Everything lives on the phone.

## Files
| file | what it is |
|---|---|
| `index.html` | the entire app |
| `manifest.json` | makes it installable |
| `sw.js` | offline cache |
| `icon-180/192/512.png` | home-screen icons |
| `products-example.json` | the bulk-import format |

## Install on iPhone (from Linux, no Mac)

Safari needs HTTPS for "Add to Home Screen", so host the folder once:

- **GitHub Pages** — new repo → upload all files to root → Settings → Pages → source `main` / root → open the URL in **Safari** → Share → *Add to Home Screen*
- **Netlify Drop** — drag the folder onto app.netlify.com/drop, get a URL instantly

After installing it runs fully offline.

Local testing on Linux: `python3 -m http.server 8080`, then http://localhost:8080

## Language
Settings → اللغة / Language / Langue. The layout flips to RTL for Arabic and back to
LTR for English and French. The currency label follows too (د.ج / DZD / DA). Your
choice is saved with the data, so a restored backup comes back in the same language.

## Where the data lives

Two stores, for a reason:

- **Product and sales data** → `localStorage`, one JSON object under the key `dukkan.v1`
- **Photos** → IndexedDB, keyed by product id

Photos are kept separate because `localStorage` caps out around 5 MB on iOS — a few
dozen images would fill it and silently break saving. Each photo is downscaled to
400 px and re-encoded as JPEG (~20–30 KB), so hundreds of products stay comfortable.

### Backups
- **Full backup** — one JSON file containing everything, photos included as base64
- **Light backup** — same file without photos, much smaller, good for a weekly copy

Both restore through the same button.

## Editing products in a JSON file

Settings → *Export product list* gives you a clean array with no internal ids:

```json
[
  { "name": "ماء 1.5 ل", "barcode": "6130001234567", "cost": 25, "price": 40, "qty": 48, "min": 12 }
]
```

Edit it in any text editor, add as many rows as you like, then *Import and merge*.
Matching is by `barcode` first, then by exact `name`:

- match found → the row updates that product (price, qty, everything)
- no match → a new product is created

Nothing is deleted by an import, and sales history is never touched. Only `name` is
required; missing numbers default to 0, and `min` defaults to 3.

See `products-example.json` for a working file.

## Photos
Tap a product in Stock → the photo box → camera or photo library. Thumbnails then
show in both the stock list and the sell list, which makes finding an item faster
than reading names.

## Barcode scanning
- Chrome / Android: the built-in `BarcodeDetector`.
- iOS Safari has no such API, so the app loads ZXing from a CDN the first time you
  scan. **Do one scan while online** — the service worker caches the library and
  scanning works offline afterwards.
- Camera needs HTTPS and permission. If blocked: iPhone Settings → Safari → Camera → Allow.

## Back up regularly
iOS can clear website storage for apps left unused for a few weeks. Installed
home-screen apps are usually spared, but don't bet your inventory on it. Take a full
backup weekly and keep the file in Files, Drive or email.

## Data model
```
db      = { store:{name,lang}, products:[{id,name,barcode,cost,price,qty,min}],
            sales:[{id,ts,items:[{pid,name,qty,price,cost}],total,profit}], seq }
photos  = { <product id>: "data:image/jpeg;base64,…" }
```
Profit is recorded per sale from the cost price at the moment of the sale, so
changing a cost later never rewrites past reports.
