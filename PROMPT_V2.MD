# Magic 8 Ball — build spec v2

Build a single-file, visually stunning Magic 8 Ball web app as it currently stands: one `magic-8-ball.html` with inline CSS and JS, no framework, no build step, no dependencies except one optional Google Fonts link. Two switchable themes, "Energy Drink" (default) and "Coffee". When done, load it in a browser, check the console for errors, and screenshot idle, shake and reveal states in both themes.

## Core behavior
- Optional question input; triggers: click the ball, click the CTA, Space/Enter (unless focus is in the input or on a button), or devicemotion shake. Each triggers a shake then a reveal.
- The 20 classic answers stored as `[type, 'line one|line break']` (type yes/maybe/no; `|` marks the wrap inside the triangle). Rebalance breaks so the longer line lands lower where the triangle widens: `Signs|point to yes`, `You may|rely on it`, `Better not|tell you now`, `Cannot|predict now`, `Concentrate|and ask again`, `Don't|count on it`, `Outlook|not so good`, `My sources|say no`, `Signs|point to no`, `Reply hazy|try again`, `Ask again|later`, `My reply|is no`, `Most likely|`, `Outlook good|`. Avoid repeating the previous answer.
- Ignore triggers while busy.
- Verdict chip under the ball shows the type label; one CTA shows ready / busy / "again".
- History panel (recent answers, colored dot per type) and tally panel (three bars with counts) update on every reveal.
- aria-live paragraph announces the answer. `prefers-reduced-motion`: skip the shake, flip the disc instantly (inline `#disc` `rotateY(180deg)`, front/back opacity crossfade) and reveal after ~200ms as fades; decorative CSS animations are disabled via a blanket `animation: none !important` on animated layers.
- Persist `m8_sound` and `m8_theme` in localStorage; wrap every access in try/catch.

## Layout
- Fixed full-width header on BOTH themes, frosted with `backdrop-filter: blur(8px)`: Energy `hsl(224 45% 7% / .55)` over a hue hairline; Coffee translucent cream `hsl(40 34% 92% / .65)` over a `#b08d57` hairline, so scrolled content blurs behind it. Theme tabs sit left of the sound toggle at ≥1100px (`header { justify-content: flex-start }` + `.tabs { margin-left: auto }`); centered flex layout below that.
- `main` is a centered flex column; `.main-inner` is also `display:flex; flex-direction:column; align-items:center` so ball, verdict, controls and panels are centered — never left-aligned — below the header at narrow widths. `main` padding-top 100px below 1100px. Non-ball UI sizes use `clamp()` so they grow on wide viewports and pin to base values on narrow ones: brand 16–19px, mast-title 44–76px, question font 15–17px, CTA 14–16px, status/verdict 13–15px, h2/t-row 12–14px, history 13–15px.
- Panels `width: clamp(250px, 26vw, 340px)` (Coffee `clamp(230px, 24vw, 320px)`): fixed left/right at ≥1100px beside a centered `main-inner` (max-width 760px); stacked below at narrower widths, with `.panel.history { margin-bottom: 14px }` as the gap between history and tally in both themes.

