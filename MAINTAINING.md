# Maintaining ProsperoWake

ProsperoWake targets PS5. SwitchWaker and the renderer dependencies are pinned
submodules; changes to their code are stored as patches in this repository.

## Where to make changes

| Area | Files |
| --- | --- |
| Gameplay options and controls | `ps5/src/features/` |
| Hooks into game code | `ps5/src/game_hooks/` |
| Disc regions, controller assets and startup integration | `ps5/src/integration/` |
| Options menu and FPS display | `ps5/src/presentation/` |
| Menu tabs, sections and ordering | `ps5/include/pc/pc_setting_presentation.h` |
| Menu translations | `ps5/src/presentation/pc_menu_translations.inc` |
| Rendering effects and shaders | `ps5/aurora/gfx/` |
| Console platform code | `ps5/src/platform.*`, `ps5/src/startup.c`, `ps5/sdl/` |
| Build configuration | `ps5/cmake/` |
| Build and maintenance scripts | `ps5/tools/` |

Edit these files directly. The PS5 menu and FPS display replace their upstream
counterparts, so upstream UI fixes need to be reviewed for the PS5 versions too.

## Changing upstream code

Edit SwitchWaker in `vendor/switchwaker/`, or a dependency in its `.deps/`
checkout. Export the changes before committing the parent repository:

```sh
python3 -B ps5/tools/manage-patches.py --write --target switchwaker
bash ps5/tools/check-source.sh
```

Choose only the targets you changed. For example, use `--target switchwaker aurora`
when both need exporting.

SwitchWaker and Aurora patches live in `ps5/patches/switchwaker/` and
`ps5/patches/aurora/`. Each directory has:

- `series`: the order in which patches apply.
- `topics.json`: which patch owns each upstream file.
- The exported `.patch` files.

Assign newly patched files in `topics.json` before exporting. Each upstream file
belongs to one patch. SDL, Dawn and Abseil use individual patch files under
`ps5/patches/`.

Do not edit exported patch bodies or generate them with a plain `git diff`.
The exporter uses full object hashes and includes new files. It checks that the
patch series reproduces the checkout by comparing file contents. Aurora's series
applies directly to its pinned revision.
It includes the retained SwitchWaker fixes; the older series inside SwitchWaker
is not applied separately.

`check-source.sh` checks source layout, assets, translations, dependency pins and
patch consistency. It does **not** compile the game or test it on a console.
Host tests that compile code have separate `check-*.sh` scripts. Controller input
regressions run with `python3 -B ps5/tests/controller-input.py`; this compiles a
small host test with mocked console calls, not the game.

## Dependency versions

Git submodule entries pin exact commits. `.gitmodules` supplies their URLs and
paths. The build initializes missing dependencies and checks existing revisions;
it does not reset populated checkouts to discard local edits.

```sh
python3 -B ps5/tools/setup-submodules.py --check
```

Dependency pins must agree with the versions declared by SwitchWaker and Aurora.
Setup applies the SwitchWaker overlay before reading its Aurora declaration.
The Payload SDK is the exception: `ps5/tools/setup-sdk.sh` pins and installs it
from the separate `PS5_PayloadSDK` checkout. `PS5_Vulkan` supplies the RADV archive,
link recipe, packaging tool and runtime.

## Updating SwitchWaker or a dependency

1. Commit or back up your work, including unexported edits inside submodules.
2. Use a separate checkout with its own dependency directories. Select the new
   revision and stage its submodule entry in the parent repository; pin checks
   read the staged revision.
3. Check the upstream dependency declarations and update affected pins together.
4. Apply the existing patches to the new revisions and resolve conflicts. Compare
   upstream gameplay and controls with the PS5 implementation before retaining
   duplicate features. Remove fixes that upstream now provides.
5. Export the revised patches, run `check-source.sh`, and review the pin and patch
   changes together.
6. Build and test on PS5 before release. Check both disc regions, save/load,
   controller input, menus and any affected rendering or gameplay features.

Keep upstream files for unsupported platforms unless they interfere with the PS5
build. Excluding them from the target usually costs less maintenance than deleting
them from the dependency.

## USA and PAL support

One executable supports revision-0 `GZLE01` and `GZLP01` discs. It uses PAL game
code and class layouts, selecting resource and save formats at runtime. Region
checks belong where disc data differs; ordinary gameplay does not need separate
USA and PAL implementations.

When changing region support, test title and file selection, PAL languages,
USA English text, controller prompts and saving in the `USA` or `EUR` directory.

## Controller artwork

Xelu artwork, layout rules and resource checksums live in
`ps5/assets/controller-ui/`. Edit the PNGs or `layout.json`, then regenerate:

```sh
python3 -B ps5/tools/generate-controller-ui.py
python3 -B ps5/tests/controller-ui.py
```

Commit both the source changes and generated outputs. The generator uses the
Python standard library; it needs no Pillow installation, mod ZIP or game ISO.
Normal builds package `res/resources-us.bin`, `res/resources-pal.bin` and
`res/menu-prompts.rgba`.
Validation manifests remain in the repository.

The bundles contain Xelu textures, generated zoom arrows and layout instructions.
At runtime, `pc_ui_transform.cpp` edits matching layouts
loaded from the player's disc and combines the disc's camera-eye texture with
the R3 prompt. It leaves the original resources unchanged. The separate host
check `bash ps5/tools/check-controller-ui.sh` compiles and tests these transforms
using synthetic data.

