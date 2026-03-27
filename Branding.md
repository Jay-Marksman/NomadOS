
***

# NomadOS Boot Branding Specification

This document defines how to transform the starting NomadOS image into all assets needed for Phase 1 Step 5 (boot branding) using GIMP 2.10.36, and specifies fonts, colors, and usage rules for a consistent user experience.

The starting image is `splash.jpg` (NomadOS logo on a starfield).

***

## 1. Goals and constraints

- Optimize for mixed displays: modern 16:9 laptops, older 4:3 LCDs, and VGA‑style virtual machines.  
- Keep boot menus readable on low‑DPI panels and under poor contrast conditions.  
- Maintain a consistent look across:
  - GRUB (UEFI) background
  - ISOLINUX/syslinux (legacy BIOS) background
  - Plymouth splash logo (during kernel boot)

All edits must preserve the logo shape and general starfield feel while ensuring there is a dark, low‑detail area where text and menus appear.

***

## 2. Font theme

Use only widely available, highly legible sans‑serif fonts suitable for low‑DPI screens.

- Primary UI font: **Inter**  
  - Fallback stack: `system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Noto Sans", Arial, sans-serif`.  
- Boot/menu font (where configurable): `Noto Sans` or `DejaVu Sans`, medium weight (500–600).  
- Body text in wizard/docs: 15–17 px Inter or Noto Sans, line height 1.4–1.6.

For any text rendered directly into images (e.g., tagline screens), use Inter or Noto Sans, bold for headings and regular for supporting text.

***

## 3. Color system

Use this palette everywhere (boot screens, Plymouth, setup wizard, docs) for a coherent experience.

- Background base: `#02030A` (near‑black blue).  
- Deep background variant: `#050814`.  
- Accent blue (logo, highlights): `#7FB5FF`.  
- Secondary accent violet: `#9A7DFF`.  
- Primary text: `#E4E7F5`.  
- Muted text / secondary labels: `#9BA4C6`.  

Stars in the background should not be pure white; cap them at approximately `#F0F4FF` to reduce harshness on low‑end panels.

***

## 4. Asset overview

Generate three key assets from `splash.jpg` using GIMP 2.10.36.

1. `nomados-grub-1920x1080.png`  
   - 16:9 background for GRUB (UEFI).  
   - Full starfield plus centered logo and “NomadOS” text.

2. `nomados-syslinux-640x480.png`  
   - 4:3, low‑resolution background for ISOLINUX/syslinux (legacy BIOS).  
   - Simplified background with logo placed higher and darker lower area for menu.

3. `nomados-logo-512.png`  
   - Square logo for Plymouth (no “NomadOS” text).  
   - Transparent or solid dark background.

***

## 5. GIMP instructions — general

Assume GIMP 2.10.36 is installed.

- For scaling, use `Image → Scale Image…` with **Cubic** interpolation.  
- When exporting PNGs:
  - Use default compression level.  
  - Keep 8‑bit RGB color mode and existing color profile.  
- For star edits and darkening, prefer non‑destructive workflows: duplicate layers, use layer masks, and adjust layer opacity instead of destructive painting where possible.

***

## 6. Create `nomados-grub-1920x1080.png` (GRUB background)

1. Open the starting image  
   - `File → Open…` and select `splash.jpg`.

2. Set image size to 1920×1080  
   - `Image → Scale Image…`  
   - Unlock aspect ratio if necessary.  
   - Width: `1920`, Height: `1080`.  
   - Interpolation: Cubic.  
   - Click “Scale”.

3. Center and size the logo region (if needed)  
   - If the logo is not already centered and sized well for 16:9:
     - Use `Layer → Scale Layer…` or the Scale tool to ensure the circular logo and “NomadOS” text sit roughly in the vertical center.  
     - The full logo plus text should be about 35–40% of the image height.

4. Create a “menu‑safe” dark zone on the right  
   - Add a new layer above the background: `Layer → New Layer…`, fill with color `#02030A`.  
   - Add a layer mask: `Layer → Mask → Add Layer Mask… → White (full opacity)`.  
   - Select the Gradient tool.  
   - On the mask, draw a left‑to‑right linear gradient from black (left side, transparent) to white (right side, opaque).  
   - Adjust gradient so the rightmost 400–500 pixels are darker and less star‑dense, while the center and left preserve the original appearance.  
   - Reduce the dark layer’s opacity until the transition is subtle but white menu text (`#E4E7F5`) remains clearly readable on the right.

5. De‑emphasize stars in the right menu area  
   - With the mask still selected, paint with a soft black brush over any large or very bright stars near the right‑center region so that area becomes smoother and less visually noisy.