## The ball
- `--ball: clamp(340px, 64vmin, 640px)`; glossy black sphere, round window `inset: 24%` (both themes), white disc with a big "8" at `inset: 12%`.
- Flip happens at the START of the shake, not at reveal (`#disc` `preserve-3d`, faces `backface-visibility: hidden`; the back face is the dark liquid window with the SVG triangle and answer inside it). `trigger()` clears the face first (`clearFace()`: blank both answer spans, fade the triangle out ~140ms) so the swirling liquid is exposed during the shake. `flipDisc()` tracks `discDeg`: first shake flips `rotateY(0→180deg)` (~480ms), later shakes do a full 360° spin (~680ms, front "8" flashes mid-spin); inline `#disc` transform is committed to `rotateY(180deg)` when the animation finishes (no fill-forwards stacking — cancel any prior face opacity animations before each spin).
- Triangle: `viewBox="0 0 100 100"`, path `M50 10 L88 84 L12 84 Z`, `inset: 2%` of the disc, `opacity: 0` by default. Apex fills most of the upper room; base corners stay inside the disc circle (corner radius ≈ 0.49× disc radius — never clip the circle). Stroke width 2 (Energy), 1.6 gold (Coffee). It fades in at reveal (WAAPI opacity 0→1, ~300ms delay, 450ms; under reduced-motion ~200ms no delay) after the trigger fade-out.
- Answer: two spans set at reveal, animated staggered (~450ms delay + 180ms each), entering with `rotateX(-24deg) translateY(10px)` → none (opacity-only under reduced-motion). `.answer` centered with `padding-top: 16%`, `gap: .35em`; base font `--ball × .052` (Energy, 600) or `--ball × .05` (Coffee, italic 500 Cormorant). Verdict chip is centered (`justify-content: center; padding: 12px 0`) with a label pop.
- Auto-fit `--fit`: both lines must sit entirely inside the triangle strokes. Iterate ~4 passes: estimate line centers at ~58% of disc height ± the current line-height/gap span; measure line widths with `measureText` on the liquid canvas ctx using the theme's font string at base size; constrain each line to the stroke width at the line's TOP edge (`boxW × 76 × (y−10)/74 / 100 − 12`). Clamp ≥ 0.35. The top-edge constraint (not center) is what stops lines crossing the slanted strokes.
- Shake: WAAPI on `#ball` — squash start frame (`scale(.9) translateY(10px)`), ~14 decaying random translate/rotate/scale keyframes, settle to none (~650ms ease-out); the `.back` face jiggles with a separate WAAPI animation using `composite: 'add'` so it adds onto the face's base `rotateY(180deg)` without breaking the spin; pointerdown squashes the ball/CTA; two shock rings pulse outward (`div.shock` + `.shock.s2`, staggered); glow pulse (Energy only). No "8" face-spin animations — the liquid side is what's visible during the shake.
- Idle: gentle float, disc bob on the front "8" face (`discBob`), plus pointer parallax — `perspective: 650px` on `.floaty`; pointermove sets targets `(x/innerWidth−0.5)×26°` rotateY and `−(y/innerHeight−0.5)×20°` rotateX, smoothed toward targets in a rAF loop (0.1 easing) applied to `#ball`. 3D depth stack: `#ball` `preserve-3d`, `.window` `translateZ(18px)` (pops to 46px then settles at reveal), `.sheen` `translateZ(10px)` pointer-tracked specular via `--sx/--sy` CSS vars set in the tilt loop. Skip when reduced-motion or shaking.
- `canvas#liquid`: radial glow wash tinted by `curHue`, six swirling arcs (amplitude/lineWidth/alpha scale with agitation), motes orbiting faster while agitated, bubbles spawned while agitation > 0.25 rising and fading; agitation = 1 on trigger, decays ×0.94 per frame. Draw only when revealed at least once or agitating. Soft floor shadow; aura behind the ball with a conic rim light arc masked to the sphere edge (`--rim` spin animation) and two tilted orbit rings with satellites (`--orbit`/`--orbit2`).

## Motion polish
- Decorative WAAPI micro-animations everywhere are guarded with `reduceMotion.matches`; decorative CSS animations are disabled under reduced motion via the blanket rule. Masthead children use a staggered `rise` keyframe (Coffee); the brand mark pulses (`markPulse`); the CTA has a shine sweep (`::after`, `--shine`).
- History items animate in with a staggered `li.anim` class (first ~6); tally count pops (scale 1.4→1) and the matching bar glows (`boxShadow` + `brightness`) on the answer's type.

## Color system
- `@property --hue { syntax:'<number>'; inherits:true; initial-value:224 }`, `transition: --hue 1.5s` on `:root`; all accents derive from `hsl(var(--hue) …)`; `setHue()` re-tints the UI per answer.

