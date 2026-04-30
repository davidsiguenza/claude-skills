# Image Hosting Strategies

This reference complements `IMAGES.md`. `IMAGES.md` covers **sourcing** (where product photos come from — Wikimedia, vendor CDNs, Ahorramás-style supermarket scrapers). This document covers **hosting** (where the URLs in the final CSV point to), which is a separate decision with its own tradeoffs, credentials, and Trusted-URL implications.

**Always ask the user which option they want in Step 1 of SKILL.md.** Image hosting is very hard to change after the fact because the URLs are embedded in every `Media Standard Url N` / `Media Listing Url N` / `Media Attachment Url N` column of the CSV, and once products are imported the URLs are persisted on `ProductMedia` records.

## Option A — Cloudinary

Best choice when the user already has a Cloudinary account (many demo engineers do, as of 2026-04-27). The upload API is simple, URLs are stable and short, and Cloudinary's on-the-fly transformations make it trivial to serve multiple image sizes from a single source.

### What to ask the user

- **Cloud name** (e.g., `dbnwhls0i`) — public, safe to store anywhere.
- **API key + secret** — ask the user to paste them. Never persist the secret to disk. Hold them in env vars only during the Python run.
- **Target folder** (e.g., `sunnie-foods`, `ascendum`). One folder per demo so assets don't collide.

### Upload flow (Python)

```python
import hashlib, time, requests

CLOUD = "<cloud_name>"
KEY   = os.environ["CLOUDINARY_API_KEY"]
SECRET = os.environ["CLOUDINARY_API_SECRET"]
FOLDER = "<folder>"

def upload(local_path, public_id):
    ts = str(int(time.time()))
    to_sign = f"folder={FOLDER}&public_id={public_id}&timestamp={ts}"
    sig = hashlib.sha1((to_sign + SECRET).encode()).hexdigest()
    with open(local_path, "rb") as f:
        r = requests.post(
            f"https://api.cloudinary.com/v1_1/{CLOUD}/image/upload",
            data={"api_key": KEY, "timestamp": ts, "signature": sig,
                  "folder": FOLDER, "public_id": public_id},
            files={"file": f},
            timeout=60,
        )
    r.raise_for_status()
    return r.json()["secure_url"]
```

Returned `secure_url` looks like `https://res.cloudinary.com/<cloud>/image/upload/v<ts>/<folder>/<sku>.jpg` — put that into the CSV as-is.

### Trusted URLs to deploy

- Org-level CSP: `https://res.cloudinary.com` with `img-src`. Add a `CspTrustedSite` as described in SKILL.md Step 2c.
- Site-level: in Experience Builder → Settings → Security → Trusted URLs, add the same domain with `img-src`, then republish.

### Gotchas

- Cloudinary **free tier** has a storage + transformation quota. For a 20-product demo it's irrelevant, but don't upload 10 MB source files per SKU when 200 KB will do.
- Public IDs must be URL-safe. Lowercase the SKU and keep `.jpg` / `.png` extensions; Cloudinary preserves the original extension.
- If the download step sometimes returns PNG (e.g. from supermarket CDNs), the uploaded URL will end in `.png` — write the CSV step to read the extension from the mapping file rather than hardcoding `.jpg`.

## Option B — Direct hotlink from source

Zero-setup, but every source domain must be in Trusted URLs and the URLs break if the source moves or blocks hotlinking.

### When to use

- Demo is ephemeral (scratch org that dies next week).
- Source is stable: Wikimedia Commons, manufacturer press kits.

### Gotchas

- Wikimedia: see IMAGES.md's "CRITICAL: Do NOT use /thumb/ URLs" warning. The direct URL under `/wikipedia/commons/<x>/<xx>/<filename>` works; thumbnail URLs 403.
- Supermarket sites (Ahorramás, Mercadona, etc.): thumbnail URLs contain query-string sizing parameters (`?sw=400&sh=900`). Strip and replace with a clean `?sw=800&sh=800&sm=fit` query before writing the CSV — otherwise the storefront gets tiny images.
- CSP: one `CspTrustedSite` per unique source domain. Wikimedia uses `upload.wikimedia.org` (not `commons.wikimedia.org`). Ahorramás uses `www.ahorramas.com`.
- This option is **fragile** — if the customer demo is long-lived, recommend Option A instead.

## Option C — Salesforce Files (ContentVersion + ContentDistribution)

Most portable: the images ride with the org, so metadata retrieval packages them automatically. No CSP changes needed. Downside: the upload flow is 3-4x more laborious to automate and the resulting URLs are long and org-scoped.

### Upload flow (SOQL/DML sketch)

```text
1. For each local image, create ContentVersion (Title, PathOnClient, VersionData=base64).
2. Query ContentDocumentId from ContentVersion.
3. Create ContentDistribution (ContentVersionId=..., Name=..., PreferencesAllowOriginalDownload=true).
4. Use ContentDistribution.DistributionPublicUrl as the value for the CSV's Media Standard Url N.
```

The public URL looks like `https://<instance>.file.force.com/sfc/dist/version/download/?oid=<org>&ids=<id>&d=<hash>&asPdf=false`.

### Gotchas

- `ContentDistribution` requires **Salesforce CRM Content** to be enabled in the org. Most dev/SDO orgs have it, but confirm before committing.
- The distribution URL can be revoked; if the org's session security hardens, the URL stops resolving.
- Not portable across orgs — cloning the demo to another org requires re-uploading all files.

## Option D — GitHub raw / user-provided CDN

User gives a public GitHub repo path. Push the `images/` folder to the repo; compose URLs as `https://raw.githubusercontent.com/<owner>/<repo>/main/images/<sku>.jpg`.

### Gotchas

- GitHub raw has a rate limit (60 req/hr from an IP by default). For a demo on a shared storefront that's borderline — acceptable for internal testing, risky for a live demo to a client.
- `raw.githubusercontent.com` redirects to `objects.githubusercontent.com` for some payloads — add **both** to Trusted URLs.
- Use a release asset or a release tag to get a cacheable, versioned URL if the demo will be long-lived.

## Regardless of option — the image verification loop

After uploading (Options A, C, D) or deciding on source URLs (Option B), run a HEAD request on every URL in the CSV and assert 200. See the `verify` function in IMAGES.md. Fail the CSV if any URL fails — do not ship broken imagery.

## Reproducibility and resume-safety

Keep a per-demo `image_mapping.csv` checkpoint with columns `sku,name,source,source_url,local_path,cloudinary_url` (or equivalent for other options). The upload script should:

1. Load existing mapping.
2. Skip any SKU that already has a non-empty hosted URL (idempotent resume).
3. Write back to the same path after each SKU.

This means when the Step-4 match-quality review (see SKILL.md) flags 3 out of 36 products as wrong, the operator can blank those 3 rows in the mapping, refine the query, rerun — and only those 3 re-upload. Saves significant time on 100+ product catalogs.

## Lesson learned (2026-04-27)

During the Sunnie Foods demo, first-pass automated matches were ~85% correct (31/36) but the remaining 5 looked plausible to an automated HTTP check: the URLs resolved, the images loaded, they just weren't the right product (pizza dough instead of frozen margherita pizza, 23A 12V battery instead of AA pack, hair shampoo query returned a conditioner, etc.). The **HTTP check is not a semantic check**. Always surface a review table of `SKU | alt text | URL` to the user before considering Step 4 done.
