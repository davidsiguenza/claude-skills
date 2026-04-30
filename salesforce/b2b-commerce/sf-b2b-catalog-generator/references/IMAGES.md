# Image Sourcing Guide

Every product in the final CSV must have **2–5 verified image URLs**. This document is the playbook for **sourcing** those images (where the photos come from). For **hosting** (what URL the CSV ultimately points to — direct hotlink vs. Cloudinary vs. Salesforce Files vs. GitHub raw), see [IMAGE_HOSTING.md](IMAGE_HOSTING.md). The two are orthogonal: you can source from Wikimedia and re-host in Cloudinary, or source from a supermarket CDN and hotlink directly.

## Sourcing priority

Try sources in this order; stop as soon as each product has 2+ real images:

1. **Vendor / customer website** (product detail pages). Often blocked by JS — WebFetch may return generic category pages.
2. **Wikimedia Commons** — the most reliable source for well-known B2B brands (Volvo, Caterpillar, John Deere, Valtra, Kioti, Ponsse, Bosch, Siemens, etc.). Real photos, CC-licensed, hotlinkable when using the direct URL.
3. **Wikipedia article image** (upload.wikimedia.org) — use same URL conventions as Commons.
4. **Adobe Stock API** (when API key available — see "Adobe Stock (planned)" below).

## Wikimedia Commons workflow

### Find images

```
https://commons.wikimedia.org/wiki/Category:<Brand>_<Model>
https://commons.wikimedia.org/wiki/Category:<Brand>_<ModelFamily>
https://commons.wikimedia.org/wiki/Category:<Brand>
```

WebFetch those URLs and extract filenames. Example category structure you'll find:
- `Category:Volvo_EC220DL` → lists files like `Volvo_EC220D_Dutch_Army.jpg`
- `Category:Ponsse_forwarders` → lists `Ponsse_Buffalo_forwarder.jpg`, etc.

### Resolve a filename → direct URL

Wikimedia stores files at `/wikipedia/commons/<x>/<xx>/<filename>` where `<x>/<xx>` are the first characters of the MD5 hash of the filename. You can't guess them — use the API:

```
https://commons.wikimedia.org/w/api.php?action=query&format=json&prop=imageinfo&iiprop=url&titles=File:<FILENAME>|File:<FILENAME2>
```

WebFetch that URL — the response contains `imageinfo[0].url` with the resolved direct URL for each file. Up to 50 files per request.

### CRITICAL: Do NOT use /thumb/ URLs

URLs like:
```
https://upload.wikimedia.org/wikipedia/commons/thumb/d/df/Volvo_FH16.700.jpg/1024px-Volvo_FH16.700.jpg
```
are **blocked** by Wikimedia for external hotlinking — they return `403 Forbidden` or `400 Bad Request`. The Salesforce B2B Commerce storefront will fail to render them.

Always use the **direct** URL:
```
https://upload.wikimedia.org/wikipedia/commons/d/df/Volvo_FH16.700.jpg
```

(no `/thumb/` and no trailing `NNNpx-...` suffix.)

### URL encoding

Filenames with spaces → `%20`. Filenames with parentheses, accents, special chars → percent-encode them. The Wikimedia API returns properly encoded URLs; prefer using those directly.

## Verification loop

Before finalizing the CSV, run a HTTP check on every unique image URL:

```python
import urllib.request, urllib.error, csv, time

UA = "CatalogBot/1.0 (contact@example.com)"  # Wikimedia requires a real UA with contact

def check(url: str) -> bool:
    try:
        req = urllib.request.Request(url, headers={"User-Agent": UA})
        with urllib.request.urlopen(req, timeout=15) as r:
            return r.status < 400
    except Exception:
        return False

# Collect URLs from the CSV
urls = set()
with open(csv_path) as f:
    for row in csv.DictReader(f):
        for col in ["Media Standard Url 1", "Media Standard Url 2",
                    "Media Standard Url 3", "Media Standard Url 4", "Media Standard Url 5"]:
            v = row.get(col, "").strip()
            if v:
                urls.add(v)

# Verify with a small delay between requests to avoid rate limits
failed = []
for u in urls:
    if not check(u):
        failed.append(u)
    time.sleep(0.3)  # Wikimedia rate-limits around ~10 req/s

if failed:
    print("FAILED URLS:", failed)
```

**Wikimedia returns HTTP 429** under heavy load — treat it as "probably OK, just rate-limited". If a URL was resolved via the API and follows correct conventions, trust it.

## Drop-the-product rule

**If a product cannot get 2 real images after searching Wikimedia + vendor sites, drop the product.** Do not:
- Use the brand logo as a stand-in image
- Duplicate image 1 as image 2
- Use a generic "category" image that isn't the product

The user explicitly prefers fewer products over products with placeholder imagery. Report dropped products in the final summary so the user understands the tradeoff.

## Adobe Stock (planned)

When the user provides an Adobe Stock API key (typically stored as `ADOBE_STOCK_API_KEY` env var in `~/.claude/settings.json`):

```
GET https://stock.adobe.io/Rest/Media/1/Search/Files
Headers:
  x-api-key: <key>
  X-Product: <product-identifier>
Query: search_parameters[words]=<product name>&search_parameters[limit]=5
```

Response includes `files[].thumbnail_url` (watermarked, OK for demos) and `files[].comp_url` (higher-res, licensed-once). Use only licensed URLs for customer-delivered catalogs; thumbnails are fine for internal demos.

This is a future enhancement — the current skill works without Adobe Stock using Wikimedia.

## Related image sources for fallback

- **Wikipedia article hero image**: look at the infobox of the Wikipedia article for the product/model; the image is on Commons.
- **Government / public sector photos** (e.g., military equipment, construction sites): `commons.wikimedia.org/wiki/Category:<Government body>_equipment`.
- **Manufacturer press kits**: some brands provide public press-image pages with usable URLs (e.g., Volvo Group media bank — check the individual terms of use).
