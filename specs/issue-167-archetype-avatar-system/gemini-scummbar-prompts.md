# Gemini Prompts for Scummbar Collection

## Lessons from Previous Collections

**Problems encountered generating bauhaus, donut-creek, and neon:**

1. **Wrong IDs** — Gemini invents creative IDs (`prop:cyber-deck`, `fhair:anchor`) instead of using the exact IDs from the spec. **Fix:** provide literal `<symbol id="...">` skeletons in every prompt. When it has the exact XML tag to fill in, it follows the IDs.

2. **Scope drift** — Gemini adds categories we didn't ask for (hands, extra expressions, pirate props) and goes off-theme. **Fix:** one category per prompt, explicit count, no room for extras.

3. **Style drift** — within a session, Gemini gradually shifts style (started neon, drifted to pirate/fantasy). **Fix:** repeat the style description in EVERY prompt, not just the first one.

4. **Download limits** — Gemini's output window is small. Large SVG files get truncated or can't be downloaded. **Fix:** max ~20 symbols per prompt. Split into batches of 10-15.

5. **Wrong prefixes** — `eyebrow:` instead of `brow:`, `facial-hair:` instead of `beard:`, `accessory:` instead of `acc:`. **Fix:** give the exact prefix in the skeleton tags.

6. **Backgrounds/opacity** — early neon attempts had opaque background panels baked into parts. **Fix:** state "NO background rectangles" in every prompt.

7. **CSS classes instead of var()** — donut-creek cast preview used `class="sk"` instead of `var(--skin)`. **Fix:** state "use var(--skin) not CSS classes" in every prompt.

8. **Context loss** — Gemini forgets previous prompts. Each prompt must be self-contained with full style description and rules.

## Style Definition — Scummbar

```
Style: LucasArts adventure game pixel art (higher resolution)
Inspired by: Monkey Island, Day of the Tentacle, Maniac Mansion,
Sam & Max, Full Throttle, Grim Fandango, Discworld, Broken Sword

Rendering rules:
- ALL shapes built from <rect> elements ONLY — no <path>, no <circle>,
  no <ellipse>, no <polygon>. This is pixel art — everything is rectangles.
- Each "pixel" is a 6×5 rect (the viewBox is 200×240, grid is ~33×48)
- NO anti-aliasing, NO smooth curves, NO rounded corners (no rx/ry)
- NO <path> elements at all — pixel art doesn't have curves
- Limited colour steps — use var(--skin), var(--primary), etc. for main
  colours, plus ONE darker shade of each for pixel shading (e.g., a
  slightly offset rect in a darker tone to simulate shadow)
- Bold black 1-pixel outlines around major shapes (a rect border)
- Characters should feel like they belong in a 1990s point-and-click
  adventure game — chunky, expressive, recognisable from few pixels
```

## Prompt Template (copy into every prompt)

```
Continue generating pixel art avatar parts for the "scummbar" collection
(LucasArts adventure game style).

RULES — read these EVERY TIME:
1. ALL shapes are <rect> elements. NO <path>, <circle>, <ellipse>, <polygon>.
2. viewBox="0 0 200 240" on every <symbol>
3. Each "pixel" is approximately 6×5 units
4. Use var(--skin), var(--primary), var(--secondary), var(--accent),
   var(--hair-color) for colours. NO hardcoded colours except black (#000)
   for outlines and white (#fff) for highlights.
5. NO background rectangles filling the full viewBox
6. Transparent layers — only draw the part itself
7. Use the EXACT id values shown — do not rename them
8. Simple pixel shading: one darker rect offset by 1 "pixel" for depth
```

---

## Prompt 1 — Heads (12 symbols)

Generate 12 head symbols for a pixel art avatar collection. LucasArts adventure game style — all shapes are `<rect>` elements, no curves. Each "pixel" is ~6×5 units. Head centred around y=90. Use `var(--skin)` for fill, `#000` for 1-pixel outline rects. Include simple pixel eyes (2×2 rects) and a pixel nose/mouth.

