# Methods

Facts of the measurement reported in the thesis. The data in `results/` come from a single run of
the `measure` workflow ([`.github/workflows/measure.yml`](.github/workflows/measure.yml)).

## Measured run

- Workflow run 37123731010, attempt 1, 3 October 2026, 12:42–15:58 UTC.
- 112 jobs (7 project/variant units × 4 scenarios × 4 shards), 5 trials per job:
  224 seed builds and 1680 measured builds, N = 20 per project × variant × scenario × mode (84 cells).
- No build failed and no build was flagged. The empty-cache check passed and the builder was removed
  in all 1904 builds; every measured build matched the expected cache behaviour.
- A smoke test (N = 2 per cell) preceded the measurement. Its only consequence was a revised
  declaration of the expected cache behaviour of the runtime-stage `COPY --from` steps (see *Expected
  cache behaviour*). Its data are not part of this package.

## Versions

| Item | Value |
|---|---|
| Runner image | ubuntu24 / 20260927.320.1, GitHub-hosted standard runner of a public repository |
| vCPUs / RAM | 4 / ≈ 16 GB |
| CPU models | AMD EPYC 7763 (1020 builds), AMD EPYC 9V74 (357), AMD EPYC 9V45 (306), Intel Xeon Platinum 8573C (119), Intel Xeon 6973P-C (68), Intel Xeon Platinum 8370C (34) |
| Kernel | 6.17.0-1022-azure |
| Docker Engine / storage driver | 28.0.4 / overlay2 |
| Buildx | v0.37.2 (binary sha256 `982ca20490b45ed1ec8d99795974d3d874a358f75938c9c237305010e6b7e548`) |
| BuildKit | v0.33.1 |
| Cache manifest media type | `application/vnd.oci.image.manifest.v1+json` |

Pinned images (linux/amd64 manifest digests; mirrored byte-for-byte to ghcr.io, see [`pins.json`](pins.json)):

| Image | Upstream | Tag | Digest |
|---|---|---|---|
| buildkit | docker.io/moby/buildkit | v0.33.1 | `sha256:98cc6a3fc46220d00f8224ae483f3274fc874e9be8d7dd1e2e2c5481209228b5` |
| buildx-bin | docker.io/docker/buildx-bin | 0.37.2 | `sha256:9e2e0742054896c699b509239732b91138f781845e764bb4853885f26e142ad5` |
| debian | docker.io/library/debian | trixie-20260918-slim | `sha256:7792b1f7702a86946cd518db72b6a407302c3e9bc1635634368b878189e8221c` |
| golang | docker.io/library/golang | 1.26.8-trixie | `sha256:af00f232205a01baf323bddc5bb4a4ea2e4cbf49e8ccc988beb2558d0f9c882a` |
| node | docker.io/library/node | 20.20.2-trixie-slim | `sha256:1694ccde5ea9efb3060bb8612b1f287256061ff2a04d75dc1b71f57cb7239520` |
| python | docker.io/library/python | 3.12.15-slim-trixie | `sha256:6b1f85a08c199d29d5b6d71ab9c27bd5b3b393492e01216a15758ff69c4be8b8` |

The projects are pinned to their release tags (Uptime Kuma 1.23.17, NetBox v4.7.2, Caddy v2.11.7);
the commit of each tag is verified at clone time ([`projects/`](projects)).

## Synthetic variants

| Variant | Bytes per step | SHA-256 iterations per step | Work steps | Observed CPU s per step (median, min–max) |
|---|---|---|---|---|
| small-short | 1 MiB | 2 000 000 | 4 | 0.94 (0.47–1.14) |
| small-long | 1 MiB | 60 000 000 | 4 | 28.40 (14.31–32.63) |
| large-short | 256 MiB | 2 000 000 | 4 | 0.85 (0.47–1.06) |
| large-long | 256 MiB | 60 000 000 | 4 | 28.26 (14.39–31.04) |

## Expected cache behaviour

Observed with BuildKit v0.33.1 and encoded in [`harness/scenario.py`](harness/scenario.py):
`max` reuses every builder step before the invalidation point; `min` stores results only for the final
image, so builder steps are reused only in S0; runtime-stage `COPY --from` steps are content-addressed
and are reused only if a full rebuild of the builder stage reproduces the copied files byte for byte.
A full rebuild of Uptime Kuma (npm ci + vite build) and of the NetBox virtual environment (pip install)
is not byte-reproducible, so their runtime copies are rebuilt after S1–S3 in `min` mode and after S3
in `max` mode. `FROM` vertices are never reported as cached and are not counted.

## Notes on the data

- **Hardware.** Every VM had 4 vCPUs and ≈ 16 GB RAM, but six CPU models occurred. Pairing within a
  trial (same VM, random order) removes this from the paired differences; it widens the IQRs of single
  modes in some cells (for example uptime-kuma S0 none, netbox S3 none, synthetic small-long S3 none).
- **One missing value.** For caddy S1, shard 3, trial 1, mode max the cache export succeeded, but reading
  the new reference back from the registry immediately afterwards failed transiently, so its cache-size
  fields are null with that reason. The cell's cache-size median uses 19 values.
- **Uploaded bytes.** BuildKit v0.33.1 reports no byte totals for cache uploads, so uploaded bytes are
  measured only by the interface tx counter (includes protocol overhead).

## Design decisions

- **Output.** All modes use `--output type=docker` (the image is loaded into the runner's Docker
  Engine, its size read, then deleted; never pushed), because a CI build normally needs its image.
- **Base images.** Debian trixie variants pinned by digest; Node.js 20 for Uptime Kuma 1.23 (the
  release's own CI tests Node 20); Python 3.12 for NetBox v4.7.2 (requires ≥ 3.12); Go 1.26.8 for
  Caddy v2.11.7.
- **Project builds.** Uptime Kuma: `CYPRESS_INSTALL_BINARY=0` (a development dependency not needed for
  the build); build = `npm run build && npm prune --omit=dev`. NetBox: build = documentation build
  (as in the upstream `upgrade.sh`) + `collectstatic` with the upstream test configuration, so no
  source file is added.
- **Seed builds** run in random order (min/max) and carry the scenario of their job.
- **Analysis.** H2 is tested with a one-sided Spearman correlation between the cache hit ratio and the
  relative time saving. The two-sided Wilcoxon tests of the S1–S3 cells are reported as a supplementary
  family without a decision. As a robustness check against the symmetry assumption of the Wilcoxon
  signed-rank test, every Wilcoxon test is repeated as an exact sign test with the same alternative and
  Holm family; the decisions are the same in all 112 Wilcoxon tests (H1, H3 and the S1–S3 family).
- **H4.** A cell counts as having a shorter cache build when the 95% bootstrap CI of the median paired
  difference lies entirely below 0 (20 cells; 8 meet the rule). The lenient reading (median saving > 0
  only: 21 cells, 9 meet the rule) is reported in `results/hypotheses.csv` as sensitivity. The r_a range
  includes the free GitHub Container Registry (r_a = 0); `ra_star_within_paid` and
  `ra_star_within_all_runner_prices` in `results/cost_breakeven.csv` give the narrower and the wider reading.
