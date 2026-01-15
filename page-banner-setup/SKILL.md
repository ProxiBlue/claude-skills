# Page Banner Setup Skill

## Purpose
Guide Claude through setting up page titles with or without full-width banner backgrounds in Magento 2 / Hyvä Themes projects. This skill covers the two different title/banner layouts used in the NTO project.

## When to Use This Skill
- Setting up custom CMS pages with banners
- Creating calculator or tool pages with branded headers
- Implementing landing pages with full-width hero sections
- Any page requiring a custom title treatment

## Template Options

### 1. Full-Width Banner Template (`title_banner.phtml`)

**Use when:**
- You want a full-width banner background image
- The title should be centered over the banner
- Typically for custom pages, calculator pages, and CMS pages
- You want a hero section look

**Features:**
- Full viewport width banner with background image
- Centered title with customizable CSS classes
- Optional subtitle support
- Responsive and mobile-optimized
- Uses breakout technique to escape container constraints

**Template Location:**
```
app/design/frontend/Uptactics/nto/Magento_Theme/templates/html/title_banner.phtml
```

### 2. Standard Title Template (`title.phtml`)

**Use when:**
- You want a simple page title without banner
- Standard Hyvä Themes title styling is sufficient
- No background image needed
- Left-aligned, uppercase title in the main content flow

**Features:**
- Simple title with custom CSS classes
- Standard Magento title behavior
- No background image
- Contained within normal page flow

**Template Location:**
```
app/design/frontend/Uptactics/nto/Magento_Theme/templates/html/title.phtml
```

## Implementation Pattern

### Step 1: Remove Default Title Block

Always start by removing the default `page.main.title` block:

```xml
<referenceBlock name="page.main.title" remove="true"/>
```

### Step 2: Add Custom Title Block

#### Option A: Full-Width Banner (Recommended for Custom Pages)

Add to `content.top` container for full-width effect:

```xml
<referenceContainer name="content.top">
    <block class="Magento\Theme\Block\Html\Title"
           name="custom.page.title"
           template="Magento_Theme::html/title_banner.phtml"
           before="-">
        <arguments>
            <argument name="css_class" xsi:type="string">text-2xl sm:text-3xl lg:text-4xl text-white uppercase</argument>
            <argument name="banner_image" xsi:type="string">Uptactics_ModuleName::images/banner-image.svg</argument>
            <!-- Optional subtitle -->
            <argument name="subtitle" xsi:type="string" translate="true">Optional descriptive subtitle</argument>
            <argument name="subtitle_css_class" xsi:type="string">text-lg text-white mt-2</argument>
        </arguments>
    </block>
</referenceContainer>
```

**Arguments:**
- `css_class` (string): Tailwind CSS classes for the title (responsive sizing, colors, etc.)
- `banner_image` (string): Path to banner image in format `ModuleName::images/filename.ext`
- `subtitle` (string, optional): Additional subtitle text below the title
- `subtitle_css_class` (string, optional): CSS classes for subtitle styling

#### Option B: Standard Title (For Simple Pages)

Add to `content` container:

```xml
<referenceContainer name="content">
    <block class="Magento\Theme\Block\Html\Title"
           name="custom.page.title"
           template="Magento_Theme::html/title.phtml"
           before="-">
        <arguments>
            <argument name="css_class" xsi:type="string">text-3xl font-bold</argument>
        </arguments>
    </block>
</referenceContainer>
```

## Common Banner Image Paths

### Module-Based Images
Store banner images in your custom module's view folder:
```
app/code/Uptactics/ModuleName/view/frontend/web/images/banner.svg
```

Reference in layout XML:
```xml
<argument name="banner_image" xsi:type="string">Uptactics_ModuleName::images/banner.svg</argument>
```

### Theme-Based Images
Store banner images in theme:
```
app/design/frontend/Uptactics/nto/web/images/banner.svg
```

Reference in layout XML:
```xml
<argument name="banner_image" xsi:type="string">images/banner.svg</argument>
```

## CSS Class Recommendations

### Title Classes (Responsive Sizing)
```xml
<!-- Large hero titles -->
<argument name="css_class" xsi:type="string">text-2xl sm:text-3xl lg:text-4xl text-white uppercase</argument>

<!-- Medium titles -->
<argument name="css_class" xsi:type="string">text-xl sm:text-2xl lg:text-3xl text-white uppercase</argument>

<!-- Small titles -->
<argument name="css_class" xsi:type="string">text-lg sm:text-xl lg:text-2xl text-white uppercase</argument>
```