```xml
<symbol id="head:round" viewBox="0 0 200 240"><!-- circle approximated as rounded pixel cluster --></symbol>
<symbol id="head:standard" viewBox="0 0 200 240"><!-- neutral rectangular head --></symbol>
<symbol id="head:soft-oval" viewBox="0 0 200 240"><!-- slightly wider pixel oval --></symbol>
<symbol id="head:oval" viewBox="0 0 200 240"><!-- tall narrow pixel head --></symbol>
<symbol id="head:square-jaw" viewBox="0 0 200 240"><!-- wide jaw, flat bottom --></symbol>
<symbol id="head:diamond" viewBox="0 0 200 240"><!-- narrow top/bottom, wide middle --></symbol>
<symbol id="head:heart" viewBox="0 0 200 240"><!-- wide top, narrow chin --></symbol>
<symbol id="head:angular" viewBox="0 0 200 240"><!-- sharp stepped edges --></symbol>
<symbol id="head:strong-sym" viewBox="0 0 200 240"><!-- large square + rounded top --></symbol>
<symbol id="head:weathered" viewBox="0 0 200 240"><!-- irregular pixel edges --></symbol>
<symbol id="head:round-wide" viewBox="0 0 200 240"><!-- wider circle cluster --></symbol>
<symbol id="head:fallback" viewBox="0 0 200 240"><!-- generic pixel head --></symbol>
```

12 symbols. EXACT ids. `<rect>` elements only. No `<path>`.

---

## Prompt 2 — Hair (15 symbols, batch 1 of 1)

Same scummbar pixel art style. All `<rect>` elements, no curves. Use `var(--hair-color)` for fill.

```xml
<symbol id="hair:bald-sides" viewBox="0 0 200 240"><!-- few pixel patches at temples --></symbol>
<symbol id="hair:buzz" viewBox="0 0 200 240"><!-- thin pixel cap on top of head --></symbol>
<symbol id="hair:afro-short" viewBox="0 0 200 240"><!-- blocky rounded volume --></symbol>
<symbol id="hair:long-flowing" viewBox="0 0 200 240"><!-- pixel curtain past shoulders --></symbol>
<symbol id="hair:messy-bun" viewBox="0 0 200 240"><!-- pixel knot on top --></symbol>
<symbol id="hair:mohawk" viewBox="0 0 200 240"><!-- tall pixel ridge --></symbol>
<symbol id="hair:wild-einstein" viewBox="0 0 200 240"><!-- pixel spikes radiating out --></symbol>
<symbol id="hair:slicked" viewBox="0 0 200 240"><!-- flat pixel cap swept back --></symbol>
<symbol id="hair:shoulder-wavy" viewBox="0 0 200 240"><!-- stepped pixel waves to shoulders --></symbol>
<symbol id="hair:cropped-fringe" viewBox="0 0 200 240"><!-- pixel fringe across forehead --></symbol>
<symbol id="hair:pixie" viewBox="0 0 200 240"><!-- short angular pixel crop --></symbol>
<symbol id="hair:windswept" viewBox="0 0 200 240"><!-- pixels trailing to one side --></symbol>
<symbol id="hair:braids" viewBox="0 0 200 240"><!-- two pixel columns down sides --></symbol>
<symbol id="hair:ponytail" viewBox="0 0 200 240"><!-- pixel bundle at back --></symbol>
<symbol id="hair:undercut" viewBox="0 0 200 240"><!-- short sides, taller top pixels --></symbol>
```

15 symbols. EXACT ids. `<rect>` only.

---

## Prompt 3 — Beards (7) + Eyebrows (8)

Same scummbar pixel art style. All `<rect>` elements. Beards use `var(--hair-color)`. Eyebrows at y=72-82.

```xml
<symbol id="beard:stubble" viewBox="0 0 200 240"><!-- scattered single-pixel dots on jaw --></symbol>
<symbol id="beard:goatee" viewBox="0 0 200 240"><!-- pixel cluster on chin --></symbol>
<symbol id="beard:full-round" viewBox="0 0 200 240"><!-- blocky pixel beard covering jaw --></symbol>
<symbol id="beard:bushy-white" viewBox="0 0 200 240"><!-- large pixel beard, use #ccc --></symbol>
<symbol id="beard:handlebar" viewBox="0 0 200 240"><!-- wide pixel moustache with ends --></symbol>
<symbol id="beard:rugged" viewBox="0 0 200 240"><!-- more dots than stubble --></symbol>
<symbol id="beard:heavy" viewBox="0 0 200 240"><!-- dense pixel shadow on jaw --></symbol>

<symbol id="brow:thin-arched" viewBox="0 0 200 240"><!-- 1-pixel-high arch --></symbol>
<symbol id="brow:thick-straight" viewBox="0 0 200 240"><!-- 2-pixel-high straight bar --></symbol>
<symbol id="brow:angular" viewBox="0 0 200 240"><!-- stepped V-shape --></symbol>
<symbol id="brow:soft-rounded" viewBox="0 0 200 240"><!-- gentle stepped arch --></symbol>
<symbol id="brow:bushy-wild" viewBox="0 0 200 240"><!-- thick irregular pixel cluster --></symbol>
<symbol id="brow:raised" viewBox="0 0 200 240"><!-- positioned higher than normal --></symbol>
<symbol id="brow:asymmetric" viewBox="0 0 200 240"><!-- one higher than other --></symbol>
<symbol id="brow:concerned" viewBox="0 0 200 240"><!-- inner ends raised --></symbol>
```

