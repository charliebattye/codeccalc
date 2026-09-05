# CodecCalc

CodecCalc is a capture data-rate and storage calculator for film and television
production. Camera operators, DITs, and producers use it to size cards and
drives before a shoot.

It is published from this repository to GitHub Pages at
<https://charliebattye.github.io/codeccalc/>, and appears on
charliebattye.com at `/tools/codeccalc` as a pinned copy (see *Publishing*).

Charlie owns this tool and its figures. Act as a calm guide as well as a
developer: explain the immediate outcome in plain English, and ask one short
question at a time when a decision is genuinely required.

## The shape of the project

The whole tool is one file: `index.html`. It carries its own CSS and
JavaScript, and has no build step, no dependencies, no package manager, and no
server. Its only external requests are the three Google Fonts families in
`<head>`.

To preview a change, open `index.html` in a browser and reload. There is
nothing to install and nothing to run.

Keep it that way unless there is a real reason not to. A build step, a
framework, or a dependency would each cost more than they currently return.

## How it calculates

Two modes share a single `render()` pass. **Stream Record** offers the full
codec list; **DIT / Data Wrangle** offers a camera's own codecs and
resolutions.

    mezzanine   Mb/s = w x h x fps x bpp / 1e6
    raw         Mb/s = w x h x fps x bitDepth / ratio / 1e6
    audio       MB/s = channels x sampleRate x (bits / 8) / 1e6

GB and TB are decimal (divide by 1000, not 1024). That is deliberate: cards and
drives are sold in decimal capacity, so the answer matches what the shop sells.
Do not "correct" it to 1024.

## The data model

`CODECS` holds 47 entries in three rate models:

- **`type:'mezz'`** (33) carries a bits-per-pixel figure, `bpp`.
- **`type:'raw'`** (9) carries `bitDepth` and a compression `ratio`, optionally
  with `presets` (the ratio chips) and `ratioLocked` for uncompressed formats.
- **`type:'table'`** (5) carries `rates`, a published bit rate per mode. Used
  where no bits-per-pixel value can be correct, because the manufacturer sets
  the rate by mode rather than by pixel count. See below.

Four flags change behaviour, and are easy to miss:

- `approx:true` appends "(approx.)" to the codec name.
- `ditOnly:true` hides the codec from Stream mode and from its comparison
  table. Sixteen entries use this: every camera-specific format lives in DIT
  mode, leaving Stream mode for codecs you can meaningfully compare across
  manufacturers.
- `resRange:[minPx,maxPx]` limits which of a camera's resolutions the codec
  offers in DIT mode, by total pixel count. It is consulted **only** in DIT
  mode, so it restricts nothing in Stream mode.
- `hint` shows a note beneath the readout.

`CAMERAS` holds 22 models across ARRI, Blackmagic, Canon, DJI, GoPro,
Panasonic, RED, and Sony. Each lists its native resolutions and a `codecs`
array of **`CODECS` ids**. A typo in that array silently drops the option from
the dropdown rather than failing loudly, so check a camera's codec list in the
browser after editing it.

### Table-driven codecs

`rates` is keyed `'WIDTHxHEIGHT'`, then by frame rate:

    rates:{ '3840x2160':{24:36.1, 30:53.0, 60:65.4, 120:89.4}, ... }

`tableMbps()` reads it. Frame rates within 1% of a published one snap to it, so
23.976 / 29.97 / 59.94 resolve to 24 / 30 / 60. A frame rate between two
published ones is interpolated. Anything outside the published range returns
`NaN`, which `fmt()` renders as an em dash.

**It deliberately never extrapolates.** Absence from the table means the
manufacturer publishes nothing, and often means the camera cannot shoot that
mode at all — GoPro mark MISSION's Max setting N/A below 480 fps at 1080p. An
earlier version scaled outside the published range and produced a confident
120 Mb/s for exactly that unavailable mode. A blank is the honest answer.

