# OpenPsalm Song Style Guide

This is the **prescriptive standard** for writing and importing songs. It defines which notation construct to use in each situation so the catalog stays consistent going forward. For the *syntax* of each construct, see [song-toml-format.md](song-toml-format.md); this document is about *choosing correctly* between them.

The rules below are not arbitrary engraving taste — most markings in OpenPsalm carry semantics beyond their printed appearance. Keep this table in mind; the rest of the guide follows from it.

| Marking | Printed effect | Lyric-slot effect | Playback effect |
|---|---|---|---|
| Slur `(` `)` | curved line | notes after the slur start share its syllable | none |
| Beam `[` `]` | joined flags | **same as a slur** — the group takes one syllable | none |
| Dashed slur `-(` `-)` | dashed curve | **none** — every note keeps its own slot | none |
| Tie `~` | tie arc | continuation note takes no syllable | one sustained note |
| Fermata `!` | fermata sign (deduped per staff) | none | none |
| Dynamic `%f` … | marking above top staff (deduped S>A>T>B) | none | sets velocity **for that part's track only** |
| Hairpin `\<` `\>` `\!` | hairpin (deduped S>A>T>B) | none | velocity ramp **for that part's track only** |
| Dedup offset `/±N` | none | none (affects dedup fingerprint only) | none |

---

## 1. Melismas (several notes, one syllable)

A melisma is any group of consecutive notes sung on a single syllable. Every melisma must be marked — an unmarked group will greedily consume one syllable per note and desynchronize the lyrics.

**Rules:**

1. **Melisma of only eighth notes or shorter → beam it**, first note `[`, last note `]`. No slur is needed; under `\autoBeamOff` the beam itself is the melisma sign, as in traditional vocal engraving.
   ```
   d'8[ e'8]            ← two-note melisma, beamed
   e'16[ f'16 g'16]     ← three-note melisma, beamed
   ```
2. **Melisma containing a quarter note or longer → slur it**, first note `(`, last note `)`.
   ```
   d'4( e'4)            ← quarter-note melisma, slurred
   ```
3. **Mixed values → slur the whole melisma, and additionally beam any run of eighths/sixteenths inside it.**
   ```
   g'4( a'8[ g'8])      ← slur spans all three; the eighths are also beamed
   ```
4. **A melisma never crosses a barline with a beam.** If it must continue into the next measure, carry it with the slur (or a tie if the pitch repeats).

**Never:**

- **Never beam notes that carry different syllables.** Instrumental-style "beam by beat" is prohibited: in OpenPsalm a beam *is* a melisma, so beat-beaming two syllables together silently drops one. This is the single most important rule in this guide — when in doubt, don't bar.
- Never slur two notes of the *same pitch* meant as one continuous sound — that is a tie (§3).
- Never mark a single note with both `(` and `)` pointlessly; a one-note group is not a melisma.

## 2. Dashed slurs (verse-dependent melismas)

When the verses *disagree* — one verse sings two syllables on a note pair where another verse melismas through — use a **dashed slur** `-(` `-)` and give every note its own lyric slot. Verses that melisma fill the extra slot with the placeholder syllable `_`:

```
notes  = "... d'8-( e'8-) ..."

[lyrics.1]
text = "... ev -- ery ..."     ← two syllables, one per note
[lyrics.2]
text = "... all _ ..."         ← melisma: second slot is a placeholder
```

**Rules:**

1. Dashed slurs are **only** for genuine verse conflicts. If *every* verse melismas the same notes, use a regular slur or beam (§1) — never a dashed slur with `_` in all verses.
2. The `_` placeholder must sit in exactly the slot the melisma passes through, so every verse has the same syllable count.
3. Do not beam a dashed-slur group; the dashed slur must remain the only marking so each note keeps its slot.

## 3. Ties