15 symbols. EXACT ids.

---

## Prompt 4 — Glasses (8) + Accessories (12)

Same scummbar pixel art style. All `<rect>`. Glasses use `var(--accent)` for frames. Accessories use `var(--accent)`.

```xml
<symbol id="glasses:round-wire" viewBox="0 0 200 240"><!-- pixel circles (stepped) --></symbol>
<symbol id="glasses:thick-rect" viewBox="0 0 200 240"><!-- bold pixel rectangles --></symbol>
<symbol id="glasses:aviator" viewBox="0 0 200 240"><!-- pixel teardrop shape --></symbol>
<symbol id="glasses:cat-eye" viewBox="0 0 200 240"><!-- upswept pixel corners --></symbol>
<symbol id="glasses:half-rim" viewBox="0 0 200 240"><!-- top bar only --></symbol>
<symbol id="glasses:monocle" viewBox="0 0 200 240"><!-- single pixel circle + chain --></symbol>
<symbol id="glasses:pince-nez" viewBox="0 0 200 240"><!-- two pixel ovals, bridge only --></symbol>
<symbol id="glasses:goggles" viewBox="0 0 200 240"><!-- wide pixel band --></symbol>

<symbol id="acc:ear-piercings" viewBox="0 0 200 240"><!-- 1-2 pixel dots at ear --></symbol>
<symbol id="acc:pendant-amulet" viewBox="0 0 200 240"><!-- pixel pendant at chest --></symbol>
<symbol id="acc:headband" viewBox="0 0 200 240"><!-- pixel band across forehead --></symbol>
<symbol id="acc:scarf-bandana" viewBox="0 0 200 240"><!-- pixel neck wrap --></symbol>
<symbol id="acc:hat-explorer" viewBox="0 0 200 240"><!-- wide pixel brim hat --></symbol>
<symbol id="acc:beret" viewBox="0 0 200 240"><!-- angled pixel cap --></symbol>
<symbol id="acc:flower-crown" viewBox="0 0 200 240"><!-- pixel flowers on head --></symbol>
<symbol id="acc:epaulettes" viewBox="0 0 200 240"><!-- pixel shoulder decorations --></symbol>
<symbol id="acc:scar" viewBox="0 0 200 240"><!-- diagonal pixel line on cheek --></symbol>
<symbol id="acc:freckles" viewBox="0 0 200 240"><!-- scattered single pixels --></symbol>
<symbol id="acc:tattoo" viewBox="0 0 200 240"><!-- small pixel pattern --></symbol>
<symbol id="acc:nose-ring" viewBox="0 0 200 240"><!-- 1-2 pixel dot at nose --></symbol>
```

20 symbols. EXACT ids.

---

## Prompt 5 — Costumes (20, batch 1 of 2)

Same scummbar pixel art style. All `<rect>`. Costumes fill y=130 to y=240. Use `var(--primary)` for main, `var(--secondary)` for details.

```xml
<symbol id="costume:blazer-tie" viewBox="0 0 200 240"><!-- pixel jacket + tie --></symbol>
<symbol id="costume:tweed-patches" viewBox="0 0 200 240"><!-- pixel jacket + elbow patches --></symbol>
<symbol id="costume:robes" viewBox="0 0 200 240"><!-- flowing pixel robes --></symbol>
<symbol id="costume:lab-coat" viewBox="0 0 200 240"><!-- white pixel coat --></symbol>
<symbol id="costume:armour" viewBox="0 0 200 240"><!-- pixel chest plate + shoulders --></symbol>
<symbol id="costume:utility-vest" viewBox="0 0 200 240"><!-- pixel vest + pockets --></symbol>
<symbol id="costume:leather-jacket" viewBox="0 0 200 240"><!-- dark pixel jacket --></symbol>
<symbol id="costume:smock" viewBox="0 0 200 240"><!-- loose pixel smock --></symbol>
<symbol id="costume:formal-sash" viewBox="0 0 200 240"><!-- pixel jacket + diagonal sash --></symbol>
<symbol id="costume:diplomatic" viewBox="0 0 200 240"><!-- clean pixel suit --></symbol>
```

