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

`CODECS` holds 44 entries. `type:'mezz'` carries a bits-per-pixel figure
(`bpp`); `type:'raw'` carries `bitDepth` and a compression `ratio`, optionally
with `presets` (the ratio chips) and `ratioLocked` for uncompressed formats.

Four flags change behaviour, and are easy to miss:

- `approx:true` appends "(approx.)" to the codec name.
- `ditOnly:true` hides the codec from Stream mode. Six entries use this.
- `resRange:[minPx,maxPx]` limits which of a camera's resolutions the codec
  offers in DIT mode, by total pixel count.
- `hint` shows a note beneath the readout.

`CAMERAS` holds 22 models across ARRI, Blackmagic, Canon, DJI, GoPro,
Panasonic, RED, and Sony. Each lists its native resolutions and a `codecs`
array of **`CODECS` ids**. A typo in that array silently drops the option from
the dropdown rather than failing loudly, so check a camera's codec list in the
browser after editing it.

## The accuracy contract

The bit-rate figures are the product. Someone books storage against them.

Change a `bpp`, `bitDepth`, or `ratio` only against a manufacturer
specification, and say in the commit message which spec and which operating
point it came from. Never adjust a figure to make a comparison look tidier, and
never round one to make two codecs agree.

Where a published spec is stated at one resolution, record that in `hint` (the
XAVC-Intra entries do this) rather than silently averaging across resolutions.

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

## Before calling a change done

- Open `index.html` and exercise **both** modes.
- Check the readout line, the four stat tiles, the storage total, and the
  comparison table, which is sorted by data rate and highlights the selection.
- Check one RAW codec (bit depth, ratio chips, the locked-ratio case) and one
  mezzanine codec.
- Check the layout at phone width. Breakpoints are 880px, 640px, and 560px; at
  560px the comparison table stacks and takes its column names from each cell's
  `data-label`. A new numeric column needs a `data-label` or it loses its
  heading on a phone.
- Confirm no new external request has been introduced.