**The keys must track the camera's resolution list.** `rates` keys are matched
against `w + 'x' + h` from `CAMERAS`. Change a camera's resolution without
changing the table (or the reverse) and every mode silently reads as a dash.
Nothing warns you, so check both after editing either.

## The accuracy contract

The bit-rate figures are the product. Someone books storage against them.

Change a `bpp`, `bitDepth`, or `ratio` only against a manufacturer
specification, and say in the commit message which spec and which operating
point it came from. Never adjust a figure to make a comparison look tidier, and
never round one to make two codecs agree.

Where a published spec is stated at one resolution, record that in `hint` (the
XAVC-Intra and X-OCN entries do this) rather than silently averaging across
resolutions.

Prefer a stated specification to one you derive. Card-capacity tables are
bitrate tables in disguise — capacity divided by recording time — and they are
often the only published source for consumer cameras. But check the derived
figure against any stated one: GoPro's MISSION table rounds every Max entry to
"2Hrs", which implies 284 Mb/s where GoPro state a constant 240.

**The page documents its own figures.** "Notes on the numbers", at the foot of
`index.html`, records where each family's numbers come from and how far they
can be trusted. It is part of the product, not a comment. Change a figure or a
rate model and change its paragraph in the same commit, or the tool will
describe itself inaccurately — worse in a provenance section than saying
nothing at all.

## Publishing

`main` is served by GitHub Pages, so pushing to `main` updates the tool's own
URL within a minute or two.

**That does not update charliebattye.com.** The website ships its own pinned,
checksummed copy of this file so its deploys never depend on Pages being up.
Refreshing it is a deliberate, separate act, done from the website repository:

    npm run sync:codeccalc         # report what would change, write nothing
    npm run sync:codeccalc:apply   # accept it, updating file + checksum + tests

So a change here reaches the website only when someone runs that there. If
Charlie asks why the site still shows the old version, this is why.

## Checking your work

`index.html` carries its own data checks. Open it with `?selftest` on the end of
the URL, or call `runSelfTest()` from the browser console — useful where a
preview pane rewrites the URL and drops the query string. `runSelfTest(true)`
returns the results without drawing the panel.

Ten checks run against the live `CODECS` and `CAMERAS`: id uniqueness, that each
codec carries the fields its rate model needs, that every codec id a camera
names exists, that every resolution a camera lists is reachable, that every
camera/codec pairing offers something, that every combination the DIT mode
offers produces a usable figure, that published table figures round-trip
exactly, that broadcast frame rates snap to their whole-number neighbour, and
that every `rates` key matches a raster on a camera that uses that codec.

They live inside `index.html` rather than a separate page because a browser will
not let one `file://` document read another's data. A standalone checker would
need a web server, breaking "there is nothing to run", or its own copy of the
data, which would drift. Here they can only ever test what the tool uses — the
`rates` key check caught two rasters added speculatively that no camera had.

It also prints notices, which are not failures: a raster a camera shoots that
sits *inside* the range a table codec covers but has no published figure.

## Before calling a change done

- Run `runSelfTest()`. All ten checks must pass before anything else is worth
  looking at.
- Open `index.html` and exercise **both** modes.
- Check the readout line, the four stat tiles, the storage total, and the
  comparison table, which is sorted by data rate and highlights the selection.
- Check one codec of each rate model: a mezzanine one, a RAW one (bit depth,
  ratio chips, the locked-ratio case), and a table-driven one.
- The self-test covers the data traps that used to be manual here: published
  figures round-tripping, and every listed resolution staying reachable. The
  BURANO's HD was once stranded — present in the dropdown, selectable by no
  codec — and stayed hidden for months. That check now runs in a second.
- Check the layout at phone width. Breakpoints are 880px, 640px, and 560px; at
  560px the comparison table stacks and takes its column names from each cell's
  `data-label`. A new numeric column needs a `data-label` or it loses its
  heading on a phone.
- Confirm no new external request has been introduced.