10 symbols. EXACT ids. If output limit hits, stop here.

---

## Prompt 6 — Costumes (batch 2 of 2)

Same scummbar pixel art style. Continuing costumes.

```xml
<symbol id="costume:vest-cross" viewBox="0 0 200 240"><!-- pixel vest + cross --></symbol>
<symbol id="costume:soft-wrap" viewBox="0 0 200 240"><!-- pixel cardigan --></symbol>
<symbol id="costume:plain-shirt" viewBox="0 0 200 240"><!-- simple pixel shirt --></symbol>
<symbol id="costume:polo" viewBox="0 0 200 240"><!-- pixel polo shirt --></symbol>
<symbol id="costume:simple-dress" viewBox="0 0 200 240"><!-- pixel dress --></symbol>
<symbol id="costume:performer" viewBox="0 0 200 240"><!-- bright pixel outfit --></symbol>
<symbol id="costume:hoodie" viewBox="0 0 200 240"><!-- pixel hoodie --></symbol>
<symbol id="costume:romantic" viewBox="0 0 200 240"><!-- pixel flowing blouse --></symbol>
<symbol id="costume:business" viewBox="0 0 200 240"><!-- crisp pixel business --></symbol>
<symbol id="costume:explorer-jacket" viewBox="0 0 200 240"><!-- rugged pixel jacket --></symbol>
```

10 symbols. EXACT ids.

---

## Prompt 7 — Props (batch 1 of 5, 19 props)

Same scummbar pixel art style. All `<rect>`. Props at x=130-185, y=120-210. Use `var(--accent)` for main colour.

```xml
<symbol id="prop:magnifying-glass" viewBox="0 0 200 240"><!-- pixel lens + handle --></symbol>
<symbol id="prop:notebook" viewBox="0 0 200 240"><!-- pixel rectangle --></symbol>
<symbol id="prop:book-open" viewBox="0 0 200 240"><!-- pixel open book --></symbol>
<symbol id="prop:chalk" viewBox="0 0 200 240"><!-- small pixel stick --></symbol>
<symbol id="prop:crystal-ball" viewBox="0 0 200 240"><!-- pixel sphere on stand --></symbol>
<symbol id="prop:feathers" viewBox="0 0 200 240"><!-- pixel feather shapes --></symbol>
<symbol id="prop:scroll" viewBox="0 0 200 240"><!-- pixel scroll --></symbol>
<symbol id="prop:rosetta-stone" viewBox="0 0 200 240"><!-- pixel tablet --></symbol>
<symbol id="prop:shield" viewBox="0 0 200 240"><!-- pixel shield --></symbol>
<symbol id="prop:sword-hilt" viewBox="0 0 200 240"><!-- pixel sword handle --></symbol>
<symbol id="prop:medal" viewBox="0 0 200 240"><!-- pixel medal --></symbol>
<symbol id="prop:wristbands" viewBox="0 0 200 240"><!-- pixel wrist bands --></symbol>
<symbol id="prop:torch" viewBox="0 0 200 240"><!-- pixel torch + flame --></symbol>
<symbol id="prop:broken-chain" viewBox="0 0 200 240"><!-- pixel chain links --></symbol>
<symbol id="prop:first-aid" viewBox="0 0 200 240"><!-- pixel box + cross --></symbol>
<symbol id="prop:rope" viewBox="0 0 200 240"><!-- pixel coil --></symbol>
<symbol id="prop:glowing-orb" viewBox="0 0 200 240"><!-- pixel orb --></symbol>
<symbol id="prop:smoke-wisps" viewBox="0 0 200 240"><!-- pixel wisps --></symbol>
<symbol id="prop:wrench-gear" viewBox="0 0 200 240"><!-- pixel wrench + gear --></symbol>
```

19 symbols. EXACT ids.

---

## Prompt 8 — Props (batch 2 of 5, 19 props)

Same style, same rules.

