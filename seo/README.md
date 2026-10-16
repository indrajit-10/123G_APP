# 123greetings sitemap — upcoming major events update

Additive update to `https://www.123greetings.com/sitemap.xml`.
**No existing entry is modified, reordered, or removed.**

| File | What it is |
|---|---|
| `sitemap.xml` | Drop-in replacement: the live 165 URLs byte-for-byte + 29 new ones (194 total) |
| `sitemap-additions.xml` | Just the 29 new `<url>` blocks, to paste before `</urlset>` |
| `sitemap-additions-stage2.xml` | 53 subcategory blocks held back for later (see "Rotating window") |

## What was added (29 URLs)

Ten upcoming major event hubs, plus full subcategory depth for Halloween
(the next flagship, 15 days out):

| Event | Date | Days out | Hub priority | Subcats added |
|---|---|---|---|---|
| Halloween | Oct 31, 2026 | 15 | `0.9` | 19 @ `0.8` |
| Veterans Day | Nov 11, 2026 | 26 | `0.8` | — |
| Children's Day | Nov 14, 2026 | 29 | `0.8` | — |
| Thanksgiving | Nov 26, 2026 | 41 | `0.8` | — |
| Black Friday | Nov 27, 2026 | 42 | `0.8` | — |
| Hanukkah | Dec 4–12, 2026 | 49 | `0.8` | — |
| Christmas | Dec 25, 2026 | 70 | `0.8` | — |
| Kwanzaa | Dec 26, 2026 | 71 | `0.8` | — |
| New Year | Jan 1, 2027 | 77 | `0.8` | — |
| Valentine's Day | Feb 14, 2027 | 121 | `0.8` | — |

All 29 URLs were verified to return HTTP 200 before inclusion.

## House conventions followed

Priorities were not invented — they mirror what the live file already does:

- Flagship / nearest big event: hub `0.9`, subcategories `0.8` (how Diwali is tiered today)
- Every other live event: hub `0.8`, subcategories `0.7` (Sweetest Day, Boss Day, Dussehra, Durga Puja)
- `<changefreq>daily</changefreq>` on event pages; tag order is `loc, priority, lastmod, changefreq`

## Rotating window

The `/events/` section of the live sitemap is a **rotating window**, not a full
catalogue: in mid-October 2026 it carried only Boss Day, Sweetest Day, Dussehra,
Durga Puja, Diwali and a few food/novelty days. Halloween, Thanksgiving,
Christmas and New Year were absent entirely even though the pages exist and
rank seasonally — that gap is what this update closes.

Because of that design, subcategory depth for Thanksgiving / Christmas /
New Year / Valentine's is parked in `sitemap-additions-stage2.xml` rather than
shipped now. Paste each block when its event enters the live window, and when
you promote an event to flagship raise its hub to `0.9` and its subcategories
to `0.8`.

## Two pre-existing issues (left alone — additive change only)

1. **Uniform `lastmod`.** Every live entry carries the same timestamp, bumped
   on each rebuild. Search engines discount a `lastmod` that cannot be trusted;
   ideally it reflects the last real content change per URL.
2. **Timezone offset.** The live file stamps `-08:00` on October dates, but
   US Pacific is `-07:00` until Nov 1. New entries match the existing format
   for file consistency; worth fixing in the generator.

Neither is changed here. Both need a generator fix rather than a hand edit.