Use a tie `~` when the **same pitch** is sustained across a notation boundary (a barline, or where a single longer value can't be written, e.g. dotted figures across beats):

```
g'2~ | g'4 ...      ← one sound held across the barline
```

**Rules:**

1. Tie only identical pitches. Different pitches on one syllable are a melisma (§1), never a tie.
2. Repeated same-pitch notes that are *re-articulated* (each with its own syllable) get neither tie nor slur.
3. The tied-to note takes no lyric slot — do not put a syllable on it.
4. A rest breaks a tie chain; never write a tie into a rest.
5. Prefer a single dotted value inside a measure over a tie where the meter allows it (`g'4.` not `g'4~ g'8`); use ties only where the value or the barline forces them.

## 4. Barlines and measures

1. Every measure is explicit, separated by ` | `, and must sum **exactly** to the time signature's tick count (64ths: quarter = 16). No short or overfull measures anywhere, including the first and last.
2. **Anacrusis (pickup):** pad the front of measure 1 with spacers so it is full: `s2. d'4 |` in 4/4. Never write a short pickup measure.
3. **Repeats are unrolled.** There is no repeat syntax; write the music out in full and let verse lyrics carry the repetition.
4. Mid-song meter changes use `time_sig_changes`; never fake them with over/under-full measures.
5. All parts must have the same measure count and the same per-measure tick totals.

## 5. Dynamics, hairpins, and tempo

1. **Dynamics and hairpins go on *every* part**, at the same musical moment with the same value. Playback velocity is computed per track, so a `%f` written only on the soprano leaves alto/tenor/bass at default volume. The print exporters dedup automatically (soprano > alto > tenor > bass), so the marking is engraved once.
2. Every hairpin `\<` or `\>` must be terminated with `\!` (or superseded by an explicit dynamic) — on every part that opened one.
3. **Tempo/expression spanners (`\rit`, `\accel`, `\atempo`, …) go on the soprano line only**, terminated with `\spanend` on the last covered note. They are song-level, not per-voice.

## 6. Fermatas and staccatos

1. Mark a held/detached moment on **every voice** that sounds at that moment (`g'2!` in all four parts). Per-staff dedup engraves it once above the top staff and once below the bottom staff.
2. Fermatas are engraving-only — they do not lengthen MIDI playback. Do not compensate with longer note values; write the notation as sung from the page.

## 7. Parts, chords, and divisi

1. Standard texture is four parts — soprano, alto, tenor, bass — one voice per part, S+A on staff 1 (treble), T+B on staff 2 (bass).
2. Use chord notation `<c' e'>4` only for true divisi *within* one part. Never encode two independent voices as a chord stream — give them separate parts.
3. In a chord, the first pitch is the primary voice and carries the lyric.
4. **Extra voices get extra parts, not chords.** When an arrangement has more than one voice of a type (e.g. a second bass line answering the choir), add a numbered part (`[parts.Bass2]`) with the same `choral_type` so export filters treat the voices together. Put it on its own staff (`staff_number = 3`) when it is rhythmically independent, or share a staff when it pairs with another voice. A part that only sings some sections fills the rest with spacers (`s1.`), never rests, and marks its first sung event with `@c`/`@e` as usual — song 103 ("Pray All The Time") is the reference example.

## 8. Lyric-slot bookkeeping

1. Every verse (and the chorus/coda) of a part must have exactly one syllable per lyric slot. Balance uneven verses with `_` placeholders (§2), never by altering the notes.
2. When a voice sings the *same words* as another voice but rhythmically offset (e.g. a tenor eighth-note pickup), add a dedup tick offset (`f8/-24`) so the phrase dedups against the other voices. Never use `/±N` to paper over an actually-wrong rhythm.
3. Use per-part `[parts.X.lyrics.N]` only when a part truly sings different words/timing (call-and-response); otherwise share the global `[lyrics.N]`.

## 9. Phrase breaks

1. `phrase_breaks` — **required** at the end of every poetic line. These define both the printed line breaks and the lyrics-page line breaks.
2. `optional_phrase_breaks` — at mid-line caesuras where the typesetter *may* break if a line runs long. Prefer adding these generously on long lines rather than forcing awkward required breaks.
3. `non_breaking_phrase_breaks` — only to split dedup fingerprints (call-and-response echoes) where no visual break is ever acceptable.
4. Never place any break inside a melisma; prefer the tick just after a slur/beam group ends.

## 10. Chorus and coda

1. Mark the chorus start with `@c` on its **first event — including a leading rest** (`r8@c r4 | …`), so pickup rests land on the chorus slide, not the verse slide.
2. Chorus text goes in `[lyrics.chorus]`; it must not be duplicated into the numbered verses.
3. A coda (sung once, after the last verse/chorus) starts at `@e` with text in `[lyrics.coda]`.

## 11. Lyric text style

1. Syllables within a word are separated by ` -- ` (space, two hyphens, space); words by single spaces.
2. Punctuation attaches to the syllable it follows (`know,` `so.`); keep the source hymnal's punctuation and capitalization (capital at each poetic line start).
3. Do not encode line breaks in the text — line structure comes from `phrase_breaks` (§9).

## 12. Converged call-and-response verses

Some gospel songs converge mid-verse: certain poetic lines are **identical in
every stanza** and the engraving prints them once (the shared text continues on
the verse-1 row; rows 2/3 simply end), often with tenor/bass answering on
different words. Song 101 ("He Bore It All") is the reference example — lines
1/3 of each verse are verse-specific, lines 2/4 are a shared call (S/A) against
a shared response (T/B). Recipe:

1. **Never paste the shared line into each `[lyrics.N]`.** Write it once as a
   shared section `[lyrics.sN]` and splice it with `@sN` (see
   [song-toml-format.md](song-toml-format.md#converged-verses-shared-lyric-sections)).
   The verbatim repetition the print collapse keys off is generated at import,
   so a typo can never silently break the convergence.
2. **T/B response lines are `sN` overrides, not per-part verse copies.** Define
   only `[parts.Tenor.lyrics.sN]` / `[parts.Bass.lyrics.sN]` (and the per-part
   chorus): the parts inherit the global verse texts, and the inherited `@sN`
   references expand to the response. The response must consume exactly as many
   lyric slots as the part has notes in that span.
3. **Make the shared line's seams phrase boundaries** (§9) — usually they are
   poetic line ends and belong in `phrase_breaks` anyway. Without a boundary at
   the seam, the verse-specific text and the shared text land in one phrase,
   which differs per verse and never converges.
4. **Echo rhythms that carry the same words need `/±N`** (§8.2): when the bass
   answers on the beat where the tenor echoes off it, offset the bass notes'
   dedup fingerprints (e.g. `e4/+24`) so the single printed response row serves
   both voices.
5. **Do not use `suppress_verses`** for this pattern — suppression hides whole
   rows per part and would also hide the verse-specific lines. Convergence,
   not suppression, produces the print-once behavior.
6. **Verify the layout matrix:** full SATB (one call stack + one response row),
   S+A only (response disappears with its staff), T+B only (verse text and
   response both print), presentation slides (every verse slide shows the
   shared lines in full — convergence is PDF-only), and the web poem (all
   lines in every verse).

## 13. Engraving defaults

These are renderer defaults, not per-song choices; every example or score image published for OpenPsalm (blog figures included) must follow them:

1. **Time signatures are always printed numerically** — `4/4` and `2/2` as numbers, never the common-time `C` or cut-time `¢` symbols.
2. **Aiken shape noteheads are the default.** Every score renders with shape notes unless the user explicitly opts out (`shape_notes=false` on the export endpoints). Round-notehead output is the exception, not the house style.

---

## Import checklist

Before committing a new `song.toml`:

- [ ] Every measure in every part sums to the time signature (parser will rebar/reject otherwise)
- [ ] Pickup padded with spacers; repeats unrolled
- [ ] Every melisma beamed (all short values) or slurred (contains quarter+); no beat-beaming anywhere
- [ ] Verse-conflicting melismas use dashed slurs + `_` placeholders
- [ ] Same-pitch sustains tied, not slurred; no syllable on tied-to notes
- [ ] Syllable count per verse = lyric slots per part, for every part
- [ ] Dynamics/hairpins duplicated on all parts; tempo spanners on soprano only
- [ ] Fermatas/staccatos on every sounding voice
- [ ] `@c` on the chorus's first event (rest included); `@e` for a coda
- [ ] `phrase_breaks` at every poetic line end; optional breaks at caesuras
- [ ] Lines identical in every verse written once as `[lyrics.sN]` shared sections (§12), never pasted per verse
- [ ] Render the PDF and play the MIDI: lyrics aligned, no swallowed syllables, all voices audible at intended volume