```xml
<symbol id="prop:schematic" viewBox="0 0 200 240"><!-- pixel blueprint --></symbol>
<symbol id="prop:lightbulb" viewBox="0 0 200 240"><!-- pixel bulb --></symbol>
<symbol id="prop:circuit-traces" viewBox="0 0 200 240"><!-- pixel traces --></symbol>
<symbol id="prop:flask" viewBox="0 0 200 240"><!-- pixel flask --></symbol>
<symbol id="prop:periodic-table" viewBox="0 0 200 240"><!-- pixel grid --></symbol>
<symbol id="prop:megaphone" viewBox="0 0 200 240"><!-- pixel cone --></symbol>
<symbol id="prop:raised-fist" viewBox="0 0 200 240"><!-- pixel fist --></symbol>
<symbol id="prop:dice" viewBox="0 0 200 240"><!-- pixel cube --></symbol>
<symbol id="prop:poker-chip" viewBox="0 0 200 240"><!-- pixel circle --></symbol>
<symbol id="prop:wrench" viewBox="0 0 200 240"><!-- pixel wrench --></symbol>
<symbol id="prop:motorcycle-key" viewBox="0 0 200 240"><!-- pixel key --></symbol>
<symbol id="prop:hammer" viewBox="0 0 200 240"><!-- pixel hammer --></symbol>
<symbol id="prop:blueprint-torn" viewBox="0 0 200 240"><!-- torn pixel paper --></symbol>
<symbol id="prop:compass" viewBox="0 0 200 240"><!-- pixel compass --></symbol>
<symbol id="prop:backpack" viewBox="0 0 200 240"><!-- pixel pack --></symbol>
<symbol id="prop:swiss-army" viewBox="0 0 200 240"><!-- pixel multi-tool --></symbol>
<symbol id="prop:flag" viewBox="0 0 200 240"><!-- pixel flag --></symbol>
<symbol id="prop:machete" viewBox="0 0 200 240"><!-- pixel blade --></symbol>
<symbol id="prop:lantern" viewBox="0 0 200 240"><!-- pixel lantern --></symbol>
```

19 symbols. EXACT ids.

---

## Prompt 9 — Props (batch 3 of 5, 19 props)

Same style, same rules.

```xml
<symbol id="prop:journal" viewBox="0 0 200 240"><!-- pixel book --></symbol>
<symbol id="prop:paintbrush" viewBox="0 0 200 240"><!-- pixel brush --></symbol>
<symbol id="prop:palette" viewBox="0 0 200 240"><!-- pixel palette --></symbol>
<symbol id="prop:blueprint" viewBox="0 0 200 240"><!-- pixel plan --></symbol>
<symbol id="prop:laptop" viewBox="0 0 200 240"><!-- pixel laptop --></symbol>
<symbol id="prop:quill" viewBox="0 0 200 240"><!-- pixel feather pen --></symbol>
<symbol id="prop:open-book" viewBox="0 0 200 240"><!-- pixel open pages --></symbol>
<symbol id="prop:telescope" viewBox="0 0 200 240"><!-- pixel scope --></symbol>
<symbol id="prop:star-chart" viewBox="0 0 200 240"><!-- pixel chart --></symbol>
<symbol id="prop:butterfly" viewBox="0 0 200 240"><!-- pixel butterfly --></symbol>
<symbol id="prop:dandelion" viewBox="0 0 200 240"><!-- pixel flower --></symbol>
<symbol id="prop:cloud" viewBox="0 0 200 240"><!-- pixel cloud --></symbol>
<symbol id="prop:stars" viewBox="0 0 200 240"><!-- pixel stars --></symbol>
<symbol id="prop:candle" viewBox="0 0 200 240"><!-- pixel candle --></symbol>
<symbol id="prop:banner" viewBox="0 0 200 240"><!-- pixel banner --></symbol>
<symbol id="prop:spark" viewBox="0 0 200 240"><!-- pixel spark --></symbol>
<symbol id="prop:music-notes" viewBox="0 0 200 240"><!-- pixel notes --></symbol>
<symbol id="prop:juggling-balls" viewBox="0 0 200 240"><!-- pixel balls --></symbol>
<symbol id="prop:red-nose" viewBox="0 0 200 240"><!-- pixel clown nose --></symbol>
```

19 symbols. EXACT ids.

---

## Prompt 10 — Props (batch 4 of 5, 19 props)

Same style, same rules.