### Color Options
- White text (for dark banners): `text-white`
- Primary brand color: `text-primary`
- Custom color: `text-[#hexcode]`

## Real-World Example: RCC Calculator Page

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <!-- Remove default title -->
        <referenceBlock name="page.main.title" remove="true"/>

        <!-- Add custom title with banner -->
        <referenceContainer name="content.top">
            <block class="Magento\Theme\Block\Html\Title"
                   name="calculator.page.title"
                   template="Magento_Theme::html/title_banner.phtml"
                   before="-">
                <arguments>
                    <argument name="css_class" xsi:type="string">text-2xl sm:text-3xl lg:text-4xl text-white uppercase</argument>
                    <argument name="banner_image" xsi:type="string">Uptactics_Rcc::images/rcc-banner.svg</argument>
                </arguments>
            </block>
        </referenceContainer>

        <!-- Rest of page content -->
        <referenceContainer name="content">
            <!-- Calculator widget or content blocks -->
        </referenceContainer>
    </body>
</page>
```

## Decision Tree

```
Do you need a banner image?
├─ YES → Use title_banner.phtml
│   ├─ Place in content.top container
│   ├─ Provide banner_image argument
│   └─ Use white text for visibility
│
└─ NO → Use title.phtml
    ├─ Place in content container
    ├─ Use standard title styling
    └─ No banner_image argument needed
```

## Technical Details

### title_banner.phtml Behavior
- Uses full viewport width (`w-screen`)
- Breaks out of container using negative margins: `left: 50%; right: 50%; margin-left: -50vw; margin-right: -50vw;`
- Centers content with `container mx-auto`
- Minimum height: 222px
- Background image covers and centers
- Falls back to standard title if no banner image provided

### title.phtml Behavior
- Standard Magento title block
- Left-aligned, uppercase
- Contained within normal page flow
- Uses Hyvä theme default styling

## Common Mistakes to Avoid

1. **Wrong container placement**
   - ❌ Placing `title_banner.phtml` in `content` container (won't be full-width)
   - ✅ Place `title_banner.phtml` in `content.top` container

2. **Missing banner_image argument**
   - Template will fall back to standard title behavior
   - Always provide `banner_image` when using `title_banner.phtml`

3. **Incorrect image path format**
   - ❌ `Uptactics_Rcc/images/banner.svg` (wrong separator)
   - ✅ `Uptactics_Rcc::images/banner.svg` (correct separator)

4. **Not removing default title**
   - Always remove `page.main.title` block first
   - Otherwise you'll have duplicate titles

## Troubleshooting

### Banner not full-width
- Check that block is in `content.top` container, not `content`
- Verify layout XML is in correct location
- Clear layout cache: `bin/magento cache:flush layout`

### Banner image not showing
- Verify image path is correct
- Check image exists in module/theme `web/images/` directory
- Clear static content: `bin/magento setup:static-content:deploy -f`
- Check browser console for 404 errors

### Title not styled correctly
- Verify `css_class` argument is set
- Check Tailwind classes are valid
- Rebuild Tailwind CSS if using custom classes
- Clear full_page cache

## Cache Management

After making layout changes:
```bash
bin/magento cache:flush layout block_html full_page
```

After adding/changing banner images:
```bash
rm -rf pub/static/frontend/Uptactics/nto/*
bin/magento setup:static-content:deploy -f
```

## Related Files
- `/app/design/frontend/Uptactics/nto/Magento_Theme/templates/html/title_banner.phtml` - Full-width banner template
- `/app/design/frontend/Uptactics/nto/Magento_Theme/templates/html/title.phtml` - Standard title template
- `/app/design/frontend/Uptactics/nto/Magento_Catalog/templates/category/banner_container.phtml` - Category banner (similar pattern)

## Key Takeaways
1. Use `title_banner.phtml` for full-width banners with centered titles
2. Use `title.phtml` for simple page titles without banners
3. Always place full-width banners in `content.top` container
4. Always remove default `page.main.title` block first
5. Provide responsive CSS classes for title sizing
6. Banner images should be in SVG or optimized image format
