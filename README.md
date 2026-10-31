# pitlog

The open file format of [PitLogger](https://github.com/PitLogger) recordings – and, soon, the
free converter that turns `.pitlog` files into CSV, GPX and more.

| | |
|---|---|
| [SPEC.md](SPEC.md) | complete format specification (version 1.0, **draft**) |
| [examples/example.pitlog](examples/example.pitlog) | small example file for testing readers |

**Why a binary format?** A PitLogger records in a car, where the power can disappear at any
moment. `.pitlog` writes data in blocks with checksums, so a power loss costs at most the last
few seconds. Every file describes its own channels and also keeps the raw OBD-II bytes, so old
recordings stay readable and can be decoded better later.

**Your data stays yours.** The format is fully documented here and the converter will be free
and open source, as a web app and as a standalone tool. Even if this project stops one day, you
can still read your files – or write your own reader from the specification.

## Status

- [x] Format specification (draft)
- [ ] Converter: web app (single HTML file, works offline)
- [ ] Converter: command line
- [ ] Exports: CSV (long and wide), GPX, JSON

## License

MIT, see [LICENSE](LICENSE).
