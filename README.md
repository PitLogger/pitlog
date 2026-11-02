# pitlog

The open file format of [PitLogger](https://github.com/PitLogger) recordings, and a free viewer
that opens `.pitlog` files in the browser and exports them as CSV and GPX.

| | |
|---|---|
| [SPEC.md](SPEC.md) | complete format specification (version 1.0, **draft**) |
| [examples/example.pitlog](examples/example.pitlog) | small example file for testing readers |
| [viewer/index.html](viewer/index.html) | the viewer: one HTML file, works offline |

## Viewer

Open `viewer/index.html` in a browser (download it, or use the published page) and drop a
`.pitlog` file on it. It shows the values over time, one chart per value on a shared time axis,
the GPS track coloured by speed (optionally on an OpenStreetMap map), laps, markers, events and trouble codes. Drag across a chart to zoom in;
pointing at a chart or the track shows the values at that moment everywhere. Laps: circuits (one
start/finish line) and sprints (start to finish, e.g. Bridge to Gate); the lines come from the
PitLogger's recording, are found automatically on a circuit, or are set on the map – "Export track"
saves them for the [tracks](https://github.com/PitLogger/tracks) list. Exports: a CSV table
(one row per 0.1 s), a CSV with every sample, and the GPS track as GPX.

The file is read in the browser only – nothing is uploaded, and without the map the page loads
nothing from the internet. The map under the track (OpenStreetMap) is optional and off by default;
switching it on shows what the tile server receives (IP address, map area). A damaged or cut-off
file (power loss) is read up to the damage; later intact blocks are recovered.

**Why a binary format?** A PitLogger records in a car, where the power can disappear at any
moment. `.pitlog` writes data in blocks with checksums, so a power loss costs at most the last
few seconds. Every file describes its own channels and also keeps the raw OBD-II bytes, so old
recordings stay readable and can be decoded better later.

**Your data stays yours.** The format is fully documented here and the converter will be free
and open source, as a web app and as a standalone tool. Even if this project stops one day, you
can still read your files – or write your own reader from the specification.

## Status

- [x] Format specification (draft)
- [x] Viewer: web app (single HTML file, works offline)
- [ ] Converter: command line
- [x] Exports: CSV (long and wide), GPX
- [ ] Export: JSON

## License

MIT, see [LICENSE](LICENSE).