## Building and releasing

Build prerequisites and installation instructions are in the [README](README.md).
For normal development:

```sh
bash ps5/tools/build.sh
```

Incremental builds reuse dependencies and generated metadata. For a clean rebuild,
stop any active build and run `bash ps5/tools/build-clean.sh`. It preserves source
checkouts and skips directories containing player data.

For a release, build from committed source and test the resulting title on PS5.
Normal builds record revisions and input hashes in `BUILD_INPUTS.json` without a
recursive dependency audit. Release packaging requires the checkout to match the
packaged build and verifies dependency contents against their pins and patches.
Export and commit dependency edits before building a release.
The packaging audit checks the current checkout, not past compilation inputs.
An older binary with an explicitly failed build-time audit must be rebuilt;
cleaning the checkout alone does not clear that failure. Use the normal build command.
Check shader caches when changing shaders or the driver.

Each normal build invocation reserves the next number in the build directory's
`param.json`. The menu and packaged title use that number. Cleaning the build
directory removes the local counter, so check for an existing release tag before
publishing; the counter is not a repository-wide release registry.

```sh
bash ps5/tools/package-release.sh
```

This reads the build number from the packaged title and creates
`dist/ProsperoWake-build-N.zip` (for example, `ProsperoWake-build-38.zip`).
Use `Build 38` as the GitHub release title and `build-38` as its tag, targeting the
source commit recorded in `SOURCE.txt`. Packaging checks the executable
against its recorded build metadata and rejects uncommitted port source or player
data in the title folder. It also rejects local home-directory paths in packaged
files, including the executable. Compiler path maps preserve relative source
locations; RADV provenance exports revision/hash fields only. Existing objects
recompile when these flags change. Keep attribution and licences with the package.

The packaging script creates an archive locally; it does not tag, push or upload.
Publish the corresponding source and dependency patches with the release.
GitHub's automatic source archives do not include submodule contents.

### Publish and update the catalog

Publish the release with its existing ZIP and reviewed release notes. Verify
that GitHub marks it as published, that the asset upload is complete, and that
the release asset's SHA-256 matches the local ZIP. Keep published assets and
release tags unchanged after the catalog pins them.

Update only `apps/PPSA99309-mgoodings.json` in a pull request to
[ps5-homebrew-catalog](https://github.com/darkxex/ps5-homebrew-catalog): set
`version` to the release tag, update `artifact_url` and `sha256` from the
published asset, and pin `icon_url` to the new tag. The unsuffixed
`PPSA99309.json` belongs to PS5_SwitchWaker. From the catalog checkout, run
`python3 -m catalog check` and
`python3 -m catalog verify PPSA99309-mgoodings`; review the reported content
version as well as the pass/fail result. After merge, verify the deployed
listing and API; a merged PR does not mean the website has finished deploying.

### Follow-up before the next release: catalog content version

Build 76 ships `01.000.076` in the ZIP's `PPSA99309/sce_sys/param.json`,
but the committed `ps5/sce_sys/param.json` at `build-76` still contains
`01.000.028`. The build script treats the committed file as a template and
increments a generated copy in the build directory; packaging uses that copy.

The catalog reads the repository's `sce_sys/param.json` at the release tag
before trying the ZIP. Its verification for Build 76 reported `01.000.028`.
The ProsperoStore website successfully lists Build 76; the effect of this
metadata mismatch on console update detection has not been verified.
Keep the published Build 76 ZIP and tag unchanged.

Before the next release:

- Fix the version workflow so catalog metadata reflects the packaged version.
  Account for the build script incrementing its generated version; updating
  the template alone does not guarantee that the next package matches it.
- Add a release check that compares the catalog-resolved content version with
  the ZIP's `contentVersion` and rejects a mismatch. A passing catalog check
  alone is insufficient: Build 76 passed while reporting the template version.
- Verify the release tag, ZIP metadata and menu agree, and that the content
  version is higher than the previous package's version.
- After the catalog update is merged and deployed, check the live store API's
  `version` and `content_version` separately, and confirm update detection on
  a console with the previous release installed.

## Other tools

- `build-runner.sh <lima-instance>` starts an optional build runner from macOS.
  It watches `.build-runner/request`, runs `build.sh` in Lima and shows progress
  in its terminal. The log and exit status are in `.build-runner/build.log` and
  `.build-runner/result`. It uses LLVM 22 and the VM's
  `~/.cache/prosperowake-build` directory by default. Stop it with Ctrl-C.
- `deploy.sh` uploads the packaged title over FTP using PS5_Vulkan's deployment tool.
- `run.sh` launches it and captures console logs through PS5_Vulkan.
- `check-*.sh` runs host tests for individual features; these are separate from normal builds.
- `ps5/art/make-title-art.sh` regenerates the PS5 icon and backgrounds. This optional
  artwork tool needs Pillow and NumPy; building the game and generating controller
  prompts do not.

Run the other tools in Linux. Set `PS5_VULKAN_DIR`,
`PS5_PAYLOAD_SDK_FORK` and `PS5_BUILD_DIR` when using a different directory layout.
