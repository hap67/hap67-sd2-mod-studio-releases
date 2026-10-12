# SD2 Mod Studio file formats

Notes on the files that SD2 Mod Studio reads and writes, so other tools can read and write them too:

| Extension | What it holds |
|---|---|
| `.sd2units` | a unit pack: new units with their data, models and names |
| `.sd2unit` | one unit, built for a pack |
| `.sd2pkg` | a share package: a mod's source, to pass to a friend who also uses SD2 Mod Studio |
| `.sd2mod` | a generated mod, ready to play |
| `.sd2div` | a division with its battlegroups |
| `.sd2bg` | one battlegroup |

The full description of each, field by field, is in **[studio-formats.md](studio-formats.md)**.

## Status

- Written 12 October 2026, for SD2 Mod Studio 0.6.
- The formats may still change while the app is in alpha. Each file carries a format number; a change that older
  versions cannot read raises it.
- The examples are made up (invented names and values).

## Licence

This documentation is licensed under the
[Creative Commons Attribution 4.0 International licence (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
You may share and adapt it, including commercially, as long as you give credit (for example "SD2 Mod Studio
format notes") and say if you changed it. The licence covers this documentation only, not the SD2 Mod Studio
application or its source code.

*Steel Division 2* is a game by Eugen Systems. SD2 Mod Studio is not made or endorsed by Eugen Systems.
