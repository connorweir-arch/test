# Bloomreach Engagement – Dynamic Product Recommendation Block

A fully parameterised, responsive HTML block for Bloomreach Engagement email campaigns. It renders a grid of product cards powered by a recommendation model, with an optional CTA button beneath.

## Quick start

1. In Bloomreach Engagement, create a new **HTML Block** (Data & Assets → Asset Manager → Blocks → New Block).
2. Open the code editor and paste the contents of [`bloomreach-product-block.html`](bloomreach-product-block.html).
3. Click **"Load parameters from code editor"** — all parameters will be detected automatically.
4. Set the **type** and **default value** for each parameter as documented below (the defaults in the code already match the Jellycat-style reference design).
5. Save the block and insert it into any email campaign via the asset picker or as a Jinja reference.

## Features

- **Recommendation-driven** — queries any Bloomreach recommendation model via a selectable `recommendation` parameter.
- **Flexible grid** — control total product count and products-per-row from the parameters UI.
- **Fully inline styles** — every CSS value is driven by a parameter (colours, spacing, typography, border radii, etc.), so non-technical users can restyle the block without touching code.
- **Email-client compatible** — uses nested `<table>` layout with MSO conditional comments for Outlook support.
- **Optional price display** — toggle price on/off with a boolean parameter.
- **Optional CTA button** — toggle visibility, label, URL, colours, shape, and spacing.

## Parameters reference

| Parameter | Type | Default | Category | Description |
|---|---|---|---|---|
| `recoModel` | recommendation | `699f134bffe1f2b21aff1773` | Data | Recommendation model to query |
| `totalProducts` | number | `4` | Data | Total products to display |
| `productsPerRow` | number | `2` | Data | Products per row |
| `blockBgColor` | color | `#ffffff` | Container | Block background colour |
| `blockPaddingTop` | number | `20` | Container | Top padding (px) |
| `blockPaddingBottom` | number | `20` | Container | Bottom padding (px) |
| `blockPaddingLeft` | number | `10` | Container | Left padding (px) |
| `blockPaddingRight` | number | `10` | Container | Right padding (px) |
| `blockMaxWidth` | number | `600` | Container | Max content width (px) |
| `cardBgColor` | color | `#f2ece6` | Product Card | Card background colour |
| `cardBorderRadius` | number | `12` | Product Card | Card corner radius (px) |
| `cardPadding` | number | `15` | Product Card | Card inner padding (px) |
| `cardGap` | number | `10` | Product Card | Gap between cards (px) |
| `imageBorderRadius` | number | `8` | Product Image | Image corner radius (px) |
| `imageMaxHeight` | number | `0` | Product Image | Max image height (px, 0 = none) |
| `titleFontFamily` | string | `Arial, Helvetica, sans-serif` | Typography | Product title font |
| `titleFontSize` | number | `13` | Typography | Title font size (px) |
| `titleFontWeight` | enum | `normal` | Typography | Title font weight |
| `titleColor` | color | `#333333` | Typography | Title text colour |
| `titleAlign` | enum | `center` | Typography | Title text alignment |
| `titlePaddingTop` | number | `10` | Typography | Space above title (px) |
| `titleLineHeight` | string | `1.4` | Typography | Title line height |
| `showPrice` | boolean | `false` | Price | Show price below title |
| `priceFontSize` | number | `13` | Price | Price font size (px) |
| `priceColor` | color | `#333333` | Price | Price text colour |
| `priceFontWeight` | enum | `bold` | Price | Price font weight |
| `pricePrefix` | string | `£` | Price | Currency prefix |
| `showCta` | boolean | `true` | CTA Button | Show CTA button |
| `ctaText` | string | `Get reunited` | CTA Button | Button label |
| `ctaUrl` | string | `https://jellycat.com` | CTA Button | Button link URL |
| `ctaBgColor` | color | `#e24740` | CTA Button | Button background colour |
| `ctaTextColor` | color | `#ffffff` | CTA Button | Button text colour |
| `ctaFontSize` | number | `14` | CTA Button | Button font size (px) |
| `ctaFontFamily` | string | `Arial, Helvetica, sans-serif` | CTA Button | Button font |
| `ctaFontWeight` | enum | `bold` | CTA Button | Button font weight |
| `ctaBorderRadius` | number | `20` | CTA Button | Button corner radius (px) |
| `ctaPaddingV` | number | `10` | CTA Button | Button vertical padding (px) |
| `ctaPaddingH` | number | `30` | CTA Button | Button horizontal padding (px) |
| `ctaTopMargin` | number | `20` | CTA Button | Space above button (px) |
| `ctaBorderColor` | color | `#e24740` | CTA Button | Button border colour |
| `ctaBorderWidth` | number | `1` | CTA Button | Button border width (px) |

## Product catalog fields used

The block reads these fields from each recommended product:

| Field | Example |
|---|---|
| `url` | `https://jellycat.com/merry-mouse/` |
| `image` | `https://cdn11.bigcommerce.com/…/MER3M__69389…` |
| `title` | `Merry Mouse` |
| `price` | `28.00` |

## Calling the block from a campaign

```jinja
{# Without parameter overrides (uses defaults): #}
{{ block('<block-id>') }}

{# With parameter overrides: #}
{{ block('<block-id>', {'recoModel': '...', 'totalProducts': 6, 'productsPerRow': 3}) }}
```
