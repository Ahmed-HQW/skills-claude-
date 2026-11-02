---
name: school-presentation-style
description: Create clean, modern PowerPoint slides for school/university presentations (cours, exposés) in a minimalist style — light backgrounds, navy/indigo text, soft purple accent cards, rounded corners, subtle gradient blobs in the corners. Use whenever the user asks for a "présentation PowerPoint" for school, a course topic, or an "exposé", especially in French/Darija contexts.
---

# School Presentation Style

## When to use
Trigger this skill whenever someone asks to build a PowerPoint (.pptx) presentation for school, a course, an exposé, or a lesson topic (math, computer science, economics, etc.), and no other specific design is requested.

## Visual identity (reference style)
- **Background:** very light, near-white (#F4F3F8 or similar), sometimes a deep navy/blue (#1B3BE0-ish) used for contrast/section slides.
- **Primary text color:** dark navy blue (#1A2550 / #1B2A4A), bold, all-caps for titles.
- **Accent cards:** soft lavender/purple rounded rectangles (#CFC9F2-ish) used to hold short labels (e.g. two-option comparisons, key terms).
- **Decoration:** faint pastel gradient blobs/circles tucked in the top-right and bottom-left corners of slides for visual texture — subtle, never distracting.
- **Typography:** clean sans-serif (e.g. Poppins, Montserrat, or similar geometric sans), titles bold and spaced out, body text medium weight, generous line spacing.
- **Layout:** lots of white space, centered or two-column content, short punchy phrases rather than dense paragraphs. Section/divider slides sometimes flip to a solid bold-blue background with white text for contrast.
- **Tone of content:** concise, exam/revision-friendly — definitions, short bullet explanations, comparisons (e.g. "X vs Y" with two cards side by side).

## Build workflow
1. Use the `pptx` skill (`/mnt/skills/public/pptx/SKILL.md`) to actually generate the .pptx file — follow its instructions for file structure.
2. Apply the palette and layout rules above to every slide master/layout you create.
3. Structure content as: Title slide → Agenda/Plan → one concept per slide (definition + short explanation, sometimes two accent cards for comparisons) → optional section-divider slides on navy background → Summary/Recap slide.
4. Keep text per slide short (under ~40 words); let the design (cards, spacing) carry the visual weight rather than dense text.
5. If the user gives a course topic and subpoints, split them into one slide per subpoint, in the order given.

## Notes
- This style was reverse-engineered from a reference example the user shared; treat the palette/typography as defaults, not rules — adjust if the user names a different color scheme.
- Always pair this skill with the `pptx` skill for actual file generation; this file only defines the visual/content style.