6. Flatten and export  
   - `Image → Flatten Image`.  
   - `File → Export As…` → `nomados-grub-1920x1080.png`.  
   - Select PNG and click “Export”, keeping default options.

This image will be used as the GRUB theme background. Boot menus should be positioned in the right‑center “menu‑safe” zone.

***

## 7. Create `nomados-syslinux-640x480.png` (legacy BIOS background)

1. Open the GRUB image  
   - `File → Open…` → `nomados-grub-1920x1080.png`.

2. Crop to 4:3  
   - `Image → Canvas Size…`  
   - Set width to `1440`, height to `1080` (4:3 aspect).  
   - Use the preview to anchor so the logo remains horizontally centered and slightly above the vertical center.  
   - Click “Resize”.

3. Scale to 640×480  
   - `Image → Scale Image…`  
   - Width: `640`, Height: `480`.  
   - Interpolation: Cubic.

4. Simplify stars for low resolution  
   - Duplicate the background layer.  
   - On the upper layer, apply a light blur: `Filters → Blur → Gaussian Blur…` with radius around 0.7–1.0 px to soften the tiniest stars and noise.  
   - Add a layer mask to the blurred layer and paint black over the logo so it stays sharp while the outer starfield remains slightly softened.

5. Deepen the lower third for menu text  
   - Add a new layer above all others, filled with `#050814`.  
   - Add a layer mask and apply a vertical gradient from black at the top to white at the bottom so the bottom third becomes noticeably darker.  
   - Adjust opacity until white menu text is clearly legible over the bottom region.

6. Convert to an indexed palette (optional but recommended for older hardware)  
   - `Image → Mode → Indexed…`  
   - “Generate optimum palette” with max colors `64`.  
   - Enable Floyd‑Steinberg dithering.

7. Export  
   - `File → Export As…` → `nomados-syslinux-640x480.png`.  
   - PNG format, default options.

This image will be used as the syslinux/isolinux `MENU BACKGROUND`, with the boot menu placed over the darker lower band.

***

## 8. Create `nomados-logo-512.png` (Plymouth logo)

1. Open the starting image  
   - `File → Open…` → `splash.jpg`.

2. Create a square crop around the circular logo  
   - Choose the Rectangle Select tool.  
   - In the Tool Options, enable “Fixed: Aspect ratio 1:1”.  
   - Drag a square selection around the circular emblem and the immediate surrounding stars, excluding the “NomadOS” text.  
   - `Image → Crop to Selection`.

3. Remove “NomadOS” text if it appears in the crop  
   - Use the Clone Tool or Healing Tool to paint out any remaining text, sampling from nearby background and stars until the area looks natural.

4. Scale to 512×512  
   - `Image → Scale Image…`  
   - Width: `512`, Height: `512`.  
   - Interpolation: Cubic.

5. Optional: make background transparent  
   - If the Plymouth theme will draw its own solid background:
     - Add an alpha channel: `Layer → Transparency → Add Alpha Channel`.  
     - Use the Fuzzy Select tool to select the dark background.  
     - Press `Delete` to clear it, leaving only the logo and a small halo of stars.  
   - If transparency is not desired, keep the dark background color near `#02030A`.

6. Export  
   - `File → Export As…` → `nomados-logo-512.png`.  
   - PNG format, preserving transparency if used.

This logo will be centered in the Plymouth theme with a simple progress animation beneath it.

***

## 9. Usage notes for theme implementers

These notes are for whoever wires the images into GRUB, syslinux, and Plymouth.

- GRUB (UEFI):  
  - Use `nomados-grub-1920x1080.png` as the theme background.  
  - Menu text color: `#E4E7F5`.  
  - Selected entry background: `#050814`; selected entry text: `#7FB5FF`.  
  - Place the menu in the right‑center dark zone reserved in the image.

- ISOLINUX/syslinux (legacy BIOS):  
  - Use `nomados-syslinux-640x480.png` as `MENU BACKGROUND`.  
  - Align menu near the bottom center in the darker band.  

- Plymouth:  
  - Use `nomados-logo-512.png` centered on screen.  
  - Background color: `#02030A`.  
  - Draw a simple animated progress indicator (three or four dots or a thin arc) in accent blue `#7FB5FF` beneath the logo.  
  - Hide verbose text on normal boots and show text only on failure.

***

This `branding.md` fully describes the edits an AI agent should perform in GIMP 2.10.36 starting from `splash.jpg`, and defines the fonts, colors, and usage patterns needed to give end users a clean, readable, and consistent NomadOS boot experience.