## Themes
`THEMES` config: idle hue, meta theme-color, per-type hues/labels, all UI copy, `liq(h,i,a)`/`mote(h,a)` liquid color functions. Coffee overrides under `html[data-theme="coffee"]`; Energy is default. Inline `<head>` script applies stored theme before paint. Switch via View Transitions (circular clip-path wipe from the clicked tab), direct-apply fallback under reduced-motion; `applyTheme()` preserves answer/history/tally and rebuilds copy/colors. History text uses theme colors — Coffee items `#1d1510`, Coffee empty-state `#8c6d3b` (never Energy's light text).

### Energy Drink (default)
- Dark navy neon glass: drifting blurred orbs, film grain, vignette; fullscreen `#fx` canvas with twinkling stars (cross-flares on bright large ones), shooting comets with gradient trails + head dots, and reveal spark bursts drawn with `globalCompositeOperation = 'lighter'` (hoisted fillStyle + per-particle `globalAlpha`, no per-particle color strings).
- Pill input with glowing focus ring; glossy sheen on sphere; glowing glass CTA `blur(10px)`; glass panels `blur(8px)` translucent navy.
- Idle hue 224; yes 162 / maybe 268 / no 342; labels Affirmative / Non-committal / Negative. Brand "Magic 8 Ball", CTA "Shake the ball", busy "Consulting the cosmos…".
- Sound: rattle burst + liquid slosh, then a two-note chime.

### Coffee (editorial luxury)
- Ivory page #f4efe6, flat radial background, no orbs/grain/vignette. Espresso ink #1d1510, champagne gold #b08d57 hairlines, #8c6d3b gold text. Cormorant Garamond (Google Fonts, weights include 700), root font `clamp(18px, 1.3vw, 21px)`, tracked caps; panel headings bold 700 (`clamp(12px, 1vw, 14px)`), status/verdict inherit the ball face size.
- Frosted surfaces below the header (matching Energy's blur): CTA translucent cream `hsl(40 34% 92% / .55)` with `blur(10px)`; panels same cream with `blur(8px)`, 12px radius, gold top hairline. CTA keeps espresso hover sweep (`::before` scaleY); hover fill paints over the translucent background, under the label.
- Matte espresso sphere (radial-gradient #3d2c1d → #140d08), no sheen/glow; inset gold hairline bezel (`inset: 3%`) shown only in Coffee via CSS (`html:not([data-theme="coffee"]) .bezel { display: none }`). Ivory "8" disc with light serif numeral; dark espresso window; gold-outline triangle; ivory italic serif answer.
- Two fine gold aura rings + slow orbiting gold dot (`inset: 18%`); generous stage margin. Verdict: hairline-flanked text in the answer hue. Input: borderless underline, italic centered, "Pose your question…".
- Masthead (hidden in Energy): kicker "Maison de Café · Est. 1946", title "The Oracle", hairline rule with diamond, today's date.
- Tally: thick bars — 5px champagne track `#e3d3b4`, fill in the answer-type hue at readable darkness against cream. Titles "Recent Consultations" / "The Ledger"; unanswered label "Unspoken"; empty history "No consultations yet.".
- Idle hue 38; yes 42 / maybe 205 / no 8; labels Confirmed / Undetermined / Declined. Brand "Maison Oracle", CTA "Consult the Oracle", busy "Consulting…", again "Consult again".
- FX: slow drifting gold dust, sway wisps of steam rising above the ball (`seedSteam()`), fine gold sparks; liquid champagne `rgba(201,168,112,a)`. Sound: soft glass sine bell.

## Audio
WebAudio only, AudioContext lazily on first gesture; header toggle with on/off icons, default on, modest volume.

## Quality bar
- Responsive from 375px (minimum ball 340px fits the content area) with no horizontal scroll; tap targets ≥ 44px.
- Keyboard accessible, visible focus, `aria-pressed` on theme and sound toggles.
- Pause canvas loops when the tab is hidden; canvases sized for devicePixelRatio.
- No console errors; tidy code, sparse comments.
