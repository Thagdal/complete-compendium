# Spelljammer Ships — Research Handoff

**Status:** Research only. No code or data changes made yet.
**Branch:** `claude/spelljammer-ship-wiki-mjcgr8`
**Written:** 2026-08-12, from a remote (cloud) Claude Code session.

This document brings a fresh session up to speed on adding **spelljamming ship data** to the
Compendium. Read it before starting work — it records what was verified, what wasn't, and which
dead ends not to repeat.

---

## 0. Why this doc exists / what's different in a local session

The research was done in a cloud container with a restrictive egress policy. Two things were
blocked there that are **probably fine locally** — verify early, because they change the plan:

| Resource | Cloud session | Check locally first |
|---|---|---|
| `spelljammer.fandom.com` (all of `fandom.com`) | ❌ 403 at proxy | Likely reachable |
| `archive.org` (incl. `ia*.us.archive.org`) | ❌ 403 at proxy | Likely reachable |
| Local filesystem (`D:\Downloads\spelljammer texts`) | ❌ not mounted | ✅ Available |

Everything in §1 about the Fandom wiki came from **web-search summaries, not from reading the
pages**. Treat it as a strong lead, not verified schema. Everything in §2 and §3 was verified
directly against files in this repo.

---

## 1. The Spelljammer Fandom wiki (UNVERIFIED — search-derived)

`https://spelljammer.fandom.com` — a 2e-centric fan wiki. Ship information appears in these forms:

1. **Ship-class pages with a stat infobox** — one page per hull design (Hammership, Nautiloid,
   Tradesman, Dragonship, Damselfly, Citadel, Armada, Tyrant Ship, Deathspider, Lamprey Ship,
   Vipership, Xebec, Wreckboat…). Indexed at `/wiki/Category:Ships`.
2. **Prose per ship** — description, appearance, history, variants, notable vessels of the class.
3. **Rules pages, separate from ships** — `Spelljamming`, `Spelljammer_Helms` (major/minor/pool/arc,
   plus cloaking helm, box-helm), `Maneuverability_Class`, `Ship's_Rating`, and weapon pages
   (Ballista, Helmseeker, Helm-Bomb).
4. **Named unique vessels vs. classes** — `Category:Named_spelljamming_ships` (The Spelljammer,
   Lady Lenore, The Batship) with a parallel unnamed category. **A class and an instance are
   different record types — model them separately.**
5. **Ships grouped by builder race** — elven (Armada, Man-o-War, Flitter), illithid (Nautiloid,
   Dreadnought), neogi (Deathspider, Urchin, Leech), dwarven (Citadel), beholder (Tyrant Ship).
