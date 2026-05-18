# muscle

TAFFISH wrapper for [MUSCLE 5](https://github.com/rcedgar/muscle), software for
multiple sequence alignment, alignment ensembles, and Reseek-based structure
alignment inputs.

This app packages upstream MUSCLE `v5.3` as `5.3-r1`. The container builds from
the upstream source tag rather than copying the release binaries, because the
official `muscle-aarch64.v5.3` Linux asset fails immediately on `linux/arm64`
with a pointer-size assertion. The TAFFISH build applies one small portability
patch: Linux `__aarch64__` is recognized as a 64-bit platform. It also removes
`-march=native` from the generated Makefile so the static binary is not tied to
the build host CPU.

## Install

```sh
taf install muscle
```

Then run:

```sh
taf-muscle --help
taf-muscle --version
taf-muscle -- -version
taf-muscle muscle -version
```

`taf-muscle --help` shows TAFFISH wrapper help. Use `--` when the first upstream
argument starts with `-`, for example `taf-muscle -- -h` or
`taf-muscle -- -version`. Command mode is also available:
`taf-muscle muscle -version`.

## Examples

Align FASTA input:

```sh
taf-muscle -- -align input.fa -output aln.afa
```

Use Super5 for large inputs:

```sh
taf-muscle -- -super5 input.fa -output aln.afa
```

Generate a small diversified ensemble in EFA format:

```sh
taf-muscle -- -align input.fa -diversified -replicates 100 -output ensemble.efa
```

Extract a representative alignment and split an EFA file:

```sh
taf-muscle -- -maxcc ensemble.efa -output maxcc.afa
taf-muscle -- -efa_explode ensemble.efa
```

Structure alignment is supported by upstream MUSCLE 5.3 when users provide
Reseek-generated `mega` and, for `-super7`, distance-matrix inputs:

```sh
taf-muscle -- -align structs.mega -output structs.afa
taf-muscle -- -super7 structs.mega -distmxin structs.distmx -reseek -output structs.afa
```

This app does not bundle `reseek`; generate those files with a separate Reseek
installation or app.

## Platform

The image is built for native `linux/amd64` and `linux/arm64`. The MUSCLE binary
is statically linked, so it has no dynamic library dependency beyond the Linux
kernel ABI.

Package contents are intentionally small: the runtime image ships the `muscle`
executable plus upstream README, LICENSE, a version marker, and a
`TAFFISH_PATCHES` note. A source scan found no runtime `system`, `popen`, or
`exec`-style helper calls in the MUSCLE C++ code; the `vcxproj_make.py` helper is
used only while building the image and is not part of the runtime path.

Runtime version output is:

```text
muscle 5.3.linux64 [v5.3-taffish-r1]
```

The `linux64` platform label is MUSCLE's upstream label for 64-bit Linux builds;
it is used for both amd64 and arm64 here.

## Smoke Coverage

Smoke tests check:

- package version marker and upstream `muscle -version`;
- help text for FASTA alignment and EFA ensemble features;
- static-link status with `ldd`;
- a tiny `-align` FASTA-to-AFA path;
- a tiny `-super5` path;
- a tiny ensemble path using `-diversified`, `-maxcc`, `-resample`, and
  `-efa_explode`.

These checks verify packaging and representative core paths. They do not replace
large biological benchmarks, memory-scaling tests, or validation of
Reseek-generated structure-alignment inputs.

## Boundaries

This app packages MUSCLE only. It does not include Reseek, PDB/mmCIF fetching,
tree-building tools, downstream phylogenetic software, or large benchmark
datasets. MUSCLE can consume structure-alignment inputs generated elsewhere,
but this app does not create those inputs by itself.

## Upstream

- Upstream repository: <https://github.com/rcedgar/muscle>
- MUSCLE 5 home page: <https://drive5.com/muscle5/>
- Release: <https://github.com/rcedgar/muscle/releases/tag/v5.3>
- License: GPL-3.0
- Citation: Edgar RC. 2022. Muscle5: High-accuracy alignment ensembles enable
  unbiased assessments of sequence homology and phylogeny. Nature
  Communications 13, 6968.
- DOI: <https://doi.org/10.1038/s41467-022-34630-w>
- PMID: 36379955

MUSCLE 5.3 also includes Muscle-3D support described by Edgar and Tolstoy
2024, bioRxiv DOI <https://doi.org/10.1101/2024.10.26.620413>.
