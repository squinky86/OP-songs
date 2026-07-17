# OpenPsalm Songs

A free, structured corpus of hymns and worship songs — every note, syllable,
dynamic, and articulation stored in plain-text TOML that you can read, diff, and
edit in any editor.

This is the song data behind [openpsalm.com](https://openpsalm.com). It lives in
its own public repository so the corpus is useful on its own: engrave it, play
it, convert it, or correct it, without needing the site.

## What's here

```
1/song.toml                 the song: metadata, parts, notes, lyrics
2/song.toml
…
93/song.toml
93/copyright.txt            (songs used by permission) a record of that permission
docs/song-toml-format.md    the format reference — every field, every marker
docs/song-style-guide.md    which construct to use when, and why
```

Each song is one directory named by its integer ID, holding a `song.toml`:

```toml
title = "Abide With Me"
verse_count = 4
key_signature = "Eb"
time_sig_numerator = 4
time_sig_denominator = 4
tempo_bpm = 90
copyrights = [
  "Lyrics: Henry F. Lyte, 1847 (public domain)",
  "Music: William H. Monk, 1861 (public domain)",
  "Arrangement by OpenPsalm, 2026 and released under the CC-BY 4.0 license",
]

[parts.Soprano]
choral_type = "soprano"
clef = "treble"
notes = "ees4 ees4. d8 | ees4 f g4. f8 | ees2 ..."
```

Notation is LilyPond-flavored: pitch plus accidental (`is` sharp, `es` flat),
octave marks, and duration — `ees4`, `c'8.`, `g,2` — with bars separated by
` | `. Slurs, ties, fermatas, dynamics, hairpins, and shape-note data all have
markers. See [docs/song-toml-format.md](docs/song-toml-format.md) for the full
reference and [docs/song-style-guide.md](docs/song-style-guide.md) for the
conventions every song here follows.

## Licensing — read before you use a song

**This repository is not under one license.** Each song's `copyrights` field is
authoritative.

Most songs set a public-domain text and tune, with an OpenPsalm arrangement
offered under CC-BY 4.0 — use those freely with attribution. **A minority are
still under copyright and appear here by permission of the rights holders**;
their `copyrights` field says "Used by permission" and gives a term end date, and
their directory carries a `copyright.txt` recording the grant. That permission was
granted to OpenPsalm and does not extend to you. Read [LICENSE](LICENSE) in full
before redistributing anything from this repo.

Whatever you do with a song, keep its `copyrights` field with it.

## Found a problem?

Wrong note, missing syllable, bad line break, incorrect attribution? Please
[open an issue](../../issues/new?template=song-problem.yml). Song pages on
openpsalm.com link here directly with the song details prefilled.

Pull requests are welcome, for corrections and for new songs. If you're adding a
song, follow the [style guide](docs/song-style-guide.md) — it exists so the
corpus stays consistent — and make sure the `copyrights` field is complete and
accurate. Don't add a song under copyright unless you can document permission.

## Using the data

The format is plain TOML, so any language reads it with a stock parser. Songs are
imported into [OpenPsalm](https://openpsalm.com), which renders them to MusicXML,
MIDI, MP3, LilyPond, engraved PDF, and presentation slides — but nothing here is
specific to that implementation.
