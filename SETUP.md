# Running EMPIRE on TOPS-10

You need a working TOPS-10 system. If you don't have one, *TOPS-10 in a box* —
a preconfigured SIMH KS10 monitor you can run on a modern machine — is a
convenient working example:
<https://www.filfre.net/2011/05/tops-10-in-a-box/>.

## 1. Transfer the files to TOPS-10

Get the repository files onto a TOPS-10 disk by whatever means your setup
offers — a mountable tape or disk image, or a host file-transfer tool. The
sources you need are the FORTRAN files (`1.FOR`..`18.FOR`, `EMPIRE.FOR`), the
assembly files (`CURSOR.MAC`, `HELP.MAC`, `PACK.MAC`, `SUBS.MAC`, `MUNCH.MAC`,
`PATH.MAC`, `FORSUP.MAC`), the help text `EMPIRE.HLP`, and the build files
`EMPIRE.CTL` and `EMP.CMD`.

The terrain maps `X.A`..`X.E` (see §3a) are binary PDP-10 word files, not text;
transfer those by a binary-safe method that preserves the bits — a tape or disk
image, or a binary-safe transfer tool — never a text paste.

## 2. Build the deck

Submit the build control file:

```
.SUBMIT EMPIRE.CTL
```

It compiles the `.MAC` and `.FOR` sources, links them via `@EMP.CMD`, and
`.SAVE`s `EMPIRE.EXE` (see `EMPIRE.CTL` for the exact steps; you can also run
them by hand).

## 3. What EMPIRE opens at run time

With `EMPIRE.EXE` built, a few data files and logical devices have to be in
place before it will run:

| Need | Where the source asks for it | What you provide |
|------|------------------------------|------------------|
| Terrain maps | `15.FOR`: `OPEN(UNIT=1,DEVICE='GAM',FILE=IFILE,...)` | `X.A`..`X.E` reachable on **`GAM:`** |
| In-game `H` help | `HELP.MAC`: `LOOKUP 11,HLPFIL` of `EMPHLP.HLP` in `[^D29970,,'WBG'-202020]`, opened on device `'ALL '` | `EMPHLP.HLP` in **`[72422,472227]`**, reachable via **`ALL:`** |
| Printed directions | `15.FOR`: `'DIRECTIONS ARE ON HLP:EMPIRE.HLP'` | `EMPIRE.HLP` on **`HLP:`** (only for the monitor's `.HELP EMPIRE`) |
| Saved game | `15.FOR`: `OPEN(UNIT=1,DEVICE='DSK',FILE='EMPIRE.DAT',...)` | nothing — created in your area on first save |

### 3a. The terrain maps on `GAM:`

A new game reads its base map from one of five files on device `GAM:`:
`GAM:X.A`, `GAM:X.B`, `GAM:X.C`, `GAM:X.D`, `GAM:X.E`. `15.FOR` selects one by
its internal `KILL` value (0–4), encoding the name in `FILE=IFILE` as packed
7-bit ASCII.

The original map files did not travel with the source and appear to be lost.
We supply five files in the same format under those names.

Put the maps where `GAM:` resolves. The simplest way, as a normal user, is to
point `GAM:` at your own disk area and keep the maps there:

```
.ASSIGN DSK GAM
```

On the original system `GAM:` was the shared games area; assign it to whatever
structure or `[p,pn]` actually holds the maps if you keep them elsewhere.

### 3b. The in-game `H` help — Walter's account `[72422,472227]`

The `H` command at the orders prompt runs `HELP.MAC`, which opens device `ALL:`
and does a `LOOKUP` for `EMPHLP.HLP` in PPN `[^D29970,,'WBG'-202020]`. That is
Walter Bright's account, `[29970,WBG]` — the same one the banner gives for mail
(`'FOR QUESTIONS OR BUGS SEND MAIL TO [29970,WBG]'`). In octal the PPN is:

```
[72422,472227]
   |      |
   |      'WBG' in SIXBIT (0o674247) minus 0o202020
   29970 decimal = 0o72422
```

Set it up:

1. **Create the UFD.** On TOPS-10 7.04, `CREDIR` makes the directory (older
   monitors use `BUILD`):

   ```
   .R CREDIR
   Create directory: [72422,472227]
   ```

   Only the directory is needed — you don't have to create a login account for
   `[29970,WBG]`. `HELP.MAC` just `LOOKUP`s the file there; nobody logs in as
   that PPN.

2. **Copy the help text in under the name `EMPHLP.HLP`.** It is the same text as
   `EMPIRE.HLP`. Use interactive PIP (the one-line CCL form `PIP dst=src` can
   misfire with `?PIP?`):

   ```
   .R PIP
   *[72422,472227]EMPHLP.HLP=EMPIRE.HLP
   *^Z
   ```

3. **Make `ALL:` resolve** so the `LOOKUP` finds it:

   ```
   .ASSIGN DSK ALL
   ```

With that, `H` in the game prints the full help text.

### 3c. The directions line — `HLP:EMPIRE.HLP`

The intro prints `DIRECTIONS ARE ON HLP:EMPIRE.HLP`. The game itself does not
open this file; the line just points the player at the system HELP facility
(`.HELP EMPIRE` at monitor level). To make that work:

- If you can write the system help library (often `[2,5]`), put `EMPIRE.HLP`
  there.
- As a normal user, point `HLP:` at your own area instead:

  ```
  .ASSIGN DSK HLP
  ```

  with `EMPIRE.HLP` in that area; then `.HELP EMPIRE` prints it.

This step is optional — it only affects the monitor-level `.HELP EMPIRE`, not
play.

### 3d. The saved game — `DSK:EMPIRE.DAT`

Nothing to pre-create. EMPIRE writes `EMPIRE.DAT` to your own disk area when you
save, and reads it back on the next run (`15.FOR` checks for it with `ILDET`
before offering a new game). You only need normal write access to `DSK:`.

## 4. Run it

A normal-user session, with `X.A`..`X.E`, `EMPHLP.HLP` (in `[72422,472227]`),
and optionally `EMPIRE.HLP` in place:

```
.ASSIGN DSK ALL                 ;HELP.MAC opens ALL:
.ASSIGN DSK GAM                 ;new-game maps GAM:X.A..X.E
.ASSIGN DSK HLP                 ;optional, for .HELP EMPIRE
.RUN EMPIRE
```

`ASSIGN DSK ALL` and `ASSIGN DSK GAM` are the two that matter for play; without
them FOROTS fails the corresponding `OPEN` (e.g. `?FRSOPN No such directory`).
