# Design System Specification: The Ethereal Archive

## 1. Overview & Creative North Star
**Creative North Star: "The Digital Keepsake"**
This design system moves away from the rigid, grid-heavy structures of modern web apps to embrace the feel of a curated, high-end editorial scrap-book. The objective is to evoke "nostalgia through light"—mimicking the hazy, warm glow of a late summer afternoon. 

To break the "template" look, designers must prioritize **intentional asymmetry**. Layouts should feel organic, with images occasionally breaking the container bounds and typography scales that lean into dramatic contrasts. We are not just building a website; we are crafting a digital vessel for memories that feels as tactile and precious as a physical yearbook.

---

## 2. Colors & Surface Philosophy
The palette is rooted in the warmth of a sunset and the softness of moonlight. We utilize a Material-inspired tonal system but apply it with an editorial touch.

### The "No-Line" Rule
**Explicit Instruction:** 1px solid borders for sectioning are strictly prohibited. Boundaries must be defined solely through:
- **Tonal Shifts:** Placing a `surface-container-low` section against a `surface` background.
- **Negative Space:** Using generous padding to imply containment.
- **Glass Transitions:** Using backdrop blurs to separate floating layers.

### Surface Hierarchy & Nesting
Treat the UI as a series of stacked sheets of fine, semi-transparent vellum. 
- **Base Layer:** `surface` (#faf9f8) for the main canvas.
- **Nested Depth:** Use `surface-container-lowest` (#ffffff) for primary content cards to create a soft "lift." Use `surface-container-high` (#e7e8e7) for recessed areas like sidebars or footer foundations.

### The "Glass & Gradient" Rule
To achieve the "dreamy" aesthetic, use Glassmorphism for navigation bars and floating modals.
- **Glass Specs:** Background: `surface` at 70% opacity | Backdrop-blur: 16px to 24px.
- **Signature Gradients:** For CTAs and Hero backgrounds, use a linear gradient: `primary_container` (#fecbcb) to `surface_bright` (#faf9f8) at a 135-degree angle. This mimics the soft bleed of watercolor paint.

---

## 3. Typography
The typographic soul of this system lies in the tension between the academic rigor of a serif and the modern clarity of a sans-serif.

- **Display & Headlines (Noto Serif):** These are your "Editorial Voices." Use `display-lg` for hero moments to evoke the feeling of a prestige magazine title. The serif conveys history, nostalgia, and importance.
- **Body & Labels (Plus Jakarta Sans):** These provide the "Contemporary Bridge." The clean, geometric nature of Plus Jakarta Sans ensures high legibility for long-form student bios and captions, preventing the design from feeling dated.
- **Hierarchy Hint:** Always pair a `headline-sm` in Noto Serif with a `label-md` in Plus Jakarta Sans (All Caps, 0.05rem letter spacing) to create a sophisticated, high-end caption style.

---

## 4. Elevation & Depth
We reject harsh drop shadows. Depth in this system is atmospheric, not structural.

- **Tonal Layering:** Instead of shadows, stack `surface-container-lowest` on `surface_container`. The subtle shift in hex value provides enough contrast for the eye to perceive depth without visual clutter.
- **Ambient Shadows:** When a float is required (e.g., a "Memory Card"), use:
  - `box-shadow: 0 12px 40px rgba(124, 85, 86, 0.06);` (A tinted shadow using the `primary` tone).
- **The "Ghost Border" Fallback:** If accessibility requires a stroke, use `outline-variant` at 15% opacity. It should be felt, not seen.
- **Moonlight Motifs:** Use circular, soft-edged radial gradients (Primary to Transparent) in the background of sections to act as "spotlights" for content, mimicking moonlight filtering through trees.

---

## 5. Components

### Buttons
- **Primary:** High-pill shape (`rounded-full`). Background: `secondary` (#ad3135). Text: `on_secondary`. Use a subtle inner-glow (top-down white gradient at 10% opacity) to give it a "jewel" feel.
- **Secondary:** Glass-style. Background: `surface_container_lowest` at 50% opacity with a 12px backdrop-blur. 
- **Tertiary:** Text-only in `primary` (#7c5556) with a `title-sm` weight.

### Cards & Lists
- **The Memory Card:** No borders. `rounded-lg` (2rem). Background: `surface-container-lowest`. 
- **List Items:** Forbid divider lines. Separate items using 16px of vertical white space. Use a `tertiary_container` (#f5ce53) dot (the "Moonlight Bullet") for list indicators.

### Input Fields
- **Styling:** `surface-container-low` background, `rounded-sm` (0.5rem) corners.
- **Focus State:** Instead of a heavy border, use a 2px "Glow" using `primary_fixed` (#fecbcb) with a 4px blur.

### Imagery & Illustrations
- **Hoa Phượng (Flamboyant Flowers):** Use as asymmetrical "corner bleeds." These should be soft-focus botanical illustrations that overlap the edges of cards, breaking the boxy layout.
- **Image Treatment:** All photos should have a `rounded-md` (1.5rem) corner radius.

---

## 6. Do's and Don'ts

### Do:
- **Embrace White Space:** Treat the screen like a physical page. Let the "Flowers" and "Moonlight" motifs breathe.
- **Use Intentional Overlaps:** Let a serif headline slightly overlap an image to create depth and an editorial feel.
- **Prioritize Softness:** Use the `xl` (3rem) corner radius for large layout containers to maintain the whimsical, innocent vibe.

### Don't:
- **Don't use pure black (#000000):** Use `on_surface` (#303333) for all "black" text to keep the contrast soft and readable.
- **Don't use sharp corners:** Even the smallest components should have at least a `sm` (0.5rem) radius.
- **Don't use standard Dividers:** If you need to separate content, use a wide gap or a subtle change in background tone. Never a solid grey line.
- **Don't over-saturate:** Keep the Soft Red (`secondary`) for accents only—it should be the "pop" of the flower, not the color of the whole garden.