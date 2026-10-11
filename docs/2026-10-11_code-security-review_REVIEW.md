# FVS Interface: Code, Bug, and Security Review

Review date: 2026-10-11. Branch `claude/repo-scan-security-ns2pl7`, base commit `c4e7336`.

Every R, JavaScript, HTML, makefile and parameter-build file in the repository was read in full: about 32,000 lines across `fvsOL`, `rFVS`, `FVSPrjBldr` and `FVSDataConvert`. R is not available in the review container, so nothing was executed. Each finding was traced through the source by hand, and the most severe items in each section were then rechecked line by line. Findings marked *Plausible* depend on behaviour outside this repository, such as the FVS Fortran API, R's internal unzip, or the deployment configuration. No source files were changed. Exploit inputs and the production hostname are deliberately omitted from this copy; a full version is held privately.

Severity assumes the hosted "online" deployment: many users share one Shiny Server, and each project is reached by a UUID link (`FVSPrjBldr`). In "onlocal" mode the user already controls the machine, so most security items drop to Low. The correctness bugs apply in both modes.

## 1. Structure

**rFVS** is a thin wrapper around the FVS shared library. `fvsLoad` calls `dyn.load` on `<bin>/<variant>.so` and checks that 17 entry points are present. The other functions call `.Fortran("fvs*")` or `.C("Cfvs*")`. They run FVS to stop points (`fvsRun`, `fvsInteractRun`), get and set tree, species, event-monitor and summary attributes, and add trees or activities. Three functions are plain R: `fvsMakeKeyFile`, `fvsSetupSummary` and `fvsCompositeSum`.

**fvsOL** is the Shiny application.

- **`R/server.R`** (9,160 lines) holds nearly all the reactive logic:
  - project start-up and locking;
  - output loading and the custom SQL query feature;
  - the ggplot explorer;
  - stand and group selection;
  - component (keyword) editing;
  - run execution;
  - downloads, SVS 3D views and leaflet maps;
  - project backup, restore and delete;
  - input-database upload and conversion, including Access via `mdb-tools`;
  - the table editor;
  - import from other projects.
- **`R/fvsRunUtilities.R`** keeps the run model in reference classes. Runs and objects are stored as serialized blobs in `FVSProject.db`, and each run's output is merged into `FVSOut.db`.
- **`R/writeKeyFile.R`** builds the FVS `.key` file.
- **`R/componentWins.R`** and **`R/mkInputElements.R`** build the management windows and the generic parameter forms. Those forms are driven by `data/prms.RData`, which `parms/mkpkeys.R` compiles from `parms/*.prm|kwd`.
- **`R/externalCallable.R`** is the exported `extn*` API. Its `extnSimulateRun` launches background runs: it writes a master `.Rscript` that fans stands out to a PSOCK cluster and starts it with `system("nohup Rscript ...")`. Interactive runs use a one-worker PSOCK cluster driven by `clusterEvalQ`.
- **`inst/extdata`** holds three R growth-and-yield plug-ins (Acadian v12.5.0, Adirondack v5.1.0, HI v0.2.0) and the scripts that drive them (`customRun_fvsRun*.R`). Each driver stops FVS at point 5 every cycle, pulls the tree list, projects it annually (Acadian, HI) or once per cycle (Adirondack), and pushes back `dg`, `htg`, `cratio` and `mort` through `fvsSetTreeAttrs`. Ingrowth is optionally added at stop point 6, and volume can come from Kozak or Westfall–Scott taper. `rFVS::fvsSetTreeAttrs` skips any attribute whose vector has the wrong length or contains NA, and only issues a warning when it does. Several model bugs below therefore surface silently as FVS-NE base-model growth. `sqlQueries*.R` build the composite, stand/stock and calibration tables.

**FVSPrjBldr** is a small Shiny app. It creates `/home/shiny/FVSwork/<uuid>/` with an `app.R` and emails the link through `ssmtp`. The UUID is the only access control on a project.

**FVSDataConvert** is a standalone Shiny app that converts uploaded Access or xlsx files to `FVS_Data.db`, using `mdb-schema`, `mdb-export` and the `sqlite3` command-line shell.

## 2. Security findings

### Critical: remote code execution or cross-tenant destruction on a hosted server

