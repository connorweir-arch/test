# Bloomreach Engagement – Dynamic Product Recommendation Block

A fully parameterised, responsive HTML block for Bloomreach Engagement email campaigns. It renders a grid of product cards powered by a recommendation model, with an optional CTA button beneath.

## Quick start

1. In Bloomreach Engagement, create a new **HTML Block** (Data & Assets → Asset Manager → Blocks → New Block).
2. Open the code editor and paste the contents of [`bloomreach-product-block.html`](bloomreach-product-block.html).
3. Click **"Load parameters from code editor"** — all `params.*` references will be detected automatically.
4. For each parameter, set the **type** and **default value** as documented in the table below (and in the comment block at the top of the HTML file).
5. Save the block and insert it into any email campaign via the asset picker or as a Jinja reference.

## Features

- **Recommendation-driven** — queries any Bloomreach recommendation model via a `recommendation` type parameter.
- **Flexible grid** — control total product count and products-per-row; uses Jinja `|batch()` to split products into rows automatically.
- **Fully inline styles** — every CSS value is driven by a `params.*` reference so non-technical users can restyle the block without touching code.
- **Email-client compatible** — uses nested `<table>` layout with MSO conditional comments and VML roundrect for Outlook button support.
- **Mobile responsive** — CSS `@media` query stacks columns to full width on screens ≤ 620px.
- **Optional price display** — toggle price on/off with a boolean parameter.
- **Optional CTA button** — toggle visibility, label, URL, colours, shape, and spacing.

## Parameters reference

After clicking "Load parameters from code editor", set the following types and defaults:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `recommendationID` | recommendation | `699f134bffe1f2b21aff1773` | Recommendation model to query |
| `productsCount` | number | `4` | Total products to display |
| `numberOfColumns` | number | `2` | Products per row |
| `backgroundColor` | color | `#ffffff` | Block background colour |
| `blockPaddingTop` | number | `20` | Top padding (px) |
| `blockPaddingBottom` | number | `20` | Bottom padding (px) |
| `blockPaddingLeft` | number | `10` | Left padding (px) |
| `blockPaddingRight` | number | `10` | Right padding (px) |
| `blockMaxWidth` | number | `600` | Max content width (px) |
| `cardBgColor` | color | `#f2ece6` | Card background colour |
| `cardBorderRadius` | number | `12` | Card corner radius (px) |
| `cardPadding` | number | `15` | Card inner padding (px) |
| `cardGap` | number | `10` | Gap between cards (px) |
| `imageBorderRadius` | number | `8` | Image corner radius (px) |
| `imageWidth` | number | `200` | Image width (px) |
| `baseFont` | string | `Arial` | Base font family |
| `titleFontSize` | string | `13px` | Title font size |
| `titleFontWeight` | string | `normal` | Title font weight |
| `titleColor` | color | `#333333` | Title text colour |
| `titleAlign` | string | `center` | Title text alignment |
| `titleLineHeight` | string | `1.4` | Title line height |
| `showPrice` | boolean | `false` | Show price below title |
| `priceFontSize` | string | `13px` | Price font size |
| `priceFontWeight` | string | `bold` | Price font weight |
| `priceColor` | color | `#333333` | Price text colour |
| `priceCurrency` | string | `£` | Currency symbol |
| `showCta` | boolean | `true` | Show CTA button |
| `ctaText` | string | `Get reunited` | Button label |
| `ctaUrl` | string | `https://jellycat.com` | Button link URL |
| `ctaBgColor` | color | `#e24740` | Button background colour |
| `ctaTextColor` | color | `#ffffff` | Button text colour |
| `ctaFontSize` | string | `14px` | Button font size |
| `ctaFontWeight` | string | `bold` | Button font weight |
| `ctaBorderRadius` | string | `20px` | Button corner radius |
| `ctaBorderColor` | color | `#e24740` | Button border colour |
| `ctaBorderWidth` | string | `1px` | Button border width |
| `ctaPaddingV` | number | `10` | Button vertical padding (px) |
| `ctaPaddingH` | number | `30` | Button horizontal padding (px) |
| `ctaTopMargin` | number | `20` | Space above button (px) |

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
{{ block('<block-id>', {'recommendationID': '...', 'productsCount': 6, 'numberOfColumns': 3}) }}
```
