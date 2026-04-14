# Bloomreach Engagement - Dynamic Product Recommendation Block

A fully parameterized, responsive HTML block for email product recommendations in Bloomreach Engagement. Paste the code into an HTML Block editor, then click **"Load parameters from the code editor"** to import all customizable parameters.

## Quick start

1. In Bloomreach Engagement, navigate to **Data & Assets > Asset Manager > Blocks** and create a new block.
2. Open `bloomreach-product-recommendation-block.html` and copy the entire contents.
3. Paste into the HTML code editor of the block.
4. Click **"Load parameters from the code editor"** — all parameters will appear in the visual editor.
5. Adjust parameter values as needed (recommendation model, colours, typography, layout, etc.).
6. Save the block and insert it into your email campaigns.

## Parameters reference

### Recommendation

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `recommendationModel` | recommendation | `699f134bffe1f2b21aff1773` | The recommendation engine to query |
| `totalProducts` | number | `2` | How many products to fetch |
| `fillWithRandom` | boolean | `true` | Fill with random products if the model returns fewer items |

### Layout

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `containerWidth` | number | `600` | Maximum width of the block in pixels |
| `productsPerRow` | number | `2` | Number of product cards per row |
| `stackOnMobile` | boolean | `true` | Stack cards vertically on mobile |
| `mobileBreakpoint` | number | `480` | Screen width (px) below which mobile styles apply |

### Container

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `containerBgColor` | color | `#ffffff` | Background colour of the outer container |
| `containerPaddingTop` | number | `20` | Top padding of the container (px) |
| `containerPaddingBottom` | number | `20` | Bottom padding of the container (px) |
| `containerPaddingHorizontal` | number | `10` | Left/right padding of the container (px) |

### Product card

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `cardBgColor` | color | `#ffffff` | Background colour of each card |
| `cardGap` | number | `16` | Gap between cards (px) |
| `cardBorderRadius` | number | `0` | Border radius of each card (px) |
| `cardBorderWidth` | number | `0` | Border width of each card (px) |
| `cardBorderColor` | color | `#dddddd` | Border colour of each card |
| `cardPadding` | number | `0` | Inner padding of each card (px) |
| `cardVerticalAlign` | enum | `top` | Vertical alignment of card content (top / middle / bottom) |

### Product image

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `imageBgColor` | color | `#F0EBE3` | Background colour of the image area |
| `imageBorderRadius` | number | `8` | Border radius of the image area (px) |
| `imagePadding` | number | `15` | Padding inside the image area (px) |
| `imageHeight` | number | `200` | Total height of the image area (px) |

### Product name

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nameFontFamily` | string | `Arial, sans-serif` | Font family for the product name |
| `nameFontSize` | number | `13` | Font size of the product name (px) |
| `nameColor` | color | `#333333` | Text colour of the product name |
| `nameTextAlign` | enum | `center` | Text alignment (left / center / right) |
| `nameFontWeight` | enum | `normal` | Font weight |
| `nameLineHeight` | number | `18` | Line height (px) |
| `namePaddingTop` | number | `12` | Space above the product name (px) |
| `nameTextDecoration` | enum | `none` | Text decoration (none / underline) |

### Price

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `showPrice` | boolean | `false` | Show the product price |
| `priceColor` | color | `#333333` | Text colour of the price |
| `priceFontSize` | number | `14` | Font size of the price (px) |
| `priceFontFamily` | string | `Arial, sans-serif` | Font family for the price |
| `priceFontWeight` | enum | `bold` | Font weight of the price |
| `priceCurrencySymbol` | string | `£` | Currency symbol prepended to the price |
| `pricePaddingTop` | number | `4` | Space above the price (px) |

### CTA button

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `showCta` | boolean | `true` | Show a CTA button below the product grid |
| `ctaText` | string | `Get reunited` | Button text |
| `ctaUrl` | string | `https://jellycat.com` | Button link URL |
| `ctaBgColor` | color | `#ffffff` | Button background colour |
| `ctaTextColor` | color | `#cc3333` | Button text colour |
| `ctaBorderColor` | color | `#cc3333` | Button border colour |
| `ctaBorderWidth` | number | `1` | Button border width (px) |
| `ctaBorderRadius` | number | `25` | Button border radius (px) |
| `ctaFontFamily` | string | `Arial, sans-serif` | Button font family |
| `ctaFontSize` | number | `14` | Button font size (px) |
| `ctaFontWeight` | enum | `normal` | Button font weight |
| `ctaPaddingVertical` | number | `10` | Vertical padding inside button (px) |
| `ctaPaddingHorizontal` | number | `30` | Horizontal padding inside button (px) |
| `ctaMarginTop` | number | `20` | Space above the button (px) |
| `ctaLetterSpacing` | number | `0` | Letter spacing (px) |
| `ctaTextTransform` | enum | `none` | Text transform (none / uppercase / lowercase / capitalize) |

## Catalog fields used

The block reads the following fields from the **bigcommerce product** catalog via the recommendation model:

- `item.image` — product image URL
- `item.title` — product name
- `item.url` — product page link
- `item.price` — product price (displayed when `showPrice` is `true`)

## Calling as a referenced block

```jinja2
{{ block('<block_id>') }}
```

Or with parameter overrides:

```jinja2
{{ block('<block_id>', {
  'totalProducts': 4,
  'productsPerRow': 4,
  'showPrice': true
}) }}
```
