# Bloomreach Engagement – Dynamic Product Recommendation Block

A fully parameterized, responsive HTML block for Bloomreach Engagement email campaigns. Displays products from a recommendation engine in a configurable grid layout, styled to match the Jellycat email aesthetic.

## Setup

1. In Bloomreach Engagement, go to **Data & Assets > Asset Manager > Blocks** and click **New Block**.
2. Paste the contents of `bloomreach-product-recommendation-block.html` into the HTML code editor.
3. Click **"Load parameters from the code editor"** to auto-populate all configurable parameters.
4. Adjust parameter values via the visual parameter editor — no coding required.

## Parameters Reference

All parameters are organised into categories and will appear in the Bloomreach visual editor.

### Recommendation Settings

| Parameter | Type | Default | Description |
|---|---|---|---|
| `recommendationModel` | recommendation | `699f134bffe1f2b21aff1773` | Recommendation engine to query |
| `productCount` | number | `2` | Total number of products to display |
| `productsPerRow` | number | `2` | Desktop products per row |
| `mobileProductsPerRow` | number | `1` | Mobile products per row |

### Layout

| Parameter | Type | Default | Description |
|---|---|---|---|
| `blockMaxWidth` | number | `600` | Max width of the block (px) |
| `blockBgColor` | color | `#ffffff` | Block background color |
| `blockPaddingTop` | number | `20` | Top padding (px) |
| `blockPaddingBottom` | number | `20` | Bottom padding (px) |
| `blockPaddingLeft` | number | `16` | Left padding (px) |
| `blockPaddingRight` | number | `16` | Right padding (px) |
| `mobileBreakpoint` | number | `480` | Mobile breakpoint width (px) |

### Product Card

| Parameter | Type | Default | Description |
|---|---|---|---|
| `cardBgColor` | color | `#f2ece6` | Card background color |
| `cardBorderRadius` | number | `12` | Card corner radius (px) |
| `cardPadding` | number | `10` | Card inner padding (px) |
| `cardGap` | number | `16` | Space between cards (px) |

### Product Image

| Parameter | Type | Default | Description |
|---|---|---|---|
| `imageBorderRadius` | number | `8` | Image corner radius (px) |

### Product Title

| Parameter | Type | Default | Description |
|---|---|---|---|
| `titleFontFamily` | string | `Georgia, Times, serif` | Title font family |
| `titleFontSize` | number | `13` | Title font size (px) |
| `titleColor` | color | `#333333` | Title text color |
| `titleFontWeight` | enum | `normal` | Title font weight |
| `titlePaddingTop` | number | `10` | Space above title (px) |
| `titleTextAlign` | enum | `center` | Title alignment |
| `titleLineHeight` | number | `18` | Title line height (px) |
| `titleTextDecoration` | enum | `none` | Title link decoration |
| `titleLinkColor` | color | `#333333` | Title link color |

### Product Price

| Parameter | Type | Default | Description |
|---|---|---|---|
| `showPrice` | boolean | `false` | Show/hide prices |
| `priceFontFamily` | string | `Arial, Helvetica, sans-serif` | Price font family |
| `priceFontSize` | number | `13` | Price font size (px) |
| `priceColor` | color | `#333333` | Price text color |
| `priceFontWeight` | enum | `bold` | Price font weight |
| `pricePrefix` | string | `£` | Currency symbol |
| `pricePaddingTop` | number | `4` | Space above price (px) |

### CTA Button

| Parameter | Type | Default | Description |
|---|---|---|---|
| `showCta` | boolean | `true` | Show/hide the CTA |
| `ctaText` | string | `Get reunited` | Button label |
| `ctaUrl` | string | `https://jellycat.com` | Button link URL |
| `ctaBgColor` | color | `#e4554a` | Button background |
| `ctaTextColor` | color | `#ffffff` | Button text color |
| `ctaFontFamily` | string | `Arial, Helvetica, sans-serif` | Button font |
| `ctaFontSize` | number | `14` | Button text size (px) |
| `ctaBorderRadius` | number | `24` | Button corner radius (px) |
| `ctaPaddingVertical` | number | `12` | Vertical padding (px) |
| `ctaPaddingHorizontal` | number | `32` | Horizontal padding (px) |
| `ctaMarginTop` | number | `24` | Space above button (px) |
| `ctaBorderColor` | color | `#e4554a` | Button border color |
| `ctaBorderWidth` | number | `1` | Button border width (px) |
| `ctaFontWeight` | enum | `normal` | Button font weight |

## Catalog Fields Used

The block references these fields from the `bigcommerce product` catalog (via the recommendation model):

| Field | Usage |
|---|---|
| `image` | Product image |
| `title` | Product name |
| `url` | Link destination |
| `price` | Product price (when `showPrice` is enabled) |

## Responsive Behaviour

- **Desktop**: Products arranged in a grid defined by `productsPerRow` (default 2 side-by-side).
- **Mobile** (below `mobileBreakpoint`, default 480 px): Products stack or rearrange per `mobileProductsPerRow` (default 1 per row).
- MSO / Outlook conditional comments included for Outlook desktop rendering support.
