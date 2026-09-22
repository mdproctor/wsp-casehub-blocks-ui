# Part Catalogue — Archetype Avatar System

Visual reference for the composable SVG part registry.
Live preview: `avatar-preview.html` (serve locally to view).

## Hair Styles (16)

| ID | Name | Silhouette | Archetype Affinity |
|----|------|------------|-------------------|
| `hair-bald-sides` | Bald + sides | Exposed dome, grey side patches | Sage/Detective, Sage/Translator |
| `hair-buzz` | Buzz cut | Thin shadow cap, stubble texture | Hero/Warrior, Hero/Athlete, Hero/Rescuer, Magician/Engineer, Everyman/Servant |
| `hair-afro-short` | Short afro | Rounded volume extending beyond head | Caregiver/Guardian, Everyman/Citizen |
| `hair-long-flowing` | Long flowing | Past shoulders, drapes both sides | Magician/Alchemist, Sage/Shaman, Jester/Shapeshifter, Lover/Romantic, Caregiver/Healer, Innocent/Dreamer |
| `hair-messy-bun` | Messy bun | High knot with loose strands falling | Creator/Artist, Innocent/Muse, Lover/Matchmaker |
| `hair-mohawk` | Mohawk | Tall central spike, shaved sides visible | Rebel/Maverick, Jester/Provocateur |
| `hair-wild-einstein` | Wild Einstein | Exploding outward, frizzy strands | Sage/Mentor, Jester/Clown, Creator/Visionary |
| `hair-slicked` | Slicked back | Swept tight, glossy lines, grey temples | Sovereign/Ruler, Sovereign/Judge, Sovereign/Ambassador, Rebel/Gambler, Lover/Hedonist, Everyman/Networker |
| `hair-braids` | Braids / plaits | Symmetrical braids, textured | Explorer/Pioneer, Rebel/Activist |
| `hair-shoulder-wavy` | Shoulder-length wavy | Past jawline, soft waves | Caregiver/Angel, Lover/Companion, Explorer/Generalist, Innocent/Idealist, Creator/Storyteller |
| `hair-cropped-fringe` | Cropped with fringe | Short sides, textured fringe forward | Caregiver/Samaritan, Everyman/Advocate, Jester/Entertainer, Sage/Translator, Everyman/Citizen (alt), Innocent/Child |
| `hair-undercut` | Undercut | Shaved sides, longer top swept | Rebel/Reformer |
| `hair-windswept` | Windswept | Swept to one side, tousled by wind | Explorer/Adventurer |
| `hair-pixie` | Pixie / short textured | Cropped, textured, modern | Creator/Entrepreneur, Magician/Innovator |
| `hair-ponytail` | Ponytail | Pulled back, tied, practical | Explorer/Seeker, Hero/Liberator |
| `hair-headwrap` | Headscarf / head wrap | Wrapped fabric, colour accent | Caregiver/Angel (alt), Magician/Shaman (alt) |

## Facial Hair (8)