| # | Location | Issue | Fix |
|:-:|:--|:--|:--|
| S1 | `fvsOL/R/server.R:2141-2144` | Plot labels beginning with `expression` go straight to `eval(parse())`. A crafted label runs arbitrary R, and through it shell commands. The label is saved in GraphSettings, so it runs again whenever that setting is loaded. | Accept only a single parsed `expression()` call and check its contents against an allow-list, or drop eval altogether. |
| S2 | `fvsOL/R/externalCallable.R:1061`, `1051-1053` | The run title (`input$title`, only trimmed) is pasted into the generated `<uuid>.Rscript`, which a background run then executes. A crafted title breaks out of the string literal and executes. `input$runScript` is pasted the same way. | Write values with `deparse()`, and check `runScript` against `runScripts`. |
| S3 | `fvsOL/R/server.R:6698-6701`, `6762-6764`; `FVSDataConvert/server.R:86-87`, `107-109`, `130-133` | Shell injection through `mdb-schema`/`mdb-export`. The file name inside an uploaded zip and the table names in an uploaded Access file are pasted into `system()`. Shell metacharacters in those names are expanded even inside double quotes. | Use `system2()` with `shQuote()`, rename extracted files to a fixed name, and only accept table names matching `^[A-Za-z0-9_ ]+$`. |
| S4 | `fvsOL/R/server.R:6409-6418` | `unlink(paste0("../", input$PrjDelSelect), recursive=TRUE)`. Shiny does not check that a select-input value is one of the offered choices, so a forged value deletes any directory the server can write, including other users' projects. | Require `%in% getProjectList()` and `basename(x)==x`. |
| S5 | `fvsOL/R/server.R:4791-4799` → `fvsRunUtilities.R:143-153` | The background-run kill button passes `input$bkgRuns` into `removeFVSRunFiles`, which deletes recursively and also calls `dir(pattern=uuid)` as a regex. Crafted values wipe the current project or a sibling project. | Require a UUID regex match and membership in `getBkgRunList()`. |
| S6 | `fvsOL/R/server.R:4993-5009`, `8646-8657` | `load()` of an uploaded `.RData` into the observer's own environment. A crafted file can replace functions the observer calls next. On R < 4.4, CVE-2024-27322 also applies. | `load(f, envir=new.env(parent=emptyenv()))`, then validate the types. |
| S7 | `fvsOL/R/server.R:8462` → `externalCallable.R:225` (*Plausible*) | `unserialize()` of run and object blobs taken from imported project databases. | Do not unserialize untrusted projects. Store runs as JSON, or require R ≥ 4.4 and validate the result. |
| S8 | `fvsOL/inst/extdata/customRun_*.R:9-21`; `server.R:155`, `4654`, `8680` (*Plausible*) | Model code and `runScripts.R` are sourced from the project directory before the packaged copy. Any route that writes files there (for example the backup restore in S10) becomes R code execution. | When `!isLocal()`, source only from `system.file()`. |

### High

