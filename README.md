# Magic 8 Ball — Oracle Deck

A single-file Magic 8 Ball web app with two full themes, a 3D disc, two canvas particle systems, procedural audio, and a reveal choreography built entirely with the Web Animations API.

This repository doubles as a capability test for **Qwen 3.8 Flash Next (Gyro S)** running on a **12 GB GPU + 32 GB DDR5** machine: the whole app was built in a near-one-shot session driven by [STRATA](https://github.com/Niko1221/Strata) — no framework, no build step, no dependencies beyond one Google Fonts link.

## Run it

Open `magic-8-ball.html` in any modern browser (Chrome/Edge recommended for View Transitions and `@property`). No install, no server. Ask a yes/no question, then tap the ball, press the CTA, or press Space/Enter.

Files in this repo:

| File | Role |
|---|---|
| `magic-8-ball.html` | The entire app — inline CSS, markup, JS (~940 lines) |
| `PROMPT.MD` | Original build prompt |
| `PROMPT_V2.MD` | Current spec, synced to the shipped app |
| `README.md` | This breakdown |
| `PROGRESS.md` | Session progress tracker (local only, gitignored) |

## Architecture

One HTML file, four layers:

1. **Theme boot script** (in `<head>`, before paint) — reads `m8_theme` from localStorage (try/catch) and sets `html[data-theme="coffee"]` so there is no theme flash.
2. **CSS token system** — `@property --hue` typed custom property with `transition: --hue 1.5s` on `:root`; every accent (border, glow, text, bars, triangle) derives from `hsl(var(--hue) …)`, so `setHue()` re-tints the entire UI per answer with one line. `--ball: clamp(340px, 64vmin, 640px)` sizes the sphere. Coffee overrides are plain `html[data-theme="coffee"]` selectors.
3. **Markup** — header (brand, theme tabs, sound toggle), masthead (Coffee-only editorial intro), `.stage` (ball button + verdict chip), `.controls` (question input + CTA + status), two `<aside>` panels (history, tally), a visually-hidden `aria-live` announcement paragraph.
4. **JS modules** (inline `<script>`): `THEMES` config, state (`busy`, `state`, `lastType`, `agitation`), `pickAnswer()` (20 classic answers, no repeat), trigger flow, WAAPI choreography, two canvas loops, tilt/parallax loop, WebAudio synth, theme switching with View Transitions, history/tally renderers.

## The trigger choreography

`trigger()` is a small state machine; triggers are ball click, CTA click, Space/Enter (not while typing), and devicemotion shake (>35 magnitude). Re-triggers are ignored while `busy` (900 ms busy window).

Sequence (non-reduced-motion):

1. `agitation = 1`, `shakeSound()` (rattle burst + low-pass liquid slosh, LFO wobble).
2. `clearFace()` — both answer spans blanked, triangle faded out (~140 ms), so the previous answer never ghosts.
3. `flipDisc()` — the disc rotates **at shake start**, not at reveal. `discDeg` parity: first shake is a 180° flip (~480 ms); every later shake is a full 360° spin (~680 ms) that flashes the "8" face mid-spin and lands back on the liquid side. Inline `#disc` transform is committed to `rotateY(180deg)` when the animation finishes — no fill-forwards stacking, face opacity animations cancelled before each spin.
4. `shakeAnim()` — WAAPI on `#ball`: squash start frame, ~14 decaying random translate/rotate/scale keyframes, settle to none (~650 ms). The `.back` face gets a separate jiggle with `composite: 'add'` so it rides on top of the flip spin's transform instead of clobbering it.
5. ~120 ms in: `shockPulse()` — two staggered shock rings expand, glow pulse (Energy only).
6. At 620 ms: `reveal()` — pick answer, measure, set `--fit`, triangle fades in (~300 ms delay), answer lines stagger in (~450 + 180 ms each, `rotateX(-24deg)` entrance), window pops forward (`translateZ` 18→46→18), verdict label pops, hue shifts to the answer type, history/tally update with pop + bar glow, `chimeSound()`.
7. At reveal + 900 ms: `busy = false`, CTA becomes "Shake again"/"Consult again".

Reduced-motion: shake and disc spin are skipped; disc flips instantly (inline `rotateY(180deg)` + front/back opacity crossfade), reveal is opacity-only fades after ~200 ms, all decorative CSS animations are disabled by a blanket `animation: none !important` rule, and every WAAPI decoration is guarded by `reduceMotion.matches`.

## The ball (3D stack)

`perspective: 650px` on `.floaty` → `#ball` (`preserve-3d`, radial-gradient glossy navy or matte espresso sphere, inset shadows) → layered children:

- **`.sheen`** `translateZ(10px)` — pointer-tracked specular highlight; `--sx/--sy` CSS vars updated in `tiltTick()` (Energy only; Coffee hides it).
- **`.rim`** — conic light arc masked to the sphere's edge band (`radial-gradient` mask at 89–91%), spinning `rimspin` 9s.
- **`.window`** `translateZ(18px)` — the dark inset window; pops to 46px on reveal.
- **`#disc`** `preserve-3d`, `inset: 12%` — two `backface-visibility: hidden` faces:
  - `.front`: white disc, Georgia "8", idle `discBob`;
  - `.back`: dark liquid window containing `canvas#liquid`, the SVG triangle (`M50 10 L88 84 L12 84 Z`, hidden until reveal), and the two answer spans.
- **`.aura`** behind the ball (`z-index: -1`): glow pulse (Energy), two hairline rings + orbiting dot (Coffee), and two tilted satellite orbit rings (`rotateX(74deg)`/`rotateX(66deg)`, counter-spinning).
- **`.floor`**: blurred soft shadow under the ball.
- Pointer parallax: targets `(x/innerWidth−0.5)×26°` / `−(y/innerHeight−0.5)×20°`, lerped 0.1 per frame onto `#ball`; suppressed while shaking.

### Answer auto-fit (`--fit`)

Both lines must sit inside the slanted triangle strokes. `reveal()` runs ~4 passes: estimate the two line centers at ~58% of disc height ± the current line-height/gap span; measure each line with `measureText` on the liquid canvas at the theme font/size; constrain each line to the stroke width at its **top edge** (`boxW × 76 × (y−10)/74 / 100 − 12`); clamp ≥ 0.35. The top-edge constraint (not the center) is what keeps lines from crossing the slanted strokes.

### Liquid canvas

`canvas#liquid` draws, per frame, a radial glow wash tinted by `curHue` (alpha rises with agitation), six swirling arcs whose amplitude/lineWidth/alpha scale with agitation, orbiting motes that speed up while agitated, and bubbles spawned while agitation > 0.25 that rise and fade (capped at 46). `agitation` is set to 1 on trigger and decays ×0.94 per frame; drawing is gated until first reveal or agitation, and the loop pauses when the tab is hidden. Canvases are sized at `devicePixelRatio` (capped at 2).

## FX canvas (full-screen backdrop)

`canvas#fx`, one loop for both themes:

- **Energy** — 90 twinkling stars with cross-flare strokes on bright ones, a shooting comet every ~5–12 s with a gradient tail and head dot, and reveal spark bursts drawn with `globalCompositeOperation = 'lighter'`. Colors hoisted, per-particle alpha via `globalAlpha` (no per-particle color strings).
- **Coffee** — 40 slow drifting gold dust motes, five sway wisps of steam rising above the ball (`seedSteam()`, reseeded on resize), fine gold spark streaks on reveal.

Plus Energy-only DOM background layers: three blurred drifting orbs, SVG-turbulence film grain, vignette. Coffee shows none of them.

## Themes

`THEMES` config holds idle hue, `meta theme-color`, per-type hues/labels, all UI copy, and liquid/mote color functions. Switching uses `document.startViewTransition()` with a circular `clip-path` wipe emanating from the clicked tab (`--vt-x/--vt-y`); reduced-motion or unsupported browsers fall back to direct apply. `applyTheme()` preserves the current answer, history, and tally while rebuilding copy/colors.

| | Energy Drink (default) | Coffee |
|---|---|---|
| Look | Dark navy neon glass: orbs, grain, vignette, glowing pills | Editorial ivory: Cormorant Garamond, champagne hairlines, flat surfaces |
| Sphere | Glossy black-navy, specular sheen, conic rim light | Matte espresso, gold bezel hairline (`inset: 3%`), gold rim tint |
| Hues | idle 224; yes 162 / maybe 268 / no 342 | idle 38; yes 42 / maybe 205 / no 8 |
| Labels | Affirmative / Non-committal / Negative | Confirmed / Undetermined / Declined |
| Brand / CTA | "Magic 8 Ball" / "Shake the ball" | "Maison Oracle" / "Consult the Oracle" |
| Panels | Navy glass `blur(8px)` | Cream `blur(8px)`, gold top hairline, bold serif headings |
| Masthead | hidden | "Maison de Café · Est. 1946 — The Oracle" with date |
| Sound | noise rattle + slosh + two-note chime | soft sine bell pair |

## Audio

WebAudio only, `AudioContext` created lazily on first gesture and resumed if suspended. Header toggle (on/off icons, `aria-pressed`) persisted in `m8_sound`; master gain 0.22. Energy shake: band-passed white-noise rattle + low-passed noise slosh modulated by a 7 Hz LFO; reveal chime is triangle+sine at 620/930 Hz. Coffee shake is a 1100/1650 Hz sine bell; same chime quieter.

## Layout & responsiveness

- `main` is a centered flex column; `.main-inner` centers ball, verdict, controls, panels below the header at narrow widths.
- ≥1100 px: panels become fixed left/right beside a centered `main-inner` (max-width 760px); theme tabs sit directly left of the sound toggle (`justify-content: flex-start` + `.tabs { margin-left: auto }`). Below that the header keeps its centered flex layout and `main` gets `padding-top: 100px`.
- All non-ball UI sizes use `clamp()` so they grow on wide screens and pin to base values on narrow ones (brand 16–19 px, mast title 44–76 px, question 15–17 px, CTA 14–16 px, panels 250–340 px / Coffee 230–320 px, coffee root 18–21 px). Ball is `clamp(340px, 64vmin, 640px)` so 375 px screens never overflow — verified with `scrollWidth` checks at 375 px in both themes.

## Accessibility

Keyboard triggers with visible `:focus-visible` rings, `aria-pressed` on theme and sound toggles, `aria-label`s on ball/sound, sr-only question label, `aria-live` status paragraph announcing every answer, decorative layers `aria-hidden`, tap targets ≥ 44 px, `prefers-reduced-motion` fully honored (see choreography section).

## Validation

Headless verification ran against Microsoft Edge via `puppeteer-core` (harness lives outside the repo, in a temp dir):

- idle / shake / reveal screenshots in both themes, console + pageerror watched — zero errors throughout;
- 375 px and 1440 px layouts, horizontal-overflow check in both themes;
- reduced-motion pass: zero rim/floaty animations, disc flip via inline style;
- second-trigger pass: face cleared mid-spin, spin commit `rotateY(180deg)`, triangle + answer visible after reveal.

## Build provenance

The session ran on STRATA with Qwen 3.8 Flash Next (Gyro S) on a 12 GB GPU / 32 GB DDR5 machine. The app went: near-one-shot full build → verification fixes (aura placement, theme-aware verdict labels) → a four-track code review (security, business, perf, dead code) with all findings applied → beautification pass (3D depth stack, rim/satellites, specular, staggered entrances, canvas upgrades) → reveal/flow redesign (flip at shake start, liquid-exposed shake, expressive fluid) → wide-screen `clamp()` scaling and header layout → spec sync into `PROMPT_V2.MD`. `PROGRESS.md` records each step; progress tracking never committed anything.

## Known caveats

- Google Fonts (Cormorant Garamond) needs network; offline falls back to Garamond/Georgia serif stacks.
- `devicemotion` shake requires a secure context and permission on iOS.
- View Transition wipe needs Chrome ≥ 111; other browsers switch instantly.
- The 900 ms busy window means rapid repeat clicks are ignored by design.
