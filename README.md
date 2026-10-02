<p align="center">
    <h1 align="center">
        OpenTPW File Formats
    </h1>
    <p align="center">
        How the files of <a href="https://en.wikipedia.org/wiki/Theme_Park_World">Sim Theme Park / Theme Park World</a> are laid out, byte by byte.
    </p>
    <p align="center">
        <a href="https://github.com/OpenTPW/OpenTPW">OpenTPW</a> · <b>File Formats</b>
    </p>
</p>

## What this is

These pages describe the file formats of Theme Park World (1999): every field, its offset, its size and what it
means. They are written for [OpenTPW](https://github.com/OpenTPW/OpenTPW), and anyone reading the game's files can
use them.

**This repository ships no game content.** It describes the files; it does not contain them.

## What is documented

One page per format. The website is offline, so read the pages here on GitHub.

| Page | Files |
|------|-------|
| [Archives](src/content/docs/formats/wad.md) | `.wad` |
| [Bitmap fonts](src/content/docs/formats/fonts.md) | `.bf4` |
| [Compiled ride scripts](src/content/docs/formats/rsse.md) | `.rse` |
| [Item height maps](src/content/docs/formats/hmp.md) | `.hmp` |
| [Lip sync](src/content/docs/formats/lip-sync.md) | `.lip` |
| [Lobby scripts](src/content/docs/formats/lobby-scripts.md) | `.txt` |
| [Models](src/content/docs/formats/models.md) | `.md2` |
| [Options and players](src/content/docs/formats/options-and-players.md) | `.tcf`, `gms.dat` |
| [Park signs](src/content/docs/formats/sgn.md) | `.sgn` |
| [Particle library](src/content/docs/formats/particles.md) | `.plb` |
| [Ride script source](src/content/docs/formats/rss.md) | `.rss` |
| [Saves](src/content/docs/formats/saves.md) | `.tpws`, `.ints`, `.lays` |
| [Settings and modifiers](src/content/docs/formats/sam.md) | `.sam` |
| [Sound categories](src/content/docs/formats/sound-categories.md) | `cat_*.map` |
| [Sound data](src/content/docs/formats/sounds.md) | `.sdt` |
| [Sprites](src/content/docs/formats/sprites.md) | `.esp`, `.tpc` |
| [Strings](src/content/docs/formats/strings.md) | `.dat`, `.str` |
| [Texture correspondence tables](src/content/docs/formats/tct.md) | `.tct` |
| [Textures](src/content/docs/formats/texture.md) | `.wct` |
| [Video](src/content/docs/formats/video.md) | `.tgq` |

Ride scripts run on a small virtual machine. Two pages cover it: [how it works](src/content/docs/vm/info.md) and
[its instruction set](src/content/docs/vm/instructions.md).

## Quick start

The pages are Markdown in [`src/content/docs/`](src/content/docs). To build the website, install
[Node.js](https://nodejs.org/) 20 or newer, then:

```sh
npm install
npm run dev      # a live preview at http://localhost:4321
npm run build    # the finished site, in dist/
```

## Writing a page

**Check every file, not one.** Write a layout down only once it fits every file of that kind the game ships. Say
near the top of the page how many files were checked.

**Say what is not known.** Mark an unworked field as unknown. Never guess.

**Bytes only.** A page says what is in the file. What the game *does* with it goes in
[OpenTPW](https://github.com/OpenTPW/OpenTPW), under `docs/exe/`.

**Match the other pages.** Start with `title: <Name> (*.ext)`; the sidebar finds the page by itself. Give the layout
as a table of Offset, Size and Field. Link another page as `](/formats/texture/)`. Add the page to the table above.
Run `npm run build` before you commit.

## Contributing

Contributions are very welcome:

1. Fork the [original project](https://github.com/OpenTPW/OpenTPW.FileFormats).
2. Create a branch named `YourName/FeatureName`.
3. Open a pull request when you are ready.

## License

This repository has no license file yet.