6. **Source/provenance citations** — pages cite the TSR product each design came from
   (Concordance of Arcane Space, Lorebook of the Void, War Captain's Companion, SJR1 Lost Ships),
   and there are canon-tagging categories such as `Category:LotV canon`. There are also product
   pages listing which ships each book introduced. **This maps well onto our existing
   source/product model.**
7. **Living / irregular vessels** — Space Leviathan, ghost ships, lifejammer-driven creature
   conversions. These straddle the ship/monster boundary and may already exist in our monster data.
8. **Images and deckplans** — art is common; deckplan coverage is uneven and unconfirmed.

### Two cautions before ingesting anything from there

- **Homebrew is mixed in with canon.** Ships tagged "Pyrespace" (Warbird, Wreckship) are
  fan-setting material sitting in the same `Category:Ships` listing as TSR designs. This is the
  same problem as the Book of Sacrifices / Book of Shadows entries called out in
  `DEVELOPMENT_PLAN.MD`. Decide the provenance flag *before* import, not after. The wiki's own
  `*canon` categories are a partial signal.
- **Licensing.** Fandom text is CC BY-SA (attribution + share-alike), layered over TSR/WotC IP.
  Different footing from the Lomion-derived material already hosted here. Worth a deliberate
  decision.

### If the wiki is reachable locally

Don't scrape HTML. Use the MediaWiki API, which returns clean wikitext including raw infobox
parameters:

```
https://spelljammer.fandom.com/api.php?action=query&list=categorymembers&cmtitle=Category:Ships&cmlimit=500&format=json
https://spelljammer.fandom.com/api.php?action=parse&page=Hammership&prop=wikitext&format=json
```

The infobox template parameters are the schema — extract those directly rather than parsing
rendered tables.

---

## 2. What's already in the repo (VERIFIED)

### The "downloaded 2e wiki" is `harvester/cmm/`

2,509 HTML files — the Lomion Complete Monster Manual. Subdirs: `possible/`, `templates/`,
`workspace/`. **It is a monster wiki, not a ship source.**

- 160 files mention "spelljammer"; 125 mention spelljamming / wildspace / phlogiston.
- Nearly all of it is **flavor prose inside monster entries** — space owls lairing in wrecks,
  jammer leeches on hulls, beholder orbi as living helms aboard tyrant ships, elven starfly plants
  growing into armadas.
- **Exactly two files contain a real ship stat block** (see §3).
- Partial ship rules: `golefurn.html` (furnace golems self-spelljam — tonnage 1/10, SR 2, MC B);
  `vineinfi.html` (infinity vine quadruples a covered ship's tonnage).

**Grep trap:** `"maneuverability class"` matches 30 files, but in all but the two ship files it is
the creature *flight* MC (A–E) from the movement line. Don't let a naive grep inflate the count.
The reliable ship-block markers are `Power Type`, `Beam Length`, `Maneuver Class`, `Armor Rating`
— each matches exactly the two files below.

### Catalog coverage — `src/data/Full_Catalog.json`

278 books. Record keys: `author`, `monster_keys`, `publish_id`, `setting`, `title`, `year`.

- 12 books tagged `setting: "Spelljammer"` → 57 monsters total.
- Plus MC7 Spelljammer Appendix I (57) and MC9 Spelljammer Appendix II (53).
- **~167 Spelljammer monster entries overall.**
- **SJR1 Lost Ships** (`publish_id` 9280) is tracked — with 10 monsters. It contains roughly
  **19 ship designs, none of which were harvested.** This is the single highest-value target.

### 🐛 Data bug found — worth fixing regardless of the ship work

| Book | Current `setting` | Should be |
|---|---|---|
| MC7 Spelljammer Appendix I | `Advanced Dungeons & Dragons 2nd Edition` | `Spelljammer` |
| MC9 Spelljammer Appendix II | `Planescape` | `Spelljammer` |

110 Spelljammer monsters do not surface under a Spelljammer setting filter because of this.
Fix in `src/data/Full_Catalog.json` (and check whether `data/settings.json` or the harvester
source-of-truth needs the same correction — the JSON under `src/data/` may be generated).

---

## 3. The ship stat block schema (VERIFIED — this is the key artifact)

Extracted from `harvester/cmm/gianspse.html` and `harvester/cmm/rockhopp.html`. This is the
canonical AD&D 2e ship block, and it **matches the Fandom infobox field set** — so both sources
can target one schema.

| Field | Spacesea Giant Galleon | Rock Hopper Skiff |
|---|---|---|
| Built by | Spacesea Giants | Rock hoppers |
| Used by / **Used Primarily by** | Spacesea Giants | Rock hoppers |
| Saves As | Thick stone | Thin wood |
| Tonnage | 60 tons | ⅓ to ½ ton |
| Hull Points | *(absent)* | 1 |
| Ship's Rating | As for helmsman | 1 |
| Crew | 11-20 Giants | 12/1 |
| Power Type | Major or Minor helm | Pedals |
| Standard Armament | Various ballistae | Harpoons |
| Maneuver Class | E | D |
| Cargo | 30 tons | ¼ ton |
| Landing—Land | No | Yes |
| Landing—Water | Yes (it floats!) | No |
| Keel length | 200' | 16' |
| Beam Length | 50' | 6' |
| Armor Rating | 3 | 9 |

### Parser notes — real variance already visible in a sample of two

- **Label drift:** `Used by:` vs `Used Primarily by:`. Match on a normalized prefix, not exact string.
- **Optional fields:** `Hull Points` is present in one record, absent in the other. Don't require it.
- **Vernacular fractions:** `⅓`, `½`, `¼` — Unicode vulgar fractions, and OCR will mangle these.
  Normalize to decimal or a numerator/denominator pair; keep the source string verbatim alongside.
- **Free-text values:** `Ship's Rating: As for helmsman` and `Landing—Water: Yes (it floats!)` are
  not enumerable. Every field needs to tolerate prose.
- **Em-dash in the key:** `Landing—Land` / `Landing—Water` use U+2014, not a hyphen.
- **Two-column layout:** in the source these are laid out as a two-column key/value grid, which is
  why flat text extraction fails (see §4).

**Use these 16 fields as the target schema.** It turns extraction into a *fill-known-fields*
problem: every record validates against a fixed list, and anything missing or unexpected flags for
review automatically. That is what makes a messy OCR source tractable.

---

## 4. The Internet Archive book — format guidance

User has a Spelljammer book from Internet Archive in multiple formats at
`D:\Downloads\spelljammer texts` (not readable from the cloud session; readable locally).

### Why plain text fails

The stat block is a **two-column key/value grid**:

```
Built by: Rock hoppers            Armor Rating: 9
Used Primarily by: Rock hoppers   Saves As: Thin wood
Tonnage: ⅓ to ½ ton               Power Type: Pedals
```

Flat OCR either reads straight across (merging unrelated fields) or reads one column fully then the
other (scrambling pairings), and mangles the fractions. **You need per-word x/y coordinates so you
can cluster on x-position and split the columns deterministically.** That single criterion drives
the ranking:

| Rank | Format | Coords | Verdict |
|---|---|---|---|
| **1** | **PDF** (main scan / `_text.pdf`) | ✅ | **Primary.** Best tooling; pages can also be viewed directly for verification |
| **2** | `_abbyy.gz` (ABBYY XML) / `_djvu.xml` | ✅ word+char | **Fallback.** Highest fidelity — font data, sometimes explicit table regions |
| **3** | `_hocr.html` | ✅ word | Same info, easier parsing (`bbox` in the `title` attr) |
| **4** | `_jp2.zip` / page images | n/a | For the ugliest pages — transcribe via vision |
| **5** | `_djvu.txt` | ❌ | Prose sections only. Useless for stat blocks |
| **6** | EPUB | ❌ | Skip. Reflowed; column structure destroyed |

### Critical first check

**Does the PDF actually have a text layer?** Many IA scans don't. For reference, this repo's own
`1077monsters.pdf` is 14 pages and yields **zero** extractable words — pure image. If the
Spelljammer PDF is the same, the ABBYY/hOCR file becomes the *primary* source, not the backup.

```python
import pymupdf
d = pymupdf.open("path/to/book.pdf")
print(d.page_count)
print(len(d[20].get_text("words")))   # 0 => image-only, use ABBYY/hOCR instead
```

### Column-splitting approach (PDF path)

```python
words = page.get_text("words")   # (x0, y0, x1, y1, word, block, line, word_no)
# cluster words by x0 into two groups at the page midpoint (or via a gap histogram),
# then group each column by y0 into lines, then split each line on ':'
```

### Tooling

- **PyMuPDF works** — verified installed and functioning in the cloud container.
- **`pdfplumber` failed to import** there (a `cryptography`/pyo3 conflict in the system Python).
  May be fine locally; PyMuPDF alone is sufficient either way.

### Recommended hybrid

SJR1 Lost Ships has ~19 designs. At that N, don't over-engineer: run the coordinate-based parser,
then **visually verify every block against the page image** and hand-transcribe the few the parser
mangles. That is faster and more accurate than tuning OCR heuristics for a corpus this small.

---

## 5. Suggested next steps

1. **Verify access locally** — can you reach `spelljammer.fandom.com` and `archive.org`? Can you
   read `D:\Downloads\spelljammer texts`? This determines source priority.
2. **Inventory the IA formats** in that folder; run the text-layer check in §4.
3. **Fix the MC7/MC9 setting tags** (§2). Small, independent, valuable on its own.
4. **Prototype the data model** against the 16-field schema in §3 — decide:
   - ship **class** vs. named **instance** as separate record types
   - how ships link to `Full_Catalog.json` books (reuse `publish_id`)
   - the homebrew/canon provenance flag, decided *before* import
   - whether living vessels (Space Leviathan et al.) are ships, monsters, or both
5. **Test extraction on one ship block** from the IA book and eyeball the quality before
   committing to a pipeline.
6. **Decide the licensing question** if Fandom content is going to be used.

---

## 6. Verified-vs-unverified summary

**Verified directly against files:** everything in §2 and §3; the `1077monsters.pdf` text-layer
finding; PyMuPDF working; the cloud egress blocks.

**Search-derived, not read:** all of §1 (Fandom wiki structure, categories, infobox fields, which
ships exist). The infobox field list is corroborated by the §3 stat blocks, which is a good sign,
but confirm against the live wiki before building to it.

**Not attempted:** any read of the IA book; any code or data change; any scrape.
