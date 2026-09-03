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

## Selling by weight

Each product has a unit: **piece, kilo, gram or litre**. Set it in the product form.

- **Piece** products behave as before — tap to add, then + / − in the cart.
- **Weighed** products (kg, g, L) open a keypad instead. Type `0.75`, or tap one of
  the quick buttons (0.25 / 0.5 / 1 / 2). Stock is kept to 3 decimals, so 12.5 kg
  minus 750 g leaves 11.75 kg.

Prices for weighed goods are **per unit** — the form and the lists show `/kg` so you
don't confuse a price per kilo with a price per piece.

## Deals and discounts

Three separate ways to cut a price, because they answer different needs:

**1. A standing deal on a product** — set a *Deal price* in the product form. The
product shows a red DEAL badge with the old price struck through, and it's charged
at the deal price automatically until you clear the field. Use this for a promo that
lasts a few days.

**2. A discount on one line in the cart** — tap the line, then a −5 / −10 / −15 / −20 %
button, or type any unit price by hand. *Back to normal price* undoes it. Use this
when you knock something off for one customer.

**3. A discount on the whole ticket** — in the cart, either type an amount in DZD or
tap a percentage. Use this for rounding down the total, which is the common case.

All three come straight off profit — the reports never flatter you. The Reports tab
shows total *discounts given* for the period next to your margin, so you can see what
your generosity actually costs over a month.

Each sale stores both the price charged and the original price, so an old receipt
still shows what the discount was even after you change the product's prices later.

## Data model
```
db      = { store:{name,lang}, products:[{id,name,barcode,unit,cost,price,promo,qty,min}],
            sales:[{id,ts,items:[{pid,name,unit,qty,price,base,cost}],sub,disc,total,profit,saved}], seq }
photos  = { <product id>: "data:image/jpeg;base64,…" }
```
Profit is recorded per sale from the cost price at the moment of the sale, so
changing a cost later never rewrites past reports.

`unit` is one of `pc`, `kg`, `g`, `l`. `promo` is 0 for no deal. On a sale line,
`price` is what was charged and `base` is the normal price. `disc` is the ticket-level
discount and `saved` is every discount on that sale added together.
