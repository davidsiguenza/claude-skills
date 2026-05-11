# DigitalExperienceBundle Branding for B2B Commerce LWR

Use this reference from both full-store builds and seeded-SDO demo builds when branding a Commerce Store (LWR) site through metadata.

## Files to edit

| File | Purpose |
|---|---|
| `sfdc_cms__brandingSet/<SET>/content.json` | Color tokens, text/link colors, base font, and the main `SiteLogo` keys. Apply colors to every branding set present, not only `B2B_Commerce`. Seeded SDO bundles can include `Home_Header` in addition to `B2B_Commerce`, `B2B_Footer`, `B2B_Home_Banner`, and `B2B_Right_Panel`. |
| `sfdc_cms__themeLayout/commerceLayout/content.json` and `myAccountLayout/content.json` | Header/footer rich text and top promo content. Replace stock phrases such as "Exclusive winter sale". |
| `sfdc_cms__view/home/content.json` | Hero text, buttons, image URLs, product/category banners, custom components, parallax copy, and buyer-group-specific variants. |
| `sfdc_cms__view/login*/content.json`, `register/content.json`, `forgotPassword/content.json` | Login/register logo references and stock text. |
| `sfdc_cms__styles/styles_css/styles.css` | CSS variables and button/link overrides. |
| `sfdc_cms__site/<SITE>/content.json` | Site title and deploy-breaking properties such as `geoBotsAllowed`. |

## Retrieve and deploy safely

Retrieve only the target bundle. Deploying every bundle in `digitalExperiences` can redeploy old work from previous demos.

```bash
sf project retrieve start --target-org <alias> --metadata "DigitalExperienceBundle:site/<SITE_BUNDLE>"
sf project deploy start --target-org <alias> --source-dir force-app/main/default/digitalExperiences/site/<SITE_BUNDLE> --wait 20
sf community publish --name '<Site Name>' --target-org <alias>
```

After retrieve, remove `geoBotsAllowed` from `sfdc_cms__site/<SITE>/content.json`; deploy rejects it:

```python
import json
from pathlib import Path

p = Path("force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/sfdc_cms__site/<SITE_BUNDLE>/content.json")
d = json.loads(p.read_text())

def strip(node):
    if isinstance(node, dict):
        node.pop("geoBotsAllowed", None)
        for value in node.values():
            strip(value)
    elif isinstance(node, list):
        for item in node:
            strip(item)

strip(d)
p.write_text(json.dumps(d, indent=2) + "\n")
```

## Logo

For LWR Commerce, deploy the logo as a `StaticResource` and reference it with the published site path:

```json
"SiteLogo": "/sfsites/c/resource/<ResourceName>",
"_SiteLogoUrl": "url(/sfsites/c/resource/<ResourceName>)"
```

Do not use `/resource/<ResourceName>` for Commerce LWR headers. It can return `200` and still fail to render in the site header.

Use a wide wordmark SVG or image. Recolor white-on-transparent logos when the header background is white.

## Home banners and stock content cleanup

Search the full home JSON, not just the first hero component. Replace every stock/template/SDO asset and text string.

Search terms that have shipped as missed branding before:

```text
assets/images/
sfdc-ckz-b2b.s3.amazonaws.com/SDO/Commerce_Images
Emergency Prep
Discover Nature
Solar
Energy
Wind
Battery
Cirrus
```

For every `imageInfo` whose inner `imageInfoV1.url` is stock/SDO/Cirrus, rewrite the URL to a brand-relevant `https://...` image and update `altText`. Verify the URL returns image content:

```bash
curl -sI -A "Mozilla/5.0" "<image_url>" | head -5
```

Examples from the validated Dentaid run:

- `emergency-prep-b2b.jpg` + visible title `Emergency Prep` became a Dentaid product kit image + `Professional Care Kits`.
- `c:parallaxcmp` title `Discover Nature's Energy` became `Committed to Sustainable Oral Health`.

If a custom component exposes only an enumerated background image, changing the visible copy is acceptable; do not leave stock energy-themed copy in a client demo.

## Buyer group variants

To create different home content for Standard vs Silver/VIP buyers, duplicate the target component tree in `sfdc_cms__view/home/content.json`, generate fresh UUIDs for every copied `id`, then add `contentOperations.operations`.

Show the VIP/Silver component only for the buyer group:

```json
{
  "targetId": "<VIP_BANNER_COMPONENT_ID>",
  "isHiddenOnOperationSuccess": false,
  "isActive": true,
  "rule": {
    "name": "<UUID>",
    "description": "Show VIP banner for VIP Customers Buyer Group",
    "criteriaType": "AllCriteriaMatch",
    "expressionCriteria": [{
      "resource": "User.Commerce.BuyerGroups",
      "operator": "Contains",
      "value": "<VIP_OR_SILVER_BUYER_GROUP_NAME>",
      "criterionNumber": 1
    }]
  }
}
```

Hide the default component for the same buyer group:

```json
{
  "targetId": "<DEFAULT_BANNER_COMPONENT_ID>",
  "isHiddenOnOperationSuccess": true,
  "isActive": true,
  "rule": {
    "name": "<UUID>",
    "description": "Hide default banner for VIP Customers Buyer Group",
    "criteriaType": "AllCriteriaMatch",
    "expressionCriteria": [{
      "resource": "User.Commerce.BuyerGroups",
      "operator": "Contains",
      "value": "<VIP_OR_SILVER_BUYER_GROUP_NAME>",
      "criterionNumber": 1
    }]
  }
}
```

Do not use `User.AccountId`, `User.ProfileId`, `User.UserType`, or custom `User`/`Contact` fields for Commerce buyer-group visibility. B2B Commerce LWR deploy validation rejects those expressions with `Enter a valid expression`.

Validate with one buyer account that belongs only to the VIP/Silver group and one buyer account that belongs only to the Standard group. Accounts in both groups match the VIP/Silver rule.