| # | Location | Issue | Fix |
|:-:|:--|:--|:--|
| S9 | `server.R:4891-4962`, `6099-6106`, `8479-8498`, `8624-8627`, `5622` | Forged `pickBackup`, `runSel`, `impPrjSource` and `SVSImgList1` values read arbitrary files (system files, other projects' databases), delete them, or import another user's runs and data. | Check each value against the server-side list it was drawn from. |
| S10 | `server.R:6216-6366`, `8446-8451`, `8205-8214` | Restore unzips any server zip that `pickBackup` names, and it copies `FVS*.so` from the archive into `fvsBin`, which is later loaded with `dyn.load`. Several `unzip()` calls keep the archive's paths (*Plausible* zip slip). | Whitelist the archive, reject entries containing `..` or absolute paths, and never write to `fvsBin` from an upload. |
| S11 | `server.R:1121-1133` | Custom Query runs any SQL, including `ATTACH` of other projects' databases or of arbitrary writable paths. | Open a separate read-only connection and reject `ATTACH`, `PRAGMA` and `VACUUM INTO`. |
| S12 | `server.R:1993-2009`; `fvsOutUtilities.R:72-73` | Column names from a custom query (SQL aliases) are pasted into `eval(parse(text="order(...)"))`. | `do.call(order, unname(dat[cols]))`. |
| S13 | `FVSPrjBldr/uuidgen.R` | Project UUIDs come from Mersenne Twister seeded once with 32 bits, and the state persists across calls. The generator state is small enough that later project links are predictable. | `uuid::UUIDgenerate()`, or `openssl::rand_bytes(16)`. |
| S14 | `FVSPrjBldr/server.R:20-22`, `43-55`; `prjListEmail.R:13`, `43`, `74-76` | Email relay. The address check uses `&&`, so any string of 5 or more characters passes. Extra recipients and header lines can be injected into the `ssmtp -t` message, and the user's title text goes into a body branded as Virginia Tech. This allows unlimited spam and phishing with no authentication. | Strict email regex, reject `\r\n,;`, cap the title length, and rate-limit. |

### Medium and Low

**SQL injection (Medium).** Client-controlled values are pasted into SQL in many places:

- `input$runs` at `server.R:545`;
- table names from `input$inTabs` at 2495-2517, 2668 and 9005-9125;
- `mapDsRunList` at 5679;
- run UUIDs at 6034 → `fvsRunUtilities.R:1860`, where a crafted value can delete every run;
- `impRuns` at 8512;
- `rowSelector` at 7613, 7914 and 8116;
- table-editor row IDs and column names at 8040-8083 and in `editDataUtilities.R:65-95`;
- CSV append values and headers at 7565 and 7742;
- stand IDs at `writeKeyFile.R:588` and 671;
- species codes at `sqlQueries.R:4-6`, where stacked statements run because `exqury` splits on `;`.

Use `params=`, `dbQuoteIdentifier` against `dbListTables`, and `dbAppendTable`.

**Cross-site scripting (Medium/Low).** `www/message-handler.js:6` calls `.html(data.message)` with a message that contains the run title (`server.R:6018`), and imported runs carry titles from elsewhere. `renderUI(HTML(...))` also echoes unescaped text: table names, search strings, CSV values, leaflet popups and `projectId.txt` content. Escape with `htmltools::htmlEscape`, or use `.text()`.

**Other Medium issues:**

- `FVSDataConvert/server.R:121-141` builds a script from the uploaded schema and data and runs it through the `sqlite3` command-line shell, so injected sqlite3 shell dot-commands could run. Load through RSQLite instead.
- `rFVS/R/fvsMakeyFile.R:32-42` writes stand IDs and titles containing newlines into keyword records, which allows keyword injection.
- The 10 GB upload limit, unlimited unzip size and uploads that are never deleted allow a disk-exhaustion denial of service.
- `change_project_dir.R:84-92` builds a shell command from directory names. The function is never called.

**Low:**

- `jsCode` handlers that call `eval(message.code)` (`fvsOL/R/ui.R:37`, `FVSDataConvert/ui.R:32`). No current sender passes user data, but the handler should be removed.
- `ui.R:662` and 701 evaluate `isLocal()` when the package is built, not at runtime, so hosted installs show local-only controls.
- Hardcoded host names and server paths. These are configuration, not secrets.

**Secrets.** No credentials were found in the working tree or in git history. The history search covered `password`, `token`, `smtp`, `secret`, `apikey` and `BEGIN` key blocks.

## 3. Correctness bugs

### Growth-and-yield plug-ins: silently wrong projections

| # | Location | Bug | Effect | Fix |
|:-:|:--|:--|:--|:--|
| G1 | `AcadianGY.R:3500-3501`, `3546-3547`, `3639` | `taper.kozak.spp=taper.kozak.spp` (and its Westfall–Scott and `calc_dib` counterparts) are default arguments that refer to themselves. The caller at 3584 does not pass the table. | "promise already under evaluation". With the default Acadian volume option, the run aborts in its first cycle. | Rename the argument, or pass the table explicitly. |
| G2 | `AcadianGY.R:1897` | `qmd.bf = sqrt(ba.bf/(0.00007854*tph.bf))` is 0/0 on plots with no balsam fir. | The NaN propagates to `dexpf` and is coalesced to 0, so **no Acadian mortality** occurs on any plot without fir. | `ifelse(tph.bf>0, …, 0)` |
| G3 | `AcadianGY.R:2346` with `3073` | Ingrowth IDs are `max(tree)` computed per plot plus `row_number()`, while tree IDs run across all plots. | ID collisions make `nrow(tofvs) != ntrees`. `fvsSetTreeAttrs` then skips everything, so that cycle silently uses FVS-NE growth. | Number ingrowth from the maximum ID over the whole list, or add an `is.new` flag. |
| G4 | `customRun_fvsRunAcadian.R:243-259` vs `AcadianGY.R:935-939` | `ops` is modified locally, while the `calc_*` defaults read the global `ops`. | Thinning modifiers are never applied. | Pass `ops` explicitly. Then also fix the double ×100 in percent removal (`AcadianGY.R:820`, 1222, 1454, 1792, 1823, 1833). |
| G5 | `AcadianGY.R:2345-2353` | Ingrowth rows lack the calibration multipliers and maximum DBH and height. | New trees do not grow for the rest of the cycle. `sum(tree.mortba)` becomes NA, so the plot has no mortality. | `left_join(calib)`; `na.rm=TRUE` |
| G6 | `AcadianGY.R:2994-2996` | Ingrowth `cratio` divides HCB in metres by height already converted to feet. | A 5 m HCB on a 10 m tree gives CR 85% instead of 50%. | Compute CR before the unit conversion. |
| G7 | `customRun_fvsRunHi.R:142-143` | The imputation step returns `TRUE ~ cr`, where `cr` is a proportion. | Every tree with a measured CR is written back to FVS as 0–1%. | `cr*100` |
| G8 | `customRun_fvsRunHi.R:137-142` | HCB imputation is called with height in feet; the equation expects metres. | Imputed CR is biased. | Convert after `calc_hcb`. |
| G9 | `HiGy.R:401` | The height increment uses the tree-level `ba` instead of `ba.plot`. The diameter equation at 288 is correct. | Height growth is driven by tree size, not stand density. | `ba = ba.plot` |
| G10 | `HiGy.R:813-852` | `tree.org` has no `cratio`, so out-of-model trees carry NA. | CR is never updated in FVS for any stand that contains snags or non-koa trees. | Add `cratio = round(cr*100,1)`. |
| G11 | `AdirondackGY.R:2205-2211` | `HCB` is reassigned to the prediction at the end of the step before `dHCB` is taken. | ΔHCB is always 0 and observed CR is discarded. This is a regression from V1. | Compute HCB at the current DBH and HT. |
| G12 | `customRun_fvsRunAdirondack.R:39-40` | `MinDBH * CMtoIN` is applied to a value the model treats as cm. | Ingrowth enters at 1.2 cm (0.46 in). | Remove or invert the conversion. |
| G13 | `AdirondackGY.R:2199` with `1993` | `dEXPF` is set to NULL for the "Base Model" mortality option, and `make_fvs_tree` still uses it. | That option crashes the run. | Omit `mort` from `tofvs` in that case. |
| G14 | `customRun_fvsRunAdirondack.R:131` | `M3toFT3` is never defined. | The Kozak volume option fails unless an Acadian run happened earlier in the same session. | Define it with `fvsUnitConversion`. |
| G15 | `AdirondackGY.R:594` | `CutPoint==0` now means "always add full ingrowth", and 0 is the UI default. | Ingrowth probability is ignored. This is a regression. | Restore `IPH*PI` when CutPoint is 0. |

**Medium items in the plug-ins:**

- HI crown recession ignores the predicted `phcb` (`HiGy.R:1094-1099`).
- Birch-group ingrowth is split only across GB, YB and PB, so the shares for RB, SB and spruce are lost (`AcadianGY.R:2320`; `AdirondackGY.R:2254`).
- Hardwoods without an inside-bark Kozak row fall back to the "other softwood" taper (`AcadianGY.R:3509-3513`).
- The mortality multiplier ignores `mortdia1` and `mortdia2`.
- The Acadian "Mortality model" UI option is never passed to `make_ops`.
- Adirondack may push its own `id` column back into FVS (*Plausible*).
- Adirondack ingrowth height and HCB use the record's BA instead of plot BA (`AdirondackGY.R:2296`, 2301).
- The HI mortality clamp compares TPA per plot with TPA per stand (`HiGy.R:842`).
- The metric stand/stock path never joins harvest (`server.R:784-787`), and `sqlQueries_Metric.R` lacks `Create_CmpCalibStats`.
- The hardwood form and risk modifiers would crash on `if()` over a vector if enabled (`AcadianGY.R:749`, 1745).
- The `dg` returned by all three plug-ins is an outside-bark increment. If the FVS `DG` slot is inside bark, growth is inflated by 1/BRATIO. *Plausible*; check against `update.f`.

The unused `AdirondackV1.R` and `AdirondackFunctionsV1.R` redefine `AdirondackGY`, `SUNY.HT` and `Ingrowth.FUN`, so sourcing them would silently replace the current versions.

### Keyword-file generation

| # | Location | Bug | Fix |
|:-:|:--|:--|:--|
| K1 | `fvsOL/R/writeKeyFile.R:710` | `"Design"` plus 53 spaces places the stand weight in columns 60–69, which straddles fields 6 and 7. The weights of replicated stands are mis-specified. | `strrep(" ",44)` |
| K2 | `fvsOL/R/componentWins.R:826` | In the Shelterwood case "Prep=Yes, BA / %SDI, Removal=No", the second ThinSDI uses `swf4/100` (the prep BA) instead of `f12/100`, so `BSDIMax*2` is requested and no cut occurs. | `input$f12` |
| K3 | `writeKeyFile.R:466` | Replicate weights match stand IDs with `grep(fixed=TRUE)`, so stand "12" also matches "112" and "123". | `which(stds==sf)` |
| K4 | `componentWins.R` throughout; `writeKeyFile.R:493` | `sprintf("%10s")` does not truncate, so a value longer than 10 characters (for example `0.333333333`) shifts every later field. Stand IDs longer than 26 characters overflow the StdIdent record. | Format numbers to a width of 10 or less, and validate the length of stand IDs. |
| K5 | `writeKeyFile.R:383-387`, 548, 632 | `kcpVetting` records an END insert without advancing its counters. The `"Addfile:"` test never matches `"AddFile: "`. | Fix the counters and use a case-insensitive match. |

### Application logic (`fvsOL/R/server.R` unless noted)

**High:**

- **Project restore deletes R's session temp directory.** It uses `td <- tempdir()`, then calls `unlink(td, recursive=TRUE)`. It also copies other sessions' uploads into the project (6266-6285). Fix: `tempfile()`.
- **Append CSV crashes with two or more stands.** `is.na(sids) || ...` (7542) errors on R ≥ 4.3 whenever there is more than one stand.
- **`setwd()` without `on.exit` at ten sites** (2716, 4979, 6317, 6691, 7665, 8213, 8250, 8446, 8483, 8747). An early return leaves the shared R process in a temp directory, and later saves from every session go there.

**Medium:**

| Location | Bug | Effect | Fix |
|:--|:--|:--|:--|
| 6591 | `ppackage=` typo | The training spatial data is never restored. | `package=` |
| 7351 | `'Groups' in (select 'Groups' …)` compares two string literals | All existing group rows are deleted. | Quote `Groups` as an identifier, not a string. |
| 6980, 6993, 6999 | `Stand_CN = (select Stand_ID …)` | Every null CN gets the first stand's ID. | `Stand_CN = Stand_ID` |
| 4752-4755 | `Y = c(Y, …) * unitConv` | The metric quick plot rescales earlier stands on every iteration. | Multiply only the new values. |
| 2507-2517 | `rbind` of a 3-column and a 2-column table | The CR-variant group query errors. | Add the missing column. |
| 4870 | `&&` on a vector | The xlsx download fails with two or more runs. | `%in%` |
| 8345 | `class(sf)==` on a length-2 class, and `append` without `x=` | "Add map" always fails. | `inherits()` |
| 8650-8658 | Inverted logic | Imported SpatialData overwrites the project's map. | Invert the condition. |
| 8790, 8927 | `ProjectIsLocked.txt` vs `projectIsLocked.txt` | The lock is never removed, and Change Directory breaks on Linux. | Use the correct case. |
| 6348 | `topgms` can be undefined | Restore from an app-made backup errors after both databases have been disconnected. | Initialise `topgms`. |
| 4927 | Fixed `tozip` directory shared by all sessions | Concurrent downloads mix files across sessions. | `tempfile()` |
| 822-823 | Copy-paste error | The `CmpSummary_East` variable list overwrites `CmpSummary`. | Assign to `CmpSummary_East`. |
| 9005-9071 | `INSTR` substring test | The group "All" collides with "All_Stands". | Token match. |
| 7309-7316 | No `return()` after a failed ATTACH | The database is left in exclusive locking mode. | Add `return()` and reset the locking mode. |

Also Medium:

- `fvsRunUtilities.R:1217-1235` returns while `FVSOut.db` is in exclusive locking mode, which blocks other sessions.
- `writeKeyFile.R` leaks the `.key` connection on six early returns.
- `externalCallable.R:875-918`: `extnAddStands` silently drops AddFiles and reads the non-existent fields `nadd$cmps`/`grps`.
- The stand tables crash when the `ADDFILES` column is absent (`fvsRunUtilities.R:1394`, 1400).
- The `pivot` function errors with two or more grouping factors (`fvsOutUtilities.R:55`).
- `componentWins.R:641` and 755 create observers at package build time, so the Shelterwood auto-update never works.

**Low (about 40 items).**

- **Typos, wrong names and wrong arguments:**
  - `"try.error"` typos (6910; `fvsRunUtilities.R:1771`);
  - `NUL` (8743);
  - `"fvs_climattr"` and `"fvs_climttr"` never match the climate table name (7162, 7398);
  - `class(try)` tested instead of the result (7165-7252);
  - `thisYr` undefined in `writeKeyFile.R:502`;
  - `waityrs` vs `waitYears` (`mkInputElements.R:7`);
  - `list` vs `alist` (`fvsRunUtilities.R:78`);
  - `overwirte=` (4868).
- **Logic and comparisons:**
  - `curX` tested where `curY` was meant (1841, 1851);
  - a condition that is always TRUE at 1235;
  - string comparison of years (4487);
  - `reactiveTimer` reassigned only locally (4589, 4812);
  - jitter discarded (5526, 5529);
  - `charmatch` partial matching of stand IDs (8309).
- **Regex and input handling:**
  - a literal newline inside the table-name regex rejects valid databases (6739, 6779, 6884);
  - user search text is used as a regex (2585, 7924).
- **Edge cases and leaks:**
  - `1:length()` / `1:nrow()` on possibly empty objects;
  - NULL-input `if()` errors;
  - a leaked `dbIcon` connection (6618);
  - `unlink("FVSbin")` without `recursive` (6166).
- **Other files:**
  - `checkMinColumnDefs.R` swallows errors because `return()` inside a `tryCatch` handler does not return from the function;
  - `svsTree.R:251` uses `next` outside a loop;
  - `mkpkeys.R` has a `==` that should be an assignment (272), an `if(is.na(grep))` (25), and drops the last character of a final single-token string (190-191).

### Side apps and rFVS

**Outages today:**

- **FVSPrjBldr never sends new-project email.** `input$version` comes from an input that is commented out, so `if(NULL==…)` errors after the directory is created (`server.R:47`). Each attempt also leaves an orphaned project directory.
- **FVSDataConvert fails every .zip upload.** Line 68 has `paste0(paste0, …)`.

Fixing either outage without also fixing S3 and S14 turns a latent vulnerability into a live one.

**FVSDataConvert:**

- `idƒ` typo in the `ui.R:29` JavaScript, so progress bars never hide.
- Log-file name mismatch.
- Column 3 of every sheet is forced to character.
- Sheet names are used as a regex.
- `if(NA)` when `mdb-schema` returns nothing.

**rFVS:**

- `fvsMakeKeyFile` refers to `morekeywords` (wrong case) and writes only the last row.
- `fvsRun` ignores a stop code given without a year.
- `fvsInteractRun` arms the wrong stop points after a reload.
- `rm` without `-f` in the makefile.
- *To verify against the FVS API source:* `fvsAddTrees` passes species and plot indices as double, and `fvsGetStandIDs`/`fvsGetSpeciesCodes` pass 1-byte buffers to `.C`. These are a heap overflow if the C side copies into the buffer rather than reassigning the pointer.

## 4. Recommended order of work

1. **Close the hosted-server attack surface.** Remove the `eval` on labels and titles (S1, S2). Quote every shell argument (S3). Check every file-path and select input against its server-side list (S4, S5, S9, S10). Load RData into sealed environments (S6). Stop sourcing code from the project directory (S8). Replace `uuidgen` with a cryptographic generator, and lock down the email form (S13, S14).
2. **Fix the silent model errors that change published projections.** Acadian G1–G6 and the double ×100 in G4, HI G7–G10, Adirondack G11–G15, and the DESIGN weight (K1) and Shelterwood (K2) keyword bugs.
3. **Parameterise SQL throughout** and add `on.exit()` guards for `setwd`, database connections and exclusive locks.
4. **Clear the remaining correctness bugs.** Most are one-line fixes.

Each fix needs a small regression stand set run on R and FVS (on firebreather) before and after. The model fixes in step 2 will change outputs, so compare against archived runs before release.
