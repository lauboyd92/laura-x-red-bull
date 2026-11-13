# Handoff: port the Heaps Normal site's design system to the Kantoko site

Source of truth (read it, don't retype it): `/Users/lboyd/code/instagram-dashboard/laura-site/index.html` (single file, vanilla HTML/CSS/JS, no dependencies, ~550 lines) and `/Users/lboyd/code/instagram-dashboard/laura-site/assets/`.
Serve it with `python3 -m http.server 4173 --directory laura-site` (launch.json entry `laura-site`) to see every effect running. Line numbers below are approximate; search for the class or variable name.

The brief: a black-and-white, Helvetica, scroll-driven "story" that works as a hypothetical job application. Her voice, photos, handwritten annotations, interactive moments that break out of the vertical flow. Keep the same structure and effects for Kantoko, swapping the copy and any Heaps Normal-specific assets (list at the bottom).

---

## 0. Fastest route

Copy `laura-site/index.html` and `laura-site/assets/` into the Kantoko project as the base, then rewrite only the content block (`var phrases`, `var jobs`, `var S = [...]`, `S.push(...)`) and the header text. All the engine, CSS and effects are reusable as is. If the Kantoko site already has its own structure, port the features below one at a time, in the order listed.

## 1. Global design rules

- Palette: `--ink:#000; --paper:#fff; --mute:#8a8a8a; --wash:#f3f3f3`. Black and white only. The one deliberate exception is the Canva logo (a coloured brand logo). Use hand-drawn **black** ticks, never the colour ✅ emoji.
- Type: Helvetica Neue / Helvetica / Arial, tight tracking (-0.03 to -0.04em), bold headlines. Handwriting font is Caveat 600 via Google Fonts (`<link href="https://fonts.googleapis.com/css2?family=Caveat:wght@600&display=swap">`).
- Type scale (reduced from v1 after "too much big text" feedback): `.huge clamp(44px,9vw,160px)`, `.big clamp(32px,5.6vw,100px)`, `.mid clamp(24px,3.6vw,62px)`, `.small clamp(17px,1.9vw,28px)`. Use `.mute` (grey) for lead-in lines and one black headline phrase per beat. Never more than a few big words competing with each other.
- Copy principle: short paragraphs, one idea per beat, her voice (dry, self-deprecating, Australian). Merge lines into beats rather than one line per scroll. The Heaps Normal site ended at 32 beats.
- Header: fixed top-left `Laura Boyd`, top-right `→ <Company>`. Scroll progress bar at the top. A "Start scrolling now ↓" pill that disappears after 30px of scroll.
- Cursor: a custom hand-drawn can follows the mouse (`#can`, hidden on touch). Heaps Normal-specific; for Kantoko use a Kantoko-relevant sketched object, or drop it.

## 2. The scroll engine ("drum"), the core of everything

Search `/* ---------- scroll mapping ---------- */`, `layout()`, `scrollToPos()`, `render()`, `onScroll()`.

- All beats are absolutely positioned inside a fixed full-screen `#stage` with `perspective:1600px`. A tall empty `#track` div provides the scroll height.
- Each beat's height is measured (`hs[]`, `ctr[]`). `pos` is lerped (`.14` per frame) toward `target`, which is derived from `scrollY` via cumulative per-beat weights (`cum[]`, `unit = innerHeight*.4`, so a normal beat takes 0.4 of a screen of scrolling).
- Only 2 beats before and 2 after are visible (`VISIBLE = 2`), fading: opacity 1 / .28 / .08 at distance 0 / 1 / 2, with a `rotateX(-d*16deg)` tilt and slight scale. This was her explicit ask: you can see what came before (faded) and what's next, but never more than two each side.
- `.step.on` is set on the beat nearest the centre. Videos play only while their beat is `on`.
- **Hold mechanic:** a beat can set `hold:.86` (fraction of its scroll length) and `w:` (weight, i.e. how many "units" long). While `scroll progress < hold`, the drum stays on that beat while its internal animation plays (bubbles popping, cards sliding, plane flying, Venn circles lighting). The drum only advances in the final `1-hold` part.
- `layout()` re-measures on resize, on `fonts.ready` and on load. If a beat is taller than the viewport it clips: check every beat at 1280×800 and at phone size (`over` check: `[...document.querySelectorAll('.step')].filter(e=>e.offsetHeight>innerHeight*.97)`).

## 3. Beat types (the `S` array)

A beat is an object. Text helper: `T(cls, html)` returns `<p class="cls">html</p>`.

| Shape | What it renders |
|---|---|
| `{ t: T(..)+T(..) }` | Text only. Alternating sides (see 4). |
| `{ t, img:[{...}] }` | Text left, one image right (stacks on mobile). Image entry kinds: `{src, alt, credit?, plain?}` photo/art; `{vid}` muted looping video; `{embed:url}` lazy iframe (Instagram `/embed`); `{tiktok:url, id}` official TikTok embed; `{html}` arbitrary HTML/SVG (used for all the custom visuals below); `{l:"label"}` grey dashed placeholder. |
| `{ stack:1, t, img }` | Image below the text (used for the job-description screenshot). |
| `{ photo:{src, ar, rot, alt, notes}, t? }` | Annotated photo with handwritten notes and arrows (section 6). |
| `{ bubbles:[...], hold, w, t }` | Speech-bubble pop-up beat (section 5). |
| `{ cards:[...], hold, w }` | Sideways card track (section 7). |
| `{ venn:{lead, items, end}, hold, w }` | Venn diagram (section 8). |
| `{ plane:1, hold, w, t }` | Plane with banner (section 9). |
| `{ last:1, w:1.3 }` | Closing beat, kept in view at the end. |

## 4. Alternating left/right paragraphs

For text-only beats the builder sets `el.dataset.side` to `l` / `r` alternately (`window._side` toggle). CSS: `.step[data-side] .txt{max-width:68%}`, right beats get `margin-left:auto; text-align:right`, and each text block starts offset `translateX(±5vw)` and eases to 0 when the beat gets `.on` (`.9s cubic-bezier(.2,.7,.2,1)`). Full width under 760px. This was added so paragraphs "take turns" and don't read as a wall of text.

## 5. Speech bubbles (a "remember when?" list that pops out of the flow)

Search `bubbles`, `.bub`, `.bubarea`. A heading line plus N rounded bubbles with a tail, scattered with fixed positions/rotations (`POS` array of `[x%, y%, rotation]`; `MPOS` for mobile in one column). Every third bubble is dark (`.dark`). Bubbles get `.in` one by one as scroll progresses through the hold (`c = floor(progress*n)+1`). `hold:.86, w:5.4` for 8 bubbles.

## 6. Annotated photos (handwritten note + hand-drawn arrow over a photo)

Search `.photo`, `.note`, `.arr`, `ARR(`, `NOTE(`. Helpers:
- `ARR(pos, cls)` is an SVG hand-drawn arrow; `cls` `"down"` rotates it 180°, `"dark"` makes it black. `pos` is inline CSS like `left:27%;top:76%`.
- `NOTE(pos, html, cls)` is a Caveat span; `--r` is rotation, `--fs` is font size in `cqw` (the figure is a container-query box so notes scale with the photo).
- Default white text with a soft dark shadow. Use `"dark"` (black text and arrow) when the photo's background is mostly white (e.g. the snow photo). For dark-text photos, put the text beside the photo via the `t` property instead.
- Photos are `ar:"3/4"`, with a slight `rot` (±1.2–1.4deg) for a pinned-up feel.
Examples used: cats (two notes, two arrows up at each cat), knitting, Canva LA (note + arrow pointing down at her), snow (black ink), Are Media ("that's me, working for the mags!").

## 7. Sideways card track ("my job is so much more than…")

Search `cards`, `.hs`, `.hs-card`. Vertical scroll translates a horizontal row of cards (`translate3d(-p*max,0,0)`). Cards alternate white/black (`dark`), each card tilts and drops by distance from the centre (`rotate(-d*5deg) translateY(|d|*16px)`), with a progress bar and an `01 / 05` counter. Cards are intentionally short and small (she asked for smaller, vertical-height focus, no video inside). First card is an intro card. `hold:.9`, `w:(jobs.length+1)*.7`.

## 8. Venn diagram (breadth of experience)

Search `venn`, `.vn`, `.vc`, `.vi`. Three overlapping circles labelled with a single word each (`c:`), a text column on the other side. As scroll progresses through the hold (4 phases: 0, 1, 2, then all three plus "me" in the middle), the matching circle lights up and its description appears; the last phase shows the closing line (`end`). Heaps Normal copy: Growth ("Experience in ads management across Meta and TikTok."), Strategy ("Creative strategy for omni-channel campaigns, including experiential, OOH and social components."), Management ("Campaign and project management across brand partnerships, sponsorships and launches."), end line "So it goes well beyond social. Social is where I started. It isn't where I stop." Keep this beat: it shows experience she can elaborate on. Adapt labels/descriptions to what Kantoko cares about.

## 9. Plane with banner (the "shooting my shot" moment)

Search `PLANE`, `.sky`, `.plane`, `st.plane`. A big text line, then a hand-sketched old-timey biplane flies left→right across the screen towing a banner ("I'm shooting my shot"). Implementation: inline SVG (`viewBox 0 0 1000 220`, stroke black 3.2, white fill, Caveat banner text), translated by scroll progress through the hold (`x = -w + p*(innerWidth+w)`, small sine bob). CSS-animated flutter (`flap` skewY), propeller flicker (`prop` scaleY) and bob. **Her cutout head (`assets/head.png`) is the pilot** (`<image href="assets/head.png" x=784 y=30 width=84 height=77>` clipped to the cockpit). `hold:.86, w:3.4`.

## 10. Smaller visual moments (each is an `img:[{html}]` entry plus a little CSS)

- **Draw-on underline:** `<span class='ul'>key words</span>`. A black bar animates under the words when the beat is `.on` (`background-size` 0 → 100%). Used on "taste and culture", "braver choice", "heads of brand".
- **Aldi receipt:** monospace uppercase receipt that "prints" top to bottom (`clip-path:inset(0 0 100% 0) → inset(0)` with `steps(16)`), rotated 2deg. Content: ALDI · LANE COVE / Cat food: lots / Wool: obviously / Youngest customer: me / Next youngest: +30 yrs / Thank you!
- **Passport stamps:** 5 bordered stamps (Theatre, New York, Japan, Magazines, Big tech), some rounded, each pops in with a scale 1.8 → 1 overshoot, staggered delays .15s–1.15s.
- **Strike-through tag + chips:** a "Social media lead" label gets a line drawn through it, then six pill chips pop in staggered (Team lead, Ads & growth, Creative strategy, Brand partnerships, Launches, Lifecycle), alternating black fill and outlined. This brings the "more than social" message early.
- **Headache beat:** `assets/drinks.png` (wine + beer illustration) with `assets/cloud.png` (storm cloud) above it. The cloud wobbles and rocks in a loop only while the beat is `.on` (`storm` keyframes, `mix-blend-mode:multiply` because the PNG has a white background).
- **Hand-drawn ticks:** `TICK` inline SVG instead of ✅.
- **Sketched bottle:** `BOTTLE` inline SVG (pop-cork and bubbles) on the closing beat in place of a photo; swap for a Kantoko-relevant sketch.
- **Embedded posts:** Instagram posts via `<iframe data-src=".../embed">`, lazily set when the beat first becomes visible. TikTok via the official `<blockquote class="tiktok-embed" cite=… data-video-id=…>` plus `https://www.tiktok.com/embed.js` loaded once on first reveal, plus a plain "Watch on TikTok ↗" link underneath as a fallback. TikTok height is capped at `min(60vh,700px)`.
- **Videos:** `<video muted loop playsinline preload="auto" poster="…-poster.jpg">`, 9:16, played only while on. Each has a poster JPG (generated with `qlmanage -t`), because without one a not-yet-loaded video shows as an empty white card (a bug she hit and asked to remove).
- **Brand logo inline:** the Canva wordmark (`assets/canva-logo.png`) replaces the big "Canva." text, `height:clamp(40px,4.6vw,64px)`.

## 11. Closing beat

Big question line, her email as a `mailto:` underlined link, a black pill "See my resume →" button linking to her Google Drive resume (`https://drive.google.com/file/d/10FUxJzKucYGdbOoMwQRbyeQX9GuFZta2/view?usp=sharing`), and the sketched bottle. Email: `lauboyd@gmail.com`. For Kantoko, re-write the question line and swap the drawing.

## 12. Assets (all in `laura-site/assets/`)

Reusable as-is (about Laura, not Heaps Normal): `head.png` (cut-out head for the plane), `cats.jpg`, `knitting.jpg`, `nyc-snow.jpg`, `are-media.jpg`, `canva-la.jpg`, `canva-video.mp4` + `canva-video-poster.jpg`, `canva-create.mp4` + `canva-create-poster.jpg`, `canva-logo.png` (real logo from her Canva Brand Kit via the Canva connector; only 200px wide, so keep it small or get an SVG), `smh.jpg` (Sydney Morning Herald Instagram screenshot), `drinks.png`, `cloud.png`, `telstra-tower.jpg` (credit line: "Art: Mister Trevor Dickinson"), `jd-taste.png` (screenshot of the *Heaps Normal* job description with "You have taste" highlighted: replace with a Kantoko equivalent if one exists).
Heaps Normal-specific (replace or drop): `can-cursor.png` (cursor), the High Hopes bottle, the "Heaps Normal can" beat, Instagram/TikTok embeds are hers and reusable if relevant: Squirrelites `https://www.instagram.com/p/DX6urZelKwm/`, Drag and Drop `https://www.instagram.com/p/DbbEF8iRDuO/`, TikTok `https://www.tiktok.com/@lauragoeslow/video/7448910114016267550`.

## 13. Heaps Normal-specific copy to rewrite for Kantoko

Beats mentioning Heaps Normal, "Head of Brand & Marketing", the DNA/ethos/headaches trio (beer and wine → needs a Kantoko equivalent or delete), the JD "taste" beat, the High Hopes closing line, the header `→ Heaps Normal`, and the Aldi/Lane Cove local-colour lines if they don't fit. Reusable story beats: Hi + capital-city line, "I'm Laura", the social-media-girlie turn, the "taste and culture" pair, the Canva beats, the job cards, the Venn, Squirrelites, Drag and Drop, Canva Create, TikTok → SMH, "I truly believe", knitter and cat mother, "Actually, don't answer that." with stamps, NYC/Are Media photos.

## 14. Gotchas learned the hard way

- Don't name a class `.tag`: it collides with the fixed header label. (`.lblwrap` is the chip/label wrapper.)
- `.art img` / `.art video` rules have specificity (0,1,1). Overriding inside a figure needs `.art .x img`, or the image will keep the box shadow and max-height.
- Hide-and-measure: beats with `visibility:hidden` have empty `innerText`, so scripts that scan by text only see the current beats.
- Embeds (Instagram/TikTok) stay blank in the app's preview pane and under `file://`; test in a real browser served over http.
- The browser preview pane is small, the emulated viewport is unreliable and scroll is smoothed (it overshoots). Scroll in steps with ~300ms waits, or test on a real screen.
- `qlmanage -t -s 720 -o <dir> file.mp4` makes poster frames; `sips -c H W --cropOffset Y X` crops; `sips -s format jpeg` converts (also reads webp/HEIC). No ffmpeg or PIL on this machine. `avconvert` compresses video (output was H.264, 720×1280).
- Cutting out the head: the source had a flat orange background. The cutout was done with a pure-Python flood fill from the image edges, keyed on high saturation and brightness (keeps skin tones, removes the orange) plus a 2-pass alpha feather. Orange tinge remains on the hair edge; a larger source photo would be cleaner.
- Private: don't quote budget numbers (she can't share them for work reasons). Lifecycle marketing for small businesses and publications like Elle and Marie Claire goes with the Are Media photo, not the Venn.

## 15. Verification checklist for the port

1. Scroll the whole page: every beat fits in the viewport at 1280×800 and at ~390×800; only ±2 neighbours show.
2. Plane, bubbles, cards and Venn each play out fully before the drum moves on.
3. Receipt prints, stamps pop, chips pop, underline draws, cloud wobbles only on its own beat.
4. Videos show a poster and play only when their beat is current; embeds load on a real browser.
5. Black and white throughout except the Canva logo; no console errors.
