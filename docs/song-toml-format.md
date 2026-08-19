# OpenPsalm Song TOML Format

This document describes the `.toml` file format used to define songs in OpenPsalm. Every song lives in a directory named for its integer ID, holding a `song.toml` — `{N}/song.toml` in the [OP-songs](https://github.com/squinky86/OP-songs) repo, which OpenPsalm mounts as a submodule at `songs/`, so the same file is `songs/{N}/song.toml` from inside OpenPsalm.

This is the *syntax* reference. For *which* construct to use when (melismas, beaming, slur vs. tie, phrase-break placement, …), follow the [Song Style Guide](song-style-guide.md).

---

## Top-level Fields

```toml
title = "Jesus Loves Me"
subtitle = "Optional subtitle or tune name"
verse_count = 3
key_signature = "Eb"
time_sig_numerator = 4
time_sig_denominator = 4
tempo_bpm = 120
phrase_breaks = ["4:64", "8:64", "12:64"]
optional_phrase_breaks = ["2:64", "6:64", "10:64"]
copyrights = [
  "Words: Anna B. Warner, 1860 (public domain)",
  "Music: 'untitled' by William B. Bradbury, 1862 (public domain)",
  "Obtained from the Open Hymnal Project, 2006 (public domain)",
  "Arrangement by OpenPsalm, 2026 and released under the CC-BY 4.0 license",
]
```

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | yes | Full song title |
| `active` | boolean | no | If `false`, song is not loaded into the database. Defaults to `true`. |
| `subtitle` | string | no | Subtitle, meter, or tune name |
| `verse_count` | integer | yes | Number of numbered verses |
| `default_verses` | integer array | no | Verse numbers ticked in the song page's "Verses" column when the page first loads. Omit it (the usual case) and every verse starts selected. See [Default Verse Selection](#default-verse-selection). |
| `key_signature` | string | yes | Key: `C G D A E B F# F Bb Eb Ab` (major) or append `m` for minor |
| `time_sig_numerator` | integer | yes | Beats per measure |
| `time_sig_denominator` | integer | yes | Beat unit (2, 4, 8, etc.) |
| `time_sig_changes` | array of tables | no | Mid-song time signature changes (see [Time Signature Changes](#time-signature-changes)) |
| `tempo_bpm` | integer | yes | Quarter-note BPM used for MIDI/MP3 export |
| `phrase_breaks` | string array | no | `"M:T"` entries that break both the poetry line and the sheet music/slide layout (see [Phrase Breaks](#phrase-breaks)) |
| `optional_phrase_breaks` | string array | no | `"M:T"` entries that break only the sheet music/slide layout — poetry lines flow through them unbroken (see [Phrase Breaks](#phrase-breaks)) |
| `non_breaking_phrase_breaks` | string array | no | `"M:T"` entries used only as phrase boundaries for cross-voice lyric deduplication — they never break the poetry line or the layout (see [Phrase Breaks](#phrase-breaks)) |
| `copyrights` | string array | no | Copyright lines displayed in the PDF footer (see [Copyright Format](#copyright-format)) |
| `commentary` | string | no | HTML-formatted text displayed below the sheet music on the song's detail page (historical context, musical notes, etc.) |
| `converge_verses` | boolean | no | When `true`, any phrase sung identically by every verse is printed once in the PDF (kept in verse 1, skipped in later verses) instead of stacked in each numbered row. Defaults to `true` when the song uses shared lyric sections (`[lyrics.sN]`), otherwise `false` — ordinary hymns whose verses intentionally repeat a tag line keep the repeat (see [Converged Verses](#converged-verses-shared-lyric-sections)). |

---

## Default Verse Selection

A long hymn is often sung from only its first few stanzas, with the rest printed
for completeness. `default_verses` lets the song say which ones the page starts
with:

```toml
verse_count = 8
default_verses = [1, 2, 3]
```

Verses 1–3 load ticked; 4–8 load unticked. It only sets the **initial state** of
the checkboxes — the reader can tick any of them, and everything downstream
(sheet music, lyrics, MIDI/MP3, presentation) follows the boxes as it always
has.

Notes:

- Omitting the key means "all verses", which is what every song did before the
  key existed. Listing every verse is the same thing.
- Numbers outside `1..=verse_count` are dropped, and the list is sorted and
  deduplicated, so `[3, 1, 3, 99]` on a 4-verse song stores as verses 1 and 3.
- A list that selects nothing at all is treated as "all verses" rather than
  rendering a page with no verse chosen.
- The export API is unaffected: `/api/songs/{n}/export/...` with no `verses`
  parameter still returns every verse. The song page always sends its checkbox
  state explicitly, and a link shared from the page carries `verses=` whenever
  the selection differs from the song's default.
- A translation overlay inherits the base song's `default_verses` unless it
  states its own.

---

## Time Signature Changes

Some songs temporarily change time signature for a few measures before reverting to the main time signature. Use the `time_sig_changes` array to define these mid-song changes.

Each entry specifies:

| Field | Type | Description |
|---|---|---|
| `measure` | integer | 1-based measure number where the change begins |
| `numerator` | integer | New beats per measure |
| `denominator` | integer | New beat unit (2, 4, 8, etc.) |
| `duration` | integer | Number of consecutive measures the change applies to |

After the specified `duration` measures, the time signature automatically reverts to the song's default (`time_sig_numerator`/`time_sig_denominator`).

### Example

A song in 4/4 that switches to 3/4 for measures 9–10, then back to 4/4:

```toml
time_sig_numerator = 4
time_sig_denominator = 4

[[time_sig_changes]]
measure = 9
numerator = 3
denominator = 4
duration = 2
```

Measures 1–8 are 4/4, measures 9–10 are 3/4, and measure 11 onward reverts to 4/4. The notes in measures 9–10 must sum to 3/4 (12 ticks) rather than 4/4 (16 ticks).

### Multiple Changes

Multiple time signature changes can coexist:

```toml
[[time_sig_changes]]
measure = 5
numerator = 3
denominator = 4
duration = 2

[[time_sig_changes]]
measure = 12
numerator = 2
denominator = 4
duration = 1
```

### Export Behavior

- **MusicXML**: A `<time>` element is emitted in the `<attributes>` block of each measure where the time signature differs from the previous measure.
- **MIDI**: A time signature meta event (0xFF 0x58) is emitted at the tick position of each time signature change.
- **LilyPond**: A `\time N/D` command is emitted inline before the first note of each measure where the time signature changes.

### Validation

During import, each measure is validated against its effective time signature. If a measure's notes don't sum to the expected tick count for its time signature, the seeder will report an error with the correct expected time signature (not just the song default).

---

## Parts

Each voice is defined in a `[parts.Name]` section. The name (`Soprano`, `Alto`, `Tenor`, `Bass`, or any custom name) becomes the variable name in the LilyPond export.

```toml
[parts.Soprano]
choral_type = "soprano"
clef = "treble"
staff_number = 1
notes = """
bes'4 g'4 g'4 f'4 | g'4 bes'4 bes'2 | ...
"""
```

| Field | Values | Description |
|---|---|---|
| `choral_type` | `soprano alto tenor bass` | Used for filtering voices on export. Several parts may share one choral type (e.g. `Bass` and `Bass2` both `"bass"`); export filters then include or exclude them together |
| `clef` | `treble bass treble_8` | Staff clef; `treble_8` for tenor octave clef |
| `staff_number` | positive integer | Parts sharing a staff number are combined onto one staff; staves are ordered by number. The standard layout is S+A on staff 1, T+B on staff 2, but any number of staves and voices is allowed (e.g. an independent second bass voice alone on staff 3). Parts alone on a staff keep free stem directions; parts sharing a staff get opposing stems (S/T up, A/B down; duplicates fall back to the next free LilyPond voice) |
| `notes` | string | Note stream in OpenPsalm note notation (see below) |
| `suppress_verses` | int array | Verse numbers to omit from the PDF/LilyPond output for this part (see [Verse Suppression](#verse-suppression)) |
| `suppress_verses_when` | string array | Choral types (lowercase) whose presence triggers the suppression (see [Verse Suppression](#verse-suppression)) |
| `splice_lyrics_into` | string | Choral type of the voice whose verse row absorbs this part's echo syllables (see [Echo Lyric Splice](#echo-lyric-splice-splice_lyrics_into)) |

### Verse Suppression

In call-and-response songs, a responding voice (e.g. tenor/bass) may repeat the same verse lyrics that are already visible in the calling voice (e.g. soprano/alto). Printing all verses again under the responding voice clutters the page. The `suppress_verses` and `suppress_verses_when` fields work together to conditionally hide extra verse lyrics:

```toml
[parts.Tenor]
choral_type = "tenor"
suppress_verses = [2, 3, 4, 5]
suppress_verses_when = ["soprano", "alto"]
```

**Behavior:** When any choral type listed in `suppress_verses_when` is among the selected voices for export, the verse numbers in `suppress_verses` are omitted from this part's lyric output. The first verse is still shown so the singer knows what notes to sing; verses 2-5 are implied since the reader can see them on the soprano/alto staff above.

**Rules:**
- Both fields must be present for suppression to take effect. If either is missing, all verses are printed normally.
- The choral type comparison is case-insensitive.
- Suppression only affects verse lyrics; chorus lyrics are unaffected.
- If none of the `suppress_verses_when` types are present (e.g. the user exports only tenor and bass), all verses are printed normally.

### Echo Lyric Splice (`splice_lyrics_into`)

Many hymns answer a verse line with an **echo**: the melody holds a long note
while a lower voice replies with a short tag. The echo is part of the same
numbered verse, and hymnals print it that way — inline in the melody's verse
row, positioned under the notes that actually sing it — not as a second stack of
lyric rows below the staff.

`splice_lyrics_into` goes on the *echoing* part and names the `choral_type` of
the voice whose verse row should absorb the tag. The echo may close the line
(song 13, "A Beautiful Life", whose alto answers `(the best I can)` at the end
of the verse) or fall in the middle of it (song 221, "Wonderful Jesus", whose
bass answers `(so long,)` under the soprano's held note and then hands the row
back for `But the soul…`):

```toml
[parts.Alto]
choral_type = "alto"
clef = "treble"
staff_number = 1
splice_lyrics_into = "soprano"
notes = """..."""

[parts.Alto.lyrics.1]
text = "... And so I'll do the best I can, (the best I can)."
```

**Behavior:** for each verse, the alto's surviving syllables are grouped into
runs and each run is moved into the soprano's verse row at the point it belongs,
instead of being printed as the alto's own lyric rows. In the LilyPond/PDF
output a run is emitted behind a `\set associatedVoice = "Alto"` handoff and
switched back afterwards, so the echo words sit under the alto's notes while
remaining part of the numbered verse row, and the soprano's row picks up again
where it left off. The alto's own row for that verse is suppressed. A verse may
carry several echoes; each is handed off and handed back independently.

**Rules:**
- The value is a **choral type** (`"soprano"`, `"bass"`, …), not a part name,
  and is matched case-insensitively.
- **Every echo syllable needs a target syllable before it.** Each surviving
  source syllable is attached behind the last target syllable that starts
  strictly before it. A source syllable with no target syllable before it — an
  echo that opens the line — has nothing to hand off from, and that verse falls
  back to its own lyric row.
- **The two voices may not sing different text at the same moment.** If a
  surviving source syllable lands on the exact tick of a surviving target
  syllable, the texts are simultaneous rather than an echo, and one row cannot
  express them; that verse falls back to its own lyric row.
- **A mid-verse echo needs at least two syllables.** The switch back to the
  target voice is written one syllable early (LilyPond applies an
  `associatedVoice` change starting with the *second* syllable after the
  `\set`), so a one-syllable run that still has target text after it has nowhere
  to put the switch-back and falls back to its own row. A one-syllable echo that
  *closes* the verse is fine, since nothing follows it.
- All of these decisions are per verse, and all-or-nothing within a verse: verse
  1 may splice while verse 2 does not, but a verse never splices some of its
  echoes and prints the rest on a separate row.
- **The echo must be its own phrase for dedup purposes.** A source part whose
  text merely repeats the target's is deduplicated phrase by phrase, and only
  the surviving syllables are spliced. An echo buried inside a phrase that
  otherwise matches the target keeps the whole phrase alive, which then trips
  the same-tick rule above. Put a boundary at the echo's start — usually a
  [`non_breaking_phrase_breaks`](#phrase-breaks) entry, as song 221 does with
  `"5:16"` — so the tag forms a phrase of its own.
- The splice only applies when the target choral type is among the exported
  voices. Export the alto alone and it prints its full text on its own row.
- Verses hidden by [Verse Suppression](#verse-suppression) on either part, or
  with no surviving syllables on either side after phrase/verse deduplication,
  are left alone.
- Splicing changes only *where* syllables print. The source part still needs one
  syllable per lyric slot of its own notes, so the echo text is authored as
  ordinary [per-part lyrics](#per-part-lyrics) covering the whole verse.
- MusicXML export follows the same intent with a simpler rule: secondary voices
  normally drop their lyrics so SATB text isn't duplicated, but a spliced part
  keeps its syllables at every onset where the target voice has no
  lyric-bearing note.
- Presentation/PPTX slides do not implement the splice — the echo prints on its
  own lyric row there (phrases identical across parts are still deduplicated).
  The song page's lyrics tab reads the first lyric-bearing part (normally the
  target voice), so an echo tag that appears only in the source part's text will
  not show up there.

### Note Notation

OpenPsalm uses a LilyPond-inspired notation. Notes are space-separated within a measure; measures are separated by ` | `.

**Pitch**: `{step}{accidental}{octave_marks}{duration}{dots}{flags}`

| Component | Syntax | Examples |
|---|---|---|
| Step | `c d e f g a b` (lowercase) | `c`, `g`, `b` |
| Accidental | `is` = sharp, `es` = flat, `isis` = double sharp, `eses` = double flat | `fis` = F♯, `bes` = B♭, `fisis` = F𝄪, `beses` = B𝄫 |
| Octave marks | `'` raises one octave above base (3), `,` lowers one | `c'` = C4, `c''` = C5, `c,` = C2 |
| Base octave | No marks = octave 3 (C3) | `g` = G3 |
| Duration | `1 2 4 8 16` | `4` = quarter, `8` = eighth |
| Dots | `.` after duration | `2.` = dotted half |
| Flags | `(` slur start, `)` slur end, `-(` dashed slur start, `-)` dashed slur end, `!` fermata | `d'8-( e'8-)` |

**Double accidentals:** Suffix the step with `isis` (double sharp) or `eses`
(double flat). Unlike LilyPond's shorthand spellings, OpenPsalm does *not* elide
the repeated vowel — write `aeses` and `eeses`, never `asas`/`eses` for A𝄫/E𝄫,
just as single flats are written `aes`/`ees` and never `as`/`es`. An
unrecognized accidental suffix parses as a natural without warning, so a typo
here is silent.

Use a double accidental only where the harmony genuinely calls for it — a
lowered/raised scale degree in an already-flat/sharp key (song 219 is in D♭
major, where the tenor's flat 6 of the flat 6 is `beses`, not `a`). Enharmonic
respelling costs nothing in the score and is easier to sing, but the double
accidental is correct when the voice leading demands that letter name.
Transposing exports (`?key=`) respell every pitch with single accidentals, so
the double accidental only survives at the song's written key.

**Special note types:**

| Token | Meaning |
|---|---|
| `r4`, `r2`, etc. | Rest of the given duration |
| `s4`, `s2`, etc. | Spacer (invisible rest; used for anacrusis padding) |
| `<c' e'>4` | Chord (multiple pitches, one duration) |
| `~` after a note | Tie to next note of same pitch |
| `{3 c'8 d'8 e'8 }` | Tuplet (triplet by default); see below |

**Divisi (Chords):** For parts that split into multiple notes (divisi), use the chord notation `<...>` to group multiple pitches into a single rhythmic event. All pitches inside the brackets share the single duration following the closing bracket:
```
<c' e'>4        ← C4 and E4 played together for a quarter note
<e' g' c''>2.   ← E4, G4, and C5 played together for a dotted half note
```
When lyrics are present, only the *first* note in the chord (the primary voice) will consume a lyric syllable. The exported PDF, presentation, and MusicXML will correctly group these as a simultaneous chord or divisi.

**Anacrusis (pickup measure):** The first measure must sum to the full time signature in ticks. Pad the front with spacers to fill the gap:
```
s2. d'4 |   ← 3-beat spacer + 1-beat pickup in 4/4
s2 d'4  |   ← 2-beat spacer + 1-beat pickup in 3/4
```

**Slurs (melismas):** Mark the first and last note of a slurred group:
```
d'8( e'8)         ← 2-note slur
e'16( f'16 g'8)   ← 3-note slur
```
Slur-continuation notes (all but the slur-start) do not consume a lyric slot.

**Dashed Slurs (Optional Melismas):** Use `-(` and `-)` for a visual slur that does **not** force a lyric skip. This is used in hymns where one verse has a melisma (using `_` in the lyrics) but other verses have separate syllables for the same notes:
```
d'8-( e'8-)       ← Visual slur only
```
Unlike regular slurs, every note in a dashed slur group still consumes a lyric slot by default.

**Tuplets (triplets, quintuplets, etc.):** Wrap a group of notes in `{N ... }` to mark
them as an N-in-the-space-of-M tuplet. `{3 ... }` is the common triplet (3 notes played
in the time of 2 of the same value). Use the explicit `{N:M ... }` form for other ratios
(e.g. `{5:4 ... }` for five in the space of four). When only `N` is given, `M` defaults
to the largest power of two less than `N` (3→2, 5→4, 6→4, 7→4).

```
{3 c'8 d'8 e'8 }          ← eighth-note triplet (3 eighths in the time of 2)
{3 c'4 d'4 e'4 }          ← quarter-note triplet (3 quarters in the time of 2)
{5:4 c'16 d'16 e'16 f'16 g'16 }   ← quintuplet
```

The notated durations inside the braces are the *notated* values; their played length is
scaled by `M/N`. Ties, slurs, beams, and lyrics all work normally inside a tuplet. The
group may not span a bar line.

Exports:
- **MusicXML**: Each note carries `<time-modification>` plus `<notations><tuplet>` start/stop markers.
- **LilyPond (PDF and presentation)**: Emitted as `\tuplet N/M { ... }`.
- **MIDI**: Each note's duration is scaled by `M/N` ticks (e.g. an eighth-note triplet member is 2/3 of a normal eighth).

**Fermatas:** Append `!` to the duration:
```
g'2!   ← half note with fermata
s4!    ← spacer with fermata (silence + fermata)
```

**Staccatos:** Append `-.` to the note (distinct from the dashed-slur markers `-(` / `-)`):
```
c''8-.   ← staccato eighth note
c''4@c-. ← staccato may follow a chorus marker
```
Staccatos are deduplicated per staff exactly like fermatas: a staccato on the same beat in soprano and alto renders once above the top staff; on the same beat in tenor and bass it renders once below the bottom staff (via `\voiceOne`/`\voiceTwo` direction).

**Dynamics:** Append `%name` to the duration to attach a dynamic marking above the note. The marking appears above the uppermost staff in the exported score.

| Suffix | Symbol | Meaning |
|---|---|---|
| `%ppp` | 𝆏𝆏𝆏 | pianississimo (very very soft) |
| `%pp` | 𝆏𝆏 | pianissimo (very soft) |
| `%p` | 𝆏 | piano (soft) |
| `%mp` | 𝆐𝆏 | mezzo-piano (moderately soft) |
| `%mf` | 𝆐𝆑 | mezzo-forte (moderately loud) |
| `%f` | 𝆑 | forte (loud) |
| `%ff` | 𝆑𝆑 | fortissimo (very loud) |
| `%fff` | 𝆑𝆑𝆑 | fortississimo (very very loud) |
| `%fp` | 𝆑𝆏 | forte-piano (loud then immediately soft) |
| `%sfz` | sfz | sforzando |

The dynamic applies to all subsequent notes until another dynamic marking overrides it.

```
c''4.%p b'8    ← piano marking above the dotted quarter; b'8 continues at piano
g'2%mf         ← mezzo-forte marking above this half note
```

The `%` suffix may be combined with other flags. It must appear after the duration digits but may appear before or after fermata (`!`), tie (`~`), slur (`(`/`)`), and beam (`[`/`]`) suffixes:
```
c''4%p!   ← piano dynamic + fermata
f'8%f(    ← forte dynamic + slur start
```

**Export behavior:**
- **LilyPond**: Emitted as `^\markup { \dynamic name }` (above-staff placement).
- **MusicXML**: Emitted as a `<direction placement="above"><dynamics>` element before the note.
- **MIDI**: Sets the note-on velocity for the marked note and all subsequent notes on that track until a new dynamic appears (ppp=20 … fff=127).

**Hairpins (Crescendo / Diminuendo):** Use `\<`, `\>`, and `\!` to create dynamic hairpins.
- `\<` : Starts a crescendo on the marked note.
- `\>` : Starts a diminuendo on the marked note.
- `\!` : Ends the current hairpin on the marked note.

These flags behave similarly to LilyPond syntax and can be combined with other note flags. Since the TOML note data uses raw strings (`"""`), the backslash does not need to be escaped.
```
c''4%p\<   ← piano dynamic + crescendo start
d''4       ← continues crescendo
e''4\!%f   ← hairpin ends, forte dynamic begins
```

**Tempo / Expression Spanners:** Write these on the soprano line to mark gradual tempo changes. The label is printed above the staff, with a dashed extender drawn automatically when the span covers more than one note.

| Marker | Meaning |
|---|---|
| `\rit`, `\ritard`, `\rall` | Gradual slowdown |
| `\accel`, `\string` | Gradual speedup |
| `\atempo` | Single-note label restoring the song's `tempo_bpm` |
| `\spanend` | Terminates the current spanner on the marked note |

```
c''4\rit d''4 e''4 f''4\spanend   ← "rit." spans four notes
g''2\spanend\atempo               ← end the span and restore tempo here
```

MIDI/MP3 export interpolates the BPM across each span (a slowdown targets ~0.6× and a speedup ~1.4× of the active BPM, unless a `\spanend\atempo` restores it).

**Deduplication Tick Offset:** Append `/N` (where N is a signed integer, in internal ticks) to shift a note's position **for lyric deduplication only**. This does not affect playback, MIDI timing, LilyPond layout, or MusicXML output — it only adjusts the tick timestamp used when the exporter checks whether two voices are singing identical phrases.

Internal tick values: whole = 192, half = 96, quarter = 48, eighth = 24, sixteenth = 12.

```
a4/+24(   ← quarter A, slur start; dedup tick shifted +24 (one eighth forward)
f8/-48    ← eighth F; dedup tick shifted -48 (one quarter backward)
```

**When to use:** Some voices (typically tenor) sing the same lyrics as soprano/alto/bass but with a slightly different rhythmic subdivision — for example, an eighth-note pickup before the beat where the other voices begin. Without an offset, the dedup fingerprinter sees different cumulative ticks and prints the lyrics again. Adding `/+N` or `/-N` to the pickup note shifts its fingerprint tick to match the other voices, allowing deduplication to correctly suppress the duplicate.

**Combining with other flags:** The `/N` suffix is extracted before beam, slur, tie, fermata, and dynamic flags, so any order of the remaining suffixes is valid:
```
a4/+24(     ← dedup offset +24, slur start
f8/-24[     ← dedup offset -24, beam start
g'4/+48%mf  ← dedup offset +48, mezzo-forte dynamic
```

**Syntax rules:**
- The `/` must immediately follow the duration (and dot, if any): `a4./-24` is valid, `a4 /-24` is not.
- N may have an explicit `+` sign or be bare positive: `/24` and `/+24` are equivalent.
- N = 0 is a no-op but valid for documentation purposes.
- Fractional values are not supported; use the nearest integer tick value.

---

## Lyrics

Numbered verses use `[lyrics.N]` sections; the chorus uses `[lyrics.chorus]`; an optional coda (sung once after the final verse/chorus) uses `[lyrics.coda]`.

```toml
[lyrics.1]
text = "Je -- sus loves me! This I know, For the Bi -- ble tells me so."

[lyrics.chorus]
text = "Yes, Je -- sus loves me! The Bi -- ble tells me so."
```

A coda's first note is marked with `@e` in the note stream (analogous to the `@c` chorus marker); the measures from that point to the end of the song form the coda section, which the `include_coda` export option can strip.

### Lyric Text Format

- Words are separated by spaces.
- Multi-syllable words use ` -- ` (space-dash-dash-space) between syllables.
- Each whitespace token (excluding `--` connectors) is one syllable that consumes one lyric slot.
- There must be exactly as many syllables as lyric slots in the note stream. (Lyric slots = pitched, non-spacer, non-rest notes; slur-continuation notes do not count.)

**Syllable slot counting example:**
```
"Je -- sus loves me!"  →  Je | sus | loves | me!  =  4 syllables (4 slots)
"Bi -- ble"            →  Bi | ble              =  2 syllables
```

### Verse/Chorus Layout

The seeder assigns each syllable to a note slot by index. Verse syllables start at slot 0. Chorus syllables start at slot `max_verse_length` (the syllable count of the longest verse). This means chorus lyrics share the same notes as the final verses — appropriate for hymns where the verse and chorus melody uses the same notes.

### Chorus Pickups (Rests)

If the chorus phrase begins with a pickup (anacrusis) that consists of rests before the first sung note (e.g. `r8 r4 | r8 c8 c8`), you should manually denote the start of the chorus by appending `@c` to the rest that begins the chorus. 

Example: `r8@c r4 | r8 c8 c8`

This guarantees that the exporter correctly ends the verses slide and pushes the leading rests onto the beginning of the chorus slide. If this marker is omitted, the exporter will fall back to using the first *pitched note* that carries a chorus lyric, which might incorrectly leave those rests at the end of the verses slide.

### Per-Part Lyrics

By default, all parts share the same global `[lyrics.N]` sections. When a part needs different lyrics (e.g. a call-and-response chorus where tenor/bass sing on different beats than soprano/alto), you can define per-part lyrics inside the part section:

```toml
[parts.Tenor]
choral_type = "tenor"
clef = "bass"
staff_number = 2
notes = """..."""

[parts.Tenor.lyrics.1]
text = "Al -- ter -- nate verse text for ten -- or..."

[parts.Tenor.lyrics.chorus]
text = "Al -- ter -- nate cho -- rus text for ten -- or..."
```

**Rules:**
- Per-part lyrics are **merged per key** with the global lyrics for that part. A part-specific key (e.g. `[parts.Tenor.lyrics.chorus]` or `[parts.Tenor.lyrics.2]`) overrides the global key of the same name, but any global key the part does *not* redefine is still used. So a part can override just its chorus while keeping the global verses, or override one verse while inheriting the rest.
- Other parts without per-part lyrics continue to use global `[lyrics.N]` sections.
- The syllable count must match the part's own lyric slots (which may differ from other parts if the note pattern differs).

### Converged Verses (Shared Lyric Sections)

Some hymns have a call-and-response line *inside* the verse that is identical in
every stanza — the engraving prints it **once** rather than stacking the same
words in every numbered row. Author this with a **shared lyric section**: write
the line once in `[lyrics.sN]` and splice it into each verse with a standalone
`@sN` token (song 101, "He Bore It All", is the reference example):

```toml
[lyrics.1]
text = "My pre -- cious Sav -- ior suf -- fered pain and ag -- o -- ny, @s1 ..."

[lyrics.2]
text = "They placed a crown of thorns up -- on my Sav -- ior’s head, @s1 ..."

[lyrics.s1]
text = "He bore it all that I might live;"
```

**Rules:**
- `@sN` must be a standalone whitespace-delimited token; it expands to the full
  syllable sequence of `[lyrics.sN]` at import time. An unknown reference fails
  the seed with an error. Shared sections may not reference other shared
  sections.
- Shared sections merge per part like any other lyric key: a part can override
  `[parts.Tenor.lyrics.s1]` with its *response* text while inheriting the
  global verse texts untouched — the inherited `@s1` references then expand to
  the tenor's own line. This is the natural encoding for call-and-response:
  T/B need no per-part verse copies at all, only their `sN` overrides (plus a
  per-part chorus if the responses continue there).
- Expansion happens at import, so MusicXML, MIDI, the JSON API, presentation
  slides, and the web lyric display all keep the complete per-verse text.

**Print behavior (LilyPond/PDF only):** using shared sections defaults the
song's `converge_verses` flag on (an explicit `converge_verses = false` still
wins). For each part, every non-suppressed verse row is split into phrases at
the phrase-break boundaries; when *all* of that part's verses sing an identical
phrase (same syllables at the same ticks), it is kept in the lowest-numbered
verse row and replaced by `\skip` in the others. In a system containing only
converged content the lower rows have no syllables, so LilyPond collapses the
stack to a single printed row. The shared line's seams must therefore be phrase
boundaries — poetic line ends usually make them so naturally (use
`non_breaking_phrase_breaks` if a seam shouldn't force a visual break).

**Why opt-in:** many hymns intentionally repeat a tag line in every stanza
(e.g. "and crown Him Lord of all", "Just as I am"). Those must keep printing the
repeat in each verse, so convergence never activates unless the song uses shared
sections or sets `converge_verses = true` (the flag alone also works for songs
whose verses carry verbatim-identical text without shared sections).

Convergence composes with cross-voice deduplication (see [Per-Part Lyrics](#per-part-lyrics)):
verse-specific lines that another voice already sings collapse across voices,
while the shared call/response line collapses across verses, together yielding a
single printed row per shared span.

---

## Phrase Breaks

OpenPsalm has three phrase break fields that control where lines may wrap in different contexts:

| Field | Lyrics page | Sheet music / slides | Lyric dedup |
|---|---|---|---|
| `phrase_breaks` | breaks here | breaks here | phrase boundary |
| `optional_phrase_breaks` | breaks here | breaks here | phrase boundary |
| `non_breaking_phrase_breaks` | flows through | flows through | phrase boundary |

Use `phrase_breaks` for breaks that must align with the ends of poetic lines (e.g. the end of "A mighty fortress is our God, a bulwark never failing;"). Use `optional_phrase_breaks` for additional break opportunities that the typesetter may use to optimize line spacing — for example, the mid-phrase caesura after "A mighty fortress is our God,"; these also break the printed lyrics on the song page. Use `non_breaking_phrase_breaks` when a phrase boundary is needed only for cross-voice lyric deduplication (splitting the fingerprinted phrases so a call-and-response echo dedups correctly) without allowing a visual break there.

All three fields share the same `"M:T"` string format and all are optional. The sheet music and slide exporters break lines at the first two lists; all three lists together define the phrase boundaries used for lyric deduplication; the lyrics page breaks lines at the first two lists.

### Format

Each entry is a string `"M:T"` where:
- **M** — 1-based measure number
- **T** — cumulative tick count within that measure, in 64th-note units (1 tick = 1/64 note)

Common tick values:
| Duration | Ticks |
|---|---|
| Whole note | 64 |
| Half note | 32 |
| Quarter note | 16 |
| Eighth note | 8 |
| 16th note | 4 |
| 32nd note | 2 |
| 64th note | 1 |

A 4/4 measure has 64 ticks total. A 3/4 measure has 48 ticks. A 2/4 measure has 32 ticks.

The break fires after the note that brings cumulative ticks in measure M to exactly T.

### How It Works

When "Phrased Notation" is enabled (on by default in the UI):

- **All regular barlines** receive `\noBreak`, preventing LilyPond from breaking at any arbitrary measure boundary.
- **End-of-measure breaks** (`T = measure_ticks`): the barline is left without `\noBreak`, marking it as a permitted break point.
- **Mid-measure breaks** (`T < measure_ticks`): an invisible barline `\bar "" \break` is inserted after the note at tick T, splitting the measure at the phrase boundary.
- LilyPond's optimizer places line breaks only at these marked positions.

### Computing Phrase Breaks

A break at `"M:T"` means: the phrase ends with the note whose cumulative ticks within measure M equal T, and the next phrase begins with the following note.

**Example** — "Jesus Loves Me" in 4/4, 4 quarter notes per measure (each quarter = 16 ticks, so 4 quarters = 64 ticks = end of measure):
```
Measures 1–4:   Je -- sus loves me! This I know,
Measures 5–8:   For the Bi -- ble tells me so.
Measures 9–12:  Lit -- tle ones to Him be -- long;
Measures 13–16: They are weak, but He is strong.
```
Each phrase ends at the barline (tick 64) of its last measure:

`phrase_breaks = ["4:64", "8:64", "12:64"]`

**Example** — A hymn where each phrase ends after 2 quarter notes (tick 32) in a 4/4 measure, leaving a 2-beat pickup into the next phrase:
```
Phrase 1 ends after tick 32 of measure 4 → "4:32"
Phrase 2 ends after tick 32 of measure 7 → "7:32"
```

`phrase_breaks = ["4:32", "7:32"]`

In this case, the exporter inserts `\bar "" \break` at tick 32 of measures 4 and 7, allowing the pickup notes (ticks 33–64) to appear at the start of the next line.

**Example** — A slurred group of 5 eighth notes (5 × 8 = 40 ticks) followed by a rest in 4/4. Breaking at tick 32 would be mid-slur; tick 48 would be after the rest. Tick 40 breaks cleanly after the slur ends:

`"9:40"`

### Songs with a Chorus

Verse and chorus phrase breaks use the **same `phrase_breaks` field** with a unified list of `"M:T"` strings. There is no separate `chorus_phrase_breaks` field. All measure numbers are absolute (relative to the start of the full score).

**Example** — "Jesus Loves Me" with chorus starting at measure 17 (4/4 time, barline breaks at tick 64):
```
Measures 1–4:   Je -- sus loves me! This I know,       (verse phrase 1)
Measures 5–8:   For the Bi -- ble tells me so.          (verse phrase 2)
Measures 9–12:  Lit -- tle ones to Him be -- long;      (verse phrase 3)
Measures 13–16: They are weak, but He is strong.        (verse phrase 4)
Measures 17–20: Yes, Je -- sus loves me! (×3)           (chorus phrase 1)
Measures 21–24: Yes, Je -- sus loves me!                (chorus phrase 2)
Measures 25–28: Yes, Je -- sus loves me!                (chorus phrase 3)
Measures 29–32: The Bi -- ble tells me so.              (chorus phrase 4)
```

`phrase_breaks = ["4:64", "8:64", "12:64", "20:64", "24:64", "28:64"]`

The verse-to-chorus boundary is handled automatically by the exporter using the `@c` chorus marker — no extra entry is needed in `phrase_breaks` for that transition.

---

## Copyright Format

Copyright lines should follow this standard format:

```toml
copyrights = [
  "Words: [Author], [year] (public domain)",
  "Music: '[TuneName]' by [Composer], [year] (public domain)",
  "Obtained from the Open Hymnal Project, [year] (public domain)",   # only if from Open Hymnal
  "Arrangement by OpenPsalm, 2026 and released under the CC-BY 4.0 license",
]
```

Rules:
- No trailing periods on individual lines.
- Year and source on the same line as the author: `"Words by Henry F. Lyte, 1847 (public domain)"`.
- For public-domain works, include `(public domain)` inline.
- The OpenPsalm arrangement line is always last.
- Lines may embed links with `[text](url)` markup, e.g.
  `"Courtesy of the [Cyber Hymnal™](http://www.hymntime.com/tch)"`. The song
  detail page renders these as clickable links; all exports (MusicXML, LilyPond,
  PDF, presentations) and the JSON API strip the markup down to the link text.

---

## Complete Example

```toml
title = "Jesus Loves Me"
verse_count = 3
key_signature = "Eb"
time_sig_numerator = 4
time_sig_denominator = 4
tempo_bpm = 120
phrase_breaks = ["4:64", "8:64", "12:64", "20:64", "24:64", "28:64"]
copyrights = [
  "Words: Anna B. Warner, 1860 (public domain)",
  "Music: 'untitled' by William B. Bradbury, 1862 (public domain)",
  "Obtained from the Open Hymnal Project, 2006 (public domain)",
  "Arrangement by OpenPsalm, 2026 and released under the CC-BY 4.0 license",
]
commentary = "<p>Historical or musical notes about this song, displayed on the song detail page. Accepts HTML.</p>"

[parts.Soprano]
choral_type = "soprano"
clef = "treble"
staff_number = 1
notes = """
bes'4 g'4 g'4 f'4 | g'4 bes'4 bes'2 | c''4 c''4 ees''4 c''4 | c''4 bes'4 bes'2 |
bes'4 g'4 g'4 f'4 | g'4 bes'4 bes'2 | c''4 c''4 bes'4 ees'4 | g'4 f'4 ees'2 |
bes'2 g'4 bes'4 | c''4 ees''2. | bes'2 g'4 ees'4 | g'4 f'2. |
bes'2 g'4 bes'4 | c''4 ees''2 c''4 | bes'4 ees'4 g'4. f'8 | ees'1
"""

[lyrics.1]
text = "Je -- sus loves me! This I know, For the Bi -- ble tells me so. Lit -- tle ones to Him be -- long; They are weak, but He is strong."

[lyrics.2]
text = "Je -- sus loves me! He who died Hea -- ven's gate to op -- en wide; He will wash a -- way my sin, Let His lit -- tle child come in."

[lyrics.3]
text = "Je -- sus take this heart of mine; make it pure and whol -- ly Thine. Thou hast bled and died for me, I will hence -- forth live for Thee."

[lyrics.chorus]
text = "Yes, Je -- sus loves me! Yes, Je -- sus loves me! Yes, Je -- sus loves me! The Bi -- ble tells me so."
```

---

## Translations

A song may carry singing translations. The base `song.toml` is the song; each
translation lives beside it as `song_{lang}.toml`, where `{lang}` is a BCP-47
code:

```
162/song.toml       Face to Face with Christ My Savior   (English — the base)
162/song_es.toml    En presencia estar de Cristo          (Spanish)
```

All versions of a song share its directory number, so they share a page id:
`/songs/162` serves the base version and `/songs/162/es` the Spanish one. The
song page shows a language switcher whenever more than one exists.

The base file's language is `en` unless it says otherwise with a top-level
`language = "es"` — so a Spanish-origin hymn can carry an English translation
in `song_en.toml`.

### Translation files are overlays

A `song_{lang}.toml` states **only what differs**. Everything it leaves out —
the tune, key, tempo, time signature, phrase breaks, verse count, and every
`[parts.*]` — is inherited from `song.toml`. This is what keeps a later fix to
the base notation from silently leaving the translations wrong.

Most translations are lyrics-only and look like this in full:

```toml
title = "En presencia estar de Cristo"
copyrights = [
  "Letra: Carrie E. Breck, 1898 (dominio público)",
  "Traducción: Vicente Mendoza, 1905 (dominio público)",
  "Música: 'Face to Face' de Grant Colfax Tullar, 1898 (dominio público)",
  "Arreglo de OpenPsalm, 2026, publicado bajo la licencia CC-BY 4.0",
]

[lyrics.1]
text = "En pre -- sen -- cia‿es -- tar de Cris -- to, …"

[lyrics.chorus]
text = "¡Ca -- ra‿a ca -- ra‿es -- pe -- ro ver -- le, …"
```

**Write the `copyrights` block in the translation's own language.** The page is
in Spanish, so its credits are too — `Letra:`, `Música:`, `Traducción:`,
`(dominio público)`. Only proper names stay as they are: people, and the tune
name (`'Face to Face'`). This is the main reason a translation restates the
block instead of appending a line to it.

| Construct | Rule |
|---|---|
| Top-level scalar/array (`title`, `subtitle`, `key_signature`, `tempo_bpm`, `verse_count`, `default_verses`, `phrase_breaks`, `commentary`, `converge_verses`, `active`, …) | Present → **replaces** the inherited value. Absent → inherited. |
| `copyrights` | Replaces the whole list. A translation should always set this: the credits are written in the translation's language, and the translator line belongs beside the original lyricist rather than appended at the end. |
| `[parts.X]` | **Field-wise merge** with the base part `X`, so `notes = """…"""` alone overrides the notation and keeps `clef`, `choral_type`, `staff_number`. |
| `[parts.X]` not in the base | Added as a new part; must be complete. |
| Base part not mentioned | Inherited unchanged. |
| `[parts.X.lyrics]` | Defining **any** entry replaces that part's *entire* lyric map. |
| `[lyrics.*]` | Defining **any** entry replaces the *entire* global lyric map. |
| `[[time_sig_changes]]` | Replaces the whole array. |

Lyric maps replace wholesale rather than merging verse by verse on purpose: a
per-verse merge would let a translation supply verses 1–2 and silently inherit
the base language's verses 3–4, producing a half-translated page.

`active = false` is read per file, so a draft translation can sit beside a
published song.

### Overriding notes

Override a part's `notes` when the translation genuinely does not fit the
original rhythm — a line that needs one more syllable than the tune has notes.
Split the note in the part(s) that carry the syllable and leave the others
alone:

```toml
# The Spanish line has one syllable more than the English at measure 3.
[parts.Soprano]
notes = """
… | c''4. bes'8 a'8 g'8 f'8 g'8 | f'2 f'2 | …
"""
```

The merged result is validated exactly like a base file: every measure must sum
to its time signature, and every verse must have one syllable per lyric slot in
every part. A mismatched syllable count is the most common authoring error in a
translation and the seeder rejects it by name.

### Elisions (synalepha)

A lyric slot cannot contain a space — syllables are split on whitespace. When
two words are sung on one note, as Spanish synalepha constantly requires, join
them with an **undertie** `‿` (U+203F), the standard engraving mark for an
elision:

```
"En pre -- sen -- cia‿es -- tar de Cris -- to,"
```

`presencia estar` is five syllables of text but four notes: `cia‿es` is one
slot. The score prints the undertie; the lyrics page and the search index
expand it back to a space, so the poem reads "En presencia estar de Cristo" and
a search for "cara a cara" matches.

### Registering a language

A `song_{lang}.toml` whose code is not in the language registry is a hard seed
error, not a skip — so a translation can never sit silently unimported. Add new
codes to `LANGUAGES` in `src/i18n.rs` (OpenPsalm side), appending only: the
`ordinal` values are part of every translation row's database identity and must
never be reused or renumbered.

---

## Seeding and Database

Songs are seeded on startup from `songs/{N}/song.toml`, plus any
`songs/{N}/song_{lang}.toml` translations, if that (song directory, language)
has not already been imported. To re-seed after changes to a TOML (including `phrase_breaks` or `optional_phrase_breaks`), clear the songs table and restart:

**SQLite (local dev):**
```bash
rm openpsalm.db && cargo run
```

**PostgreSQL (production):**
```sql
TRUNCATE songs CASCADE;
```
Then restart the service.