| ID | Name | Shape | Archetype Affinity |
|----|------|-------|-------------------|
| `beard-none` | Clean shaven | No facial hair | Hero/Athlete, Innocent/*, Jester/* |
| `beard-stubble` | Stubble | Light shadow on jaw | Explorer/Adventurer, Rebel/Gambler |
| `beard-goatee` | Pointed goatee | V-shape chin point with moustache connect | Magician/Alchemist, Magician/Scientist |
| `beard-full-round` | Full round | Dense, covers jaw to cheeks, rounded bottom | Caregiver/Guardian, Everyman/Citizen |
| `beard-bushy-white` | Bushy white | Large, textured, flowing — with hair strands | Sage/Mentor (Einstein pairing) |
| `beard-handlebar` | Handlebar moustache | Wide moustache with curled upswept ends | Sovereign/Ruler, Sovereign/Patriarch |
| `beard-rugged` | Rugged stubble | Heavier than stubble, visible dot texture | Explorer/Pioneer, Hero/Rescuer |
| `beard-heavy` | Heavy stubble | Dense shadow, almost-beard | Rebel/Maverick, Rebel/Reformer |

### Reserved for future expansion

- Van Dyke — Magician/Engineer variant
- Mutton chops — Sovereign/Patriarch variant
- Soul patch — Jester/Provocateur variant
- Short boxed — Everyman/Advocate variant
- Long wizard — Sage/Shaman variant

## Glasses (8)

| ID | Name | Frame Shape | Archetype Affinity |
|----|------|-------------|-------------------|
| `glasses-round-wire` | Round wire | Thin circles, nose bridge, arm hooks | Sage/Detective, Sage/Translator |
| `glasses-thick-rect` | Thick rectangular | Bold black frames, wide bridge | Creator/Visionary, Sovereign/Judge |
| `glasses-aviator` | Aviator | Teardrop lenses, thin metal | Explorer/Pioneer, Hero/Liberator |
| `glasses-cat-eye` | Cat-eye | Upswept outer corners, bold colour | Creator/Artist, Jester/Shapeshifter |
| `glasses-half-rim` | Half-rim | Top rim only, wire bottom | Sage/Mentor, Everyman/Advocate |
| `glasses-monocle` | Monocle | Single lens, right eye, chain | Sovereign/Ruler, Sovereign/Ambassador |
| `glasses-pince-nez` | Pince-nez | Nose-pinch only, no arms | Sage/Shaman, Magician/Scientist |
| `glasses-goggles` | Goggles | Wraparound, tinted lenses, strap | Explorer/Adventurer, Magician/Engineer |

## Eyebrows (8)

| ID | Name | Shape | Expression |
|----|------|-------|-----------|
| `brow-thin-arched` | Thin arched | High arch, thin stroke | Composed, knowing |
| `brow-thick-straight` | Thick straight | Flat, heavy stroke | Determined, serious |
| `brow-angular` | Angular / angry | V-shape, sharp inner angle | Fierce, aggressive |
| `brow-soft-rounded` | Soft rounded | Gentle curve, medium weight | Warm, approachable |
| `brow-bushy-wild` | Bushy wild | Filled shape with tufts | Wise, aged, eccentric |
| `brow-raised` | Raised expressive | High lift, medium weight | Curious, surprised, open |
| `brow-asymmetric` | Asymmetric | One higher than other | Skeptical, cocky, questioning |
| `brow-concerned` | Concerned | Inner ends raised, outer lowered | Worried, empathic, caring |

## Head Shapes (11)

11 shapes for 12 families — Innocent and Caregiver share `head-round`. At xs size (24px), head shape + palette together differentiate: Innocent uses cream white, Caregiver uses forest green. The shared silhouette is intentional — both families share the "soft, approachable" visual language from their Stability/Independence quadrant overlap.

| ID | Name | Shape | Families |
|----|------|-------|----------|
| `head-round` | Round | Circle — soft, approachable | Innocent, Caregiver |
| `head-square-jaw` | Square jaw | Wide jaw, flat chin, strong angles | Hero |
| `head-diamond` | Diamond | Narrow chin and forehead, wide cheekbones | Magician |
| `head-heart` | Heart | Wide forehead, tapered pointed chin | Lover |
| `head-oval` | Oval | Elongated, scholarly | Sage |
| `head-angular` | Angular | Sharp angles, asymmetric hints | Rebel |
| `head-soft-oval` | Soft oval | Slightly wider, expressive proportions | Creator |
| `head-strong-sym` | Strong symmetrical | Square + oval hybrid, commanding | Sovereign |
| `head-weathered` | Weathered oval | Oval with stronger bone structure | Explorer |
| `head-round-wide` | Round-wide | Wider than round, more animated proportions | Jester |
| `head-standard` | Standard oval | Neutral — unremarkable, evenly proportioned | Everyman |

## Costumes (16+)

| ID | Name | Description | Archetype Affinity |
|----|------|-------------|-------------------|
| `costume-blazer-tie` | Blazer + tie | Formal jacket, collared shirt, tie | Sage/Detective, Sage/Translator |
| `costume-tweed-patches` | Tweed + elbow patches | Casual academic jacket | Sage/Mentor |
| `costume-robes` | Robes | Long flowing robes with trim | Magician/Alchemist, Magician/Shaman, Sage/Shaman |
| `costume-lab-coat` | Lab coat | White coat, buttoned | Magician/Scientist, Magician/Engineer |
| `costume-armour` | Armour + pauldrons | Chest plate, shoulder guards | Hero/Warrior, Hero/Athlete |
| `costume-utility-vest` | Utility vest | Multi-pocket vest, bandana | Explorer/Adventurer, Explorer/Pioneer |
| `costume-leather-jacket` | Leather jacket | Black/dark, popped collar, zipper | Rebel/Maverick, Rebel/Gambler |
| `costume-smock` | Paint smock | Loose, paint-splattered | Creator/Artist |
| `costume-formal-sash` | Formal + sash | Navy jacket, epaulettes, diagonal sash, medals | Sovereign/Ruler, Sovereign/Patriarch |
| `costume-diplomatic` | Diplomatic suit | Clean suit, pocket square | Sovereign/Ambassador, Sovereign/Judge |
| `costume-vest-cross` | Vest + cross | Sturdy vest with medical/caregiver symbol | Caregiver/Guardian, Caregiver/Samaritan |
| `costume-soft-wrap` | Soft wrap | Flowing cardigan/shawl | Caregiver/Angel, Caregiver/Healer |
| `costume-plain-shirt` | Plain shirt | Unremarkable collared or crew shirt | Everyman/Citizen, Everyman/Servant |
| `costume-polo` | Polo / casual | Smart-casual, approachable | Everyman/Advocate, Everyman/Networker |
| `costume-simple-dress` | Simple dress | Light, clean, unadorned | Innocent/Child, Innocent/Dreamer |
| `costume-performer` | Performer outfit | Bright, dynamic, sequins or bold pattern | Jester/Entertainer, Jester/Clown |
| `costume-hoodie` | Hoodie | Casual, drawn-up hood or loose | Jester/Provocateur, Rebel/Activist |
| `costume-romantic` | Romantic blouse | Soft, flowing, rich fabric | Lover/Romantic, Lover/Companion |
| `costume-business` | Business attire | Crisp, driven, polished | Creator/Entrepreneur, Hero/Liberator |
| `costume-explorer-jacket` | Explorer jacket | Weathered, rolled sleeves, rugged | Explorer/Generalist, Explorer/Seeker |

## Props (30+)

| ID | Name | Description | Sub-Archetype |
|----|------|-------------|---------------|
| `prop-magnifying-glass` | Magnifying glass | Held to side, lens + handle | Sage/Detective |
| `prop-book` | Book | Open or closed, held at side | Sage/Mentor |
| `prop-crystal-ball` | Crystal ball | Translucent orb, faint glow | Sage/Shaman |
| `prop-scroll` | Scroll | Unfurled parchment | Sage/Translator |
| `prop-shield` | Shield | Emblem on front, held at side | Hero/Warrior |
| `prop-medal` | Medal / trophy | Athletic achievement | Hero/Athlete |
| `prop-torch` | Torch | Raised, flame lit | Hero/Liberator |
| `prop-first-aid` | First aid kit | Cross-marked bag | Hero/Rescuer |
| `prop-glowing-orb` | Glowing orb | Magical, sparkles, floating | Magician/Alchemist |
| `prop-wrench-gear` | Wrench + gear | Engineering tools | Magician/Engineer |
| `prop-lightbulb` | Lightbulb | Lit, idea symbol | Magician/Innovator |
| `prop-flask` | Flask / beaker | Lab glass with liquid | Magician/Scientist |
| `prop-megaphone` | Megaphone | Held up, rallying | Rebel/Activist |
| `prop-dice` | Dice | Tumbling, mid-roll | Rebel/Gambler |
| `prop-wrench` | Wrench | Lone tool, self-reliant | Rebel/Maverick |
| `prop-hammer` | Hammer | Building/breaking tool | Rebel/Reformer |
| `prop-compass` | Compass | Navigational, held flat | Explorer/Adventurer |
| `prop-swiss-army` | Multi-tool | Versatile, folding | Explorer/Generalist |
| `prop-flag` | Flag / pennant | Planted, trailblazing | Explorer/Pioneer |
| `prop-lantern` | Lantern | Inner light, searching | Explorer/Seeker |
| `prop-paintbrush` | Paintbrush + palette | Art tools | Creator/Artist |
| `prop-blueprint` | Blueprint | Rolled plan / startup pitch | Creator/Entrepreneur |
| `prop-quill` | Quill pen | Writing, storytelling | Creator/Storyteller |
| `prop-telescope` | Telescope | Far-seeing, visionary | Creator/Visionary |
| `prop-dove` | Dove | Peace, innocence | Innocent/Child |
| `prop-cloud` | Cloud / dream bubble | Dreamy, floating thought | Innocent/Dreamer |
| `prop-candle` | Candle | Steady light, hope | Innocent/Idealist |
| `prop-spark` | Spark / star | Inspirational glint | Innocent/Muse |
| `prop-juggling` | Juggling balls | Mid-air, playful | Jester/Clown |
| `prop-microphone` | Microphone | Performing, stage | Jester/Entertainer |
| `prop-mirror-mask` | Mirror / mask | Dual face, subversive | Jester/Provocateur |
| `prop-playing-cards` | Playing cards | Fanned, trickster | Jester/Shapeshifter |
| `prop-flower` | Flower | Romance, beauty | Lover/Romantic |
| `prop-gift` | Gift box | Giving, generosity | Lover/Companion |
| `prop-wine` | Wine glass | Indulgence, pleasure | Lover/Hedonist |
| `prop-ribbon` | Ribbon | Connecting, matchmaking | Lover/Matchmaker |
| `prop-stethoscope` | Stethoscope | Medical care | Caregiver/Healer |
| `prop-umbrella` | Umbrella | Protection, sheltering | Caregiver/Guardian |
| `prop-halo` | Halo | Angelic glow above head | Caregiver/Angel |
| `prop-bandage` | Bandage roll | Practical first aid | Caregiver/Samaritan |
| `prop-crown` | Crown | Jewelled, regal | Sovereign/Ruler |
| `prop-gavel` | Gavel | Judicial, balanced | Sovereign/Judge |
| `prop-scepter` | Scepter | Authority, legacy | Sovereign/Patriarch |
| `prop-olive-branch` | Olive branch | Diplomacy, peace | Sovereign/Ambassador |
| `prop-clipboard` | Clipboard | Organizing, civic | Everyman/Citizen |
| `prop-handshake` | Linked hands | Networking, connection | Everyman/Networker |
| `prop-broom` | Broom / mop | Humble service | Everyman/Servant |
| `prop-megaphone-small` | Small megaphone | Advocacy, speaking up | Everyman/Advocate |

## Accessories (beyond glasses)

| ID | Name | Description | Archetype Affinity |
|----|------|-------------|-------------------|
| `acc-ear-piercings` | Ear piercings | Multiple studs/rings | Rebel/Maverick, Rebel/Gambler |
| `acc-pendant-amulet` | Pendant / amulet | Magical, glowing center | Magician/Alchemist |
| `acc-headband` | Headband | Athletic, practical | Hero/Athlete, Explorer/Adventurer |
| `acc-scarf-bandana` | Scarf / bandana | Neck wrap, colour accent | Explorer/*, Creator/Artist |
| `acc-hat-explorer` | Explorer hat | Wide brim, weathered | Explorer/Adventurer, Explorer/Pioneer |
| `acc-beret` | Beret | Artistic, angled | Creator/Artist, Creator/Storyteller |
| `acc-flower-crown` | Flower crown | Woven flowers on head | Innocent/Muse, Lover/Romantic |
| `acc-epaulettes` | Epaulettes | Gold shoulder decorations | Sovereign/Ruler, Sovereign/Patriarch |
| `acc-scar` | Scar | Face scar, battle mark | Hero/Warrior, Rebel/Reformer |
| `acc-freckles` | Freckles | Across cheeks and nose | Innocent/Child, Everyman/Citizen |
| `acc-tattoo` | Visible tattoo | Neck or arm ink | Rebel/Activist, Rebel/Maverick |
| `acc-nose-ring` | Nose ring | Small hoop or stud | Rebel/Activist, Jester/Provocateur |

## Full Sub-Archetype Part Assignments (48)

The definitive mapping — each row is a complete visual identity.

### Caregiver (head: round, palette: forest green)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Angel | `shoulder-wavy` | `flower-crown` | none | `soft-wrap` | halo + dove | — | `soft-rounded` | `rosy-cheeks` | — | Radiant glow behind head, flowing light fabric |
| Guardian | `afro-short` | — | `full-round` | `vest-cross` | umbrella + shield-small | — | `thick-straight` | `furrowed-brows` | — | Broad stance, protective posture, big beard |
| Healer | `long-flowing` | — | none | `soft-wrap` | stethoscope + herb-bundle | — | `concerned` | `rosy-cheeks` | — | Gentle expression, green herbs, caring hands |
| Samaritan | `cropped-fringe` | — | `stubble` | `vest-cross` | bandage + toolkit | — | `soft-rounded` | `sweat-drop` | `headband` | Practical, sleeves-rolled, ready to help |

### Everyman (head: standard, palette: neutral grey)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Advocate | `cropped-fringe` | — | none | `polo` | megaphone-small + leaflet | `half-rim` | `raised` | `raised-brow` | — | Leaning forward, passionate speaker pose |
| Networker | `slicked` | — | none | `polo` | phone + business-cards | — | `soft-rounded` | `squint-joy` | — | Open-handed gesture, warm smile |
| Servant | `buzz` | — | none | `plain-shirt` | broom + cloth | — | `concerned` | `flat-brows` | — | Humble posture, rolled sleeves, simple |
| Citizen | `afro-short` | `baseball-cap` | `stubble` | `plain-shirt` | clipboard + pen | — | `soft-rounded` | — | `freckles` | Unremarkable — the most generic avatar |

### Creator (head: soft-oval, palette: bold red)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Artist | `messy-bun` | `beret` | none | `smock` | paintbrush + palette | `cat-eye` | `raised` | `sparkle-eyes` | paint splatters | Colourful chaos, pencil in bun |
| Entrepreneur | `pixie` | — | none | `business` | blueprint + laptop | `thick-rect` | `thick-straight` | `idea-spark` | — | Crisp, driven, pitch-ready |
| Storyteller | `shoulder-wavy` | — | `goatee` | `smock` | quill + open-book | — | `raised` | `raised-brow` | `scarf-bandana` | Narrative gesture, dramatic expression |
| Visionary | `wild-einstein` | — | none | `business` | telescope + star-chart | `thick-rect` | `raised` | `starry-eyes` | — | Eyes up, seeing beyond the horizon |

### Innocent (head: round, palette: cream white)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Child | `cropped-fringe` | — | none | `simple-dress` | butterfly + dandelion | — | `raised` | `rosy-cheeks` | `freckles` | Wide eyes, wonder-struck, rosy cheeks |
| Dreamer | `long-flowing` | — | none | `simple-dress` | cloud + stars | — | `soft-rounded` | `starry-eyes` | — | Upward gaze, floating sparkles |
| Idealist | `shoulder-wavy` | — | none | `simple-dress` | candle + banner | — | `raised` | `sparkle-eyes` | — | Steady flame, hopeful expression |
| Muse | `messy-bun` | `flower-crown` | none | `simple-dress` | spark + music-notes | — | `thin-arched` | `sparkle-eyes` | — | Luminous aura, ethereal quality |

### Explorer (head: weathered, palette: earth brown)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Adventurer | `windswept` | `explorer` | `rugged` | `utility-vest` | compass + rope | `goggles` | `thick-straight` | `squint-joy` | — | Windblown, squinting, rugged gear |
| Generalist | `shoulder-wavy` | — | `stubble` | `explorer-jacket` | swiss-army + backpack | — | `soft-rounded` | — | `scarf-bandana` | Casual competence, many pockets |
| Pioneer | `braids` | — | `rugged` | `utility-vest` | flag + machete | `aviator` | `thick-straight` | `furrowed-brows` | `headband` | Forward-leaning, trailblazer stance |
| Seeker | `ponytail` | — | none | `explorer-jacket` | lantern + journal | — | `concerned` | — | — | Introspective gaze, walking inward |

### Hero (head: square-jaw, palette: crimson)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Athlete | `buzz` | — | none | `armour` | medal + wristbands | — | `thick-straight` | `furrowed-brows` | `headband` | Lean, disciplined, athletic tape on hands |
| Liberator | `ponytail` | — | `heavy` | `business` | torch + broken-chain | `aviator` | `angular` | `angry-vein` | — | Raised torch, fierce determined gaze |
| Rescuer | `buzz` | — | `rugged` | `armour` | first-aid + rope | — | `angular` | `sweat-drop` | `scar` | Alert stance, ready-to-move tension |
| Warrior | `buzz` | — | none | `armour` | shield + sword-hilt | — | `angular` | `furrowed-brows` | `scar` | Broadest shoulders, most imposing silhouette |

### Jester (head: round-wide, palette: orange/purple)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Clown | `wild-einstein` | `jester` | none | `performer` | juggling-balls + red-nose | — | `raised` | `squint-joy` | — | Big red nose is the instant read |
| Entertainer | `cropped-fringe` | — | none | `performer` | microphone + spotlight | — | `raised` | `wink` | — | Stage presence, dazzling sequins |
| Provocateur | `mohawk` | — | `stubble` | `hoodie` | mirror-mask + speech-bubble | — | `asymmetric` | `raised-brow` | `nose-ring` | Half-mask, subversive smirk |
| Shapeshifter | `long-flowing` | — | none | `performer` | playing-cards + shadow-self | `cat-eye` | `thin-arched` | `dazed-spirals` | — | Faded duplicate silhouette behind |

### Lover (head: heart, palette: deep rose)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Companion | `shoulder-wavy` | — | none | `romantic` | gift-box + scarf-shared | — | `soft-rounded` | `rosy-cheeks` | — | Warm smile, comfortable steady presence |
| Hedonist | `slicked` | — | `stubble` | `romantic` | wine-glass + grapes | — | `thin-arched` | `wink` | — | Luxurious, sensual, rich textures |
| Matchmaker | `messy-bun` | — | none | `romantic` | ribbon + address-book | — | `raised` | `squint-joy` | — | Connecting gesture, knowing smile |
| Romantic | `long-flowing` | `flower-crown` | none | `romantic` | rose + poetry-book | — | `soft-rounded` | `heart-eyes` | — | Most flowing hair, dreamiest expression |

### Magician (head: diamond, palette: deep purple)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Alchemist | `long-flowing` | — | `goatee` | `robes` | glowing-orb + smoke-wisps | — | `thin-arched` | `sparkle-eyes` | `pendant-amulet` | Mysterious, orb light illuminates face |
| Engineer | `buzz` | — | `stubble` | `lab-coat` | wrench-gear + schematic | `goggles` | `thick-straight` | — | — | Goggles on forehead, systematic precision |
| Innovator | `pixie` | — | none | `business` | lightbulb + circuit-traces | — | `raised` | `idea-spark` | — | Eyes lit up, eureka energy |
| Scientist | `bald-sides` | — | `goatee` | `lab-coat` | flask + periodic-table | `pince-nez` | `thick-straight` | `raised-brow` | — | Peering at flask, empirical focus |

### Rebel (head: angular, palette: black)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Activist | `braids` | — | none | `hoodie` | megaphone + raised-fist | — | `angular` | `angry-vein` | `tattoo` | Fist up, mouth open, rallying cry |
| Gambler | `slicked` | — | `stubble` | `leather-jacket` | dice + poker-chip | — | `asymmetric` | `wink` | `ear-piercings` | Cocky smirk, dice in mid-toss |
| Maverick | `mohawk` | — | `heavy` | `leather-jacket` | wrench + motorcycle-key | — | `asymmetric` | `raised-brow` | `ear-piercings` | Tallest mohawk, most piercings, defiant |
| Reformer | `undercut` | — | `heavy` | `hoodie` | hammer + blueprint-torn | — | `angular` | `furrowed-brows` | `scar` | Tearing down to rebuild, constructive fury |

### Sage (head: oval, palette: cool blue)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Detective | `bald-sides` | — | none | `blazer-tie` | magnifying-glass + notebook | `round-wire` | `thin-arched` | `squint-joy` | — | Scrutinising squint, glass raised |
| Mentor | `wild-einstein` | — | `bushy-white` | `tweed-patches` | book-open + chalk | `half-rim` | `bushy-wild` | — | — | Most hair+beard volume of any archetype |
| Shaman | `long-flowing` | — | none | `robes` | crystal-ball + feathers | `pince-nez` | `thin-arched` | `dazed-spirals` | `pendant-amulet` | Otherworldly gaze, liminal quality |
| Translator | `cropped-fringe` | — | none | `blazer-tie` | scroll + rosetta-stone | `round-wire` | `soft-rounded` | — | — | Bridging gesture, accessible expression |

### Sovereign (head: strong-sym, palette: navy/gold)

| Sub-Archetype | Hair | Hat | Facial Hair | Costume | Props | Glasses | Eyebrows | Expression | Accessories | Visual Hook |
|---------------|------|-----|-------------|---------|-------|---------|----------|------------|-------------|-------------|
| Ambassador | `slicked` | — | none | `diplomatic` | olive-branch + treaty | — | `soft-rounded` | — | — | Open palm gesture, diplomatic smile |
| Judge | `slicked` | — | none | `diplomatic` | gavel + scales | `thick-rect` | `thick-straight` | `furrowed-brows` | — | Stern, measuring, scales in balance |
| Patriarch | `slicked` | — | `handlebar` | `formal-sash` | scepter + family-crest | — | `thick-straight` | `furrowed-brows` | `epaulettes` | Most imposing Sovereign, legacy weight |
| Ruler | `slicked` | `crown` | `handlebar` | `formal-sash` | crown + orb-of-state | `monocle` | `angular` | — | `epaulettes` | Crown is the instant read, most gold |

## Family Colour Palettes

| Family | Primary | Secondary | Accent | Skin emphasis |
|--------|---------|-----------|--------|---------------|
| Caregiver | `#2e5940` forest green | `#3d7a55` | `#e8e4dc` cream | Warm |
| Everyman | `#5a5a5a` neutral grey | `#7a7a7a` | `#d4c4a8` khaki | Neutral |
| Creator | `#c0392b` bold red | `#e67e22` orange | `#f1c40f` yellow | Warm |
| Innocent | `#f0e6d8` cream white | `#ddd4c4` | `#f4d03f` golden | Light |
| Explorer | `#5a4a35` earth brown | `#6a5a45` | `#c0392b` red scarf | Tanned |
| Hero | `#8b1a1a` crimson | `#c0c0c0` silver | `#ffd700` gold | Medium |
| Jester | `#e67e22` orange | `#9b59b6` purple | `#2ecc71` green | Variable |
| Lover | `#8b2252` deep rose | `#c0546a` | `#ffd700` gold | Warm |
| Magician | `#2d1b4e` deep purple | `#5b3a8c` | `#d4a0ff` glow | Olive |
| Rebel | `#1a1a1a` black | `#333` charcoal | `#cc0000` red | Variable |
| Sage | `#2c3e6b` cool blue | `#4a6fa5` | `#e8e4dc` parchment | Neutral |
| Sovereign | `#1a2744` navy | `#c9a227` gold | `#8b0000` sash red | Medium |

## Iteration Notes

This catalogue is a living document. To iterate:
1. Edit `avatar-preview.html` — add/modify SVG parts
2. Serve locally and screenshot to verify
3. Update this table with new part IDs and affinities
4. The spec (`*-design.md`) references this catalogue but doesn't duplicate the SVG data

### Uniqueness audit

Scan the assignment table for duplicate part combinations. Within a family, every sub-archetype
must differ by at least 2 parts (not counting palette — that's shared). Across families, the
head shape + palette already differentiates, but props should still be unique per sub-archetype.
