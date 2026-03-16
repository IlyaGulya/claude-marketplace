---
name: frontend-design-v2
description: Build frontend interfaces with externally randomized style direction. Use when the user asks to build web components, pages, or applications with high design diversity. Produces distinctive, production-grade UI that avoids repetitive AI aesthetics.
---

This skill creates distinctive, production-grade frontend interfaces by combining externally randomized aesthetic direction with high-quality implementation.

The user provides frontend requirements: a component, page, application, or interface to build. They may include context about the purpose, audience, or technical constraints.

## Phase 1: Style Direction

FIRST: Launch the `style-director-v2` subagent to generate 4 style direction cards with externally randomized constraints (palette, typography, density, motion energy).

Use this exact prompt for the subagent:
> Brief: $ARGUMENTS

Present all 4 style cards to the user. Ask which card they want to use for the implementation. If the user says "pick for me" or "any" or similar, choose the card that best fits their stated requirements.

Do NOT proceed to Phase 2 until the user has selected a card.

## Phase 2: Implementation

Using the selected style card as your aesthetic foundation, implement the user's frontend requirements as working code (HTML/CSS/JS, React, Vue, etc.).

### Constraints from the style card are MANDATORY:

- **Typography**: Use the card's assigned display and body fonts exactly. Load via Google Fonts, CDN, or @font-face. Apply the card's recommended weights and sizes. No substitutions.
- **Palette**: Use the card's assigned hex colors as CSS variables. You may derive hover/border/shadow shades by adjusting lightness, but the 5 core colors (background, surface, text, accent, muted) are fixed.
- **Density**: Follow the card's density level for all spacing and layout decisions:
  - sparse: generous whitespace, few elements per viewport, content floats in space
  - balanced: standard spacing, clear hierarchy, comfortable rhythm
  - dense: maximum information per viewport, tight spacing, no wasted space
- **Motion energy**: Follow the card's motion level for all animation timing and complexity:
  - minimal: near-static, subtle transitions under 200ms, mostly opacity/color shifts
  - moderate: purposeful motion, 200-500ms, transforms and reveals
  - energetic: expressive motion, 400ms+, physics-based springs, multi-stage animations
- **Concept, layout motifs, motion motifs, signature detail**: Use these as creative direction. They define the personality of the interface.

### Implementation standards

Produce code that is:
- Production-grade and functional
- Faithful to the selected style card's vision
- Meticulously refined in every detail

**Typography**: Pair the card's display and body fonts intentionally. Display font for headings, hero text, and emphasis. Body font for paragraphs, labels, and UI text. Choose weights that create clear hierarchy.

**Color & Theme**: Build the full theme from the card's 5-color palette using CSS variables. Map background, surface, text, accent, and muted to semantic roles. Derive additional states (hover, active, disabled, focus) by adjusting lightness of existing colors. The accent color should drive focal points and key interactions.

**Motion**: Implement the card's specific motion motifs. The hero interaction described in the card should be the centerpiece. Prioritize CSS-only solutions for HTML. Use Motion library for React when available. Focus on high-impact moments: one well-orchestrated page load with staggered reveals creates more delight than scattered micro-interactions.

**Spatial Composition**: Follow the card's layout motifs and density. Create unexpected layouts with asymmetry, overlap, diagonal flow, or grid-breaking elements as the card directs. Generous negative space OR controlled density, depending on the card.

**Backgrounds & Visual Details**: Build atmosphere through the card's signature detail. Apply textures, patterns, borders, shadows, gradients, or effects that match the card's creative direction. Create depth rather than flat surfaces.

**IMPORTANT**: Match implementation complexity to the style card's vision. A card with energetic motion and dense layout needs elaborate code with extensive animations and effects. A card with minimal motion and sparse density needs restraint, precision, and careful attention to spacing, typography, and subtle details.

The style card provides the aesthetic DNA. Your job is to faithfully express that DNA in working code that solves the user's actual requirements.