```xml
<symbol id="prop:microphone" viewBox="0 0 200 240"><!-- pixel mic --></symbol>
<symbol id="prop:spotlight" viewBox="0 0 200 240"><!-- pixel light cone --></symbol>
<symbol id="prop:mirror-mask" viewBox="0 0 200 240"><!-- pixel mask --></symbol>
<symbol id="prop:speech-bubble" viewBox="0 0 200 240"><!-- pixel bubble --></symbol>
<symbol id="prop:playing-cards" viewBox="0 0 200 240"><!-- pixel cards --></symbol>
<symbol id="prop:shadow-self" viewBox="0 0 200 240"><!-- dark pixel silhouette --></symbol>
<symbol id="prop:gift-box" viewBox="0 0 200 240"><!-- pixel box + ribbon --></symbol>
<symbol id="prop:scarf-shared" viewBox="0 0 200 240"><!-- pixel scarf --></symbol>
<symbol id="prop:wine-glass" viewBox="0 0 200 240"><!-- pixel glass --></symbol>
<symbol id="prop:grapes" viewBox="0 0 200 240"><!-- pixel grape cluster --></symbol>
<symbol id="prop:ribbon" viewBox="0 0 200 240"><!-- pixel ribbon --></symbol>
<symbol id="prop:address-book" viewBox="0 0 200 240"><!-- pixel book with tabs --></symbol>
<symbol id="prop:rose" viewBox="0 0 200 240"><!-- pixel rose --></symbol>
<symbol id="prop:poetry-book" viewBox="0 0 200 240"><!-- pixel book --></symbol>
<symbol id="prop:halo" viewBox="0 0 200 240"><!-- pixel ring above head --></symbol>
<symbol id="prop:dove" viewBox="0 0 200 240"><!-- pixel bird --></symbol>
<symbol id="prop:umbrella" viewBox="0 0 200 240"><!-- pixel umbrella --></symbol>
<symbol id="prop:shield-small" viewBox="0 0 200 240"><!-- small pixel shield --></symbol>
<symbol id="prop:stethoscope" viewBox="0 0 200 240"><!-- pixel stethoscope --></symbol>
```

19 symbols. EXACT ids.

---

## Prompt 11 — Props (batch 5 of 5, 19 props, FINAL)

Same style, same rules. This is the last batch.

```xml
<symbol id="prop:herb-bundle" viewBox="0 0 200 240"><!-- pixel herbs --></symbol>
<symbol id="prop:bandage" viewBox="0 0 200 240"><!-- pixel bandage --></symbol>
<symbol id="prop:toolkit" viewBox="0 0 200 240"><!-- pixel toolbox --></symbol>
<symbol id="prop:clipboard" viewBox="0 0 200 240"><!-- pixel clipboard --></symbol>
<symbol id="prop:pen" viewBox="0 0 200 240"><!-- pixel pen --></symbol>
<symbol id="prop:phone" viewBox="0 0 200 240"><!-- pixel phone --></symbol>
<symbol id="prop:business-cards" viewBox="0 0 200 240"><!-- pixel cards --></symbol>
<symbol id="prop:broom" viewBox="0 0 200 240"><!-- pixel broom --></symbol>
<symbol id="prop:cloth" viewBox="0 0 200 240"><!-- pixel cloth --></symbol>
<symbol id="prop:megaphone-small" viewBox="0 0 200 240"><!-- small pixel megaphone --></symbol>
<symbol id="prop:leaflet" viewBox="0 0 200 240"><!-- pixel paper --></symbol>
<symbol id="prop:olive-branch" viewBox="0 0 200 240"><!-- pixel branch --></symbol>
<symbol id="prop:treaty" viewBox="0 0 200 240"><!-- pixel document --></symbol>
<symbol id="prop:gavel" viewBox="0 0 200 240"><!-- pixel gavel --></symbol>
<symbol id="prop:scales" viewBox="0 0 200 240"><!-- pixel balance --></symbol>
<symbol id="prop:scepter" viewBox="0 0 200 240"><!-- pixel scepter --></symbol>
<symbol id="prop:family-crest" viewBox="0 0 200 240"><!-- pixel heraldic shield --></symbol>
<symbol id="prop:crown" viewBox="0 0 200 240"><!-- pixel crown --></symbol>
<symbol id="prop:orb-of-state" viewBox="0 0 200 240"><!-- pixel orb --></symbol>
```

19 symbols. EXACT ids. This completes the scummbar collection — 177 standard symbols total!

---

## Verification Checklist (run after each batch)

Give Claude (me) each downloaded SVG and I will verify:
1. Correct ID count and names
2. No `<path>`, `<circle>`, `<ellipse>`, `<polygon>` elements (pixel art = rects only)
3. Uses `var(--*)` not hardcoded colours
4. No background rectangles
5. Composability (transparent layers)
