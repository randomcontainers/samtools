# samtools

Container images with [Samtools](https://www.htslib.org/), the samtools project's programs for sorting, indexing, viewing and converting aligned sequencing reads in SAM, BAM and CRAM files. The images also have HTSlib's `bgzip`, `tabix`, `htsfile` and `annot-tsv`, and the default image adds [BCFtools](https://github.com/randomcontainers/bcftools) for variant calling. Samtools is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when a new Samtools release comes out and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the Samtools developers or Genome Research Ltd. Report problems with the image in this repository and Samtools bugs at [samtools/samtools](https://github.com/samtools/samtools/issues).

## Quick start

Sort a BAM file in the current directory by coordinate and write a CSI index next to it:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/samtools sort -o sorted.bam --write-index in.bam
```

The entrypoint runs `samtools` under `tini`, so the samtools command and its options go after the image name. A few more commands:

```sh
# Print the reads of a region as SAM
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/samtools \
  view sorted.bam chr1:10000-20000

# Count reads by mapping status and flags
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/samtools \
  flagstat sorted.bam

# Convert BAM to CRAM against a reference
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/samtools \
  view -C -T ref.fa -o out.cram sorted.bam

# Sort reads that an aligner writes to standard output
bwa mem ref.fa reads_1.fq reads_2.fq | docker run --rm -i --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/samtools sort -@ 4 -o sorted.bam -
```

The [Samtools manual](https://www.htslib.org/doc/samtools.html) describes every command, and running the image without arguments lists them. `samtools tview` is interactive and needs `docker run -it`; `tview -d T` prints the same view as text.

## bgzip, tabix, htsfile and annot-tsv

These are HTSlib's own programs. `bgzip` compresses text files such as VCF, BED or GFF into the block-gzip format that indexed formats need, `tabix` indexes them, `htsfile` names the format of a file, and `annot-tsv` adds information from the overlapping regions of one tab-separated file to the regions of another. Run them with `--entrypoint`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint bgzip ghcr.io/randomcontainers/samtools regions.bed
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint tabix ghcr.io/randomcontainers/samtools -p bed regions.bed.gz
```

`bgzip regions.bed` replaces `regions.bed` with `regions.bed.gz`; add `-k` to keep the original.

## Remote files and CRAM references

HTSlib is built with libcurl, so samtools also reads files from `https://` URLs and the other protocols libcurl supports, from Amazon S3 (`s3://`) and from Google Cloud Storage (`gs://`). S3 credentials come from `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN`, or from the file named by `AWS_SHARED_CREDENTIALS_FILE`. Google Cloud Storage takes an access token in `GCS_OAUTH_TOKEN`. Pass them with `-e`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY \
  ghcr.io/randomcontainers/samtools view -c s3://<bucket>/sample.bam chr20:1000000-1100000
```

Reading CRAM needs the reference it was written against. Give it with `-T ref.fa`, or set `REF_PATH` and `REF_CACHE` as the [HTSlib reference documentation](https://www.htslib.org/doc/reference_seqs.html) describes. HTSlib does not download references from a server unless `REF_PATH` names one.

## What is in the image

- `samtools` in `/usr/local/bin`, with HTSlib linked in statically. Apart from the distro's C library it loads zlib, libbz2, liblzma, libdeflate, libcurl, OpenSSL's libcrypto and ncursesw (with libtinfo on Ubuntu).
- `bgzip`, `tabix`, `htsfile` and `annot-tsv` in `/usr/local/bin`, from the HTSlib bundled with Samtools.
- The C programs from Samtools' `misc/` directory: `wgsim`, `md5fa`, `md5sum-lite`, `ace2sam`, `maq2sam-short` and `maq2sam-long`.

Not included:

- The scripts in Samtools' `misc/` directory, such as `plot-bamstats`, `seq_cache_populate.pl` and `wgsim_eval.pl`, which need perl, python3 or gnuplot.
- HTSlib's `ref-cache` server, HTSlib's shared library and headers, and the man pages.

`/usr/local/share/randomcontainers/samtools/buildinfo` lists the configure flags, the HTSlib version and the features Samtools and HTSlib were built with.

## Default or slim

Use the default image (`latest`) for general work. It adds BCFtools, so one image can sort and index reads and call variants from them:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh ghcr.io/randomcontainers/samtools \
  -c 'samtools index sorted.bam && bcftools mpileup -Ou -f ref.fa sorted.bam | bcftools call -mv -Oz -o calls.vcf.gz'
```

`slim` has the programs listed above and the libraries they need, without BCFtools. Use it as the base for your own image, or when you do not call variants. The default image of [Minimap2](https://github.com/randomcontainers/minimap2) is built on `slim`, for sorting and indexing the alignments Minimap2 writes.

BCFtools bundles its own copy of HTSlib, and in the default image `bgzip` and `tabix` are BCFtools' copies. When one of the two projects publishes a point release with a newer HTSlib first, these two programs are from a different HTSlib release than `samtools`, `htsfile` and `annot-tsv` until the other project catches up. `samtools --version` and `bcftools --version` print the HTSlib each was built with. BCFtools is built without GSL and perl filters, which keeps it under the MIT/Expat license. The default image is also published as `ghcr.io/randomcontainers/samtools-bcftools`, built in the [samtools-bcftools](https://github.com/randomcontainers/samtools-bcftools) repository with the same contents and a different digest.

## Tags

`<version>` is a Samtools release such as `1.24`, or `1.24.1` for a point release. `<minor>` and `<major>` are its shorter forms, `1.24` and `1`, and follow the newest release in that series. For a release without a point number, such as `1.24`, `<version>` and `<minor>` are the same tag.

| Default (with BCFtools) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Samtools version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Mount the directory that holds your files read-write: `samtools index`, `faidx` and `--write-index` write the index next to the file. `samtools sort` keeps up to 768 MB per thread in memory (`-m`) and writes the rest to temporary files next to the output file, or in `/work` when it writes to standard output; `-T` moves them elsewhere.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no BCFtools, so BCFtools updates do not rebuild it. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Samtools release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/samtools:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends procps \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
ENTRYPOINT []
```

This prepares the image for Nextflow, which needs `bash` and `ps` in a task's container and starts tasks with its own `/bin/bash` command, so the example clears the `samtools` entrypoint. On Alpine, start from `slim-alpine` and use `apk add --no-cache bash procps`. Leave out `ENTRYPOINT []` to keep the slim image's entrypoint, `["tini", "--", "samtools"]`. To pick up new Samtools releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/samtools/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages Samtools needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/samtools:latest \
  --repo randomcontainers/samtools --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/samtools:latest --format '{{ json .SBOM }}'
```

The SBOM does not list HTSlib, because it is compiled into each program; `samtools --version` and `buildinfo` name its version. Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`. After compiling, it runs the Samtools and HTSlib test suites.

## Updates

The project checks the releases of [samtools/samtools](https://github.com/samtools/samtools/releases) every 15 minutes and follows the 1.x series. A release is picked up once it is 24 hours old. The Samtools developers publish no checksum file or signature, so the tarball is checked against the SHA-256 digest GitHub records for the release asset. The new version and the tarball's SHA-256 are then committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built, and a point release of an older series, such as 1.23.2 after 1.24, is not built.

HTSlib security fixes reach the image with the Samtools release that bundles them. Since HTSlib 1.15.1, every HTSlib point release has come with a Samtools release on the same day or the next. In the default image, `bgzip` and `tabix` follow BCFtools releases instead, which can lag: HTSlib 1.22.1 came out in July 2025, and the next BCFtools release followed five months later.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new BCFtools image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t samtools:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

Samtools and HTSlib are under the MIT/Expat license, except HTSlib's `cram/` directory, which is under three-clause BSD licenses from Genome Research Ltd. and the Medical Research Council. HTSlib's htscodecs library is BSD-3-Clause. Other code compiled in keeps its own terms:

- MIT: code from the Broad Institute, the Massachusetts Institute of Technology, Heng Li, Attractive Chaos (klib), Intel, Illumina, Peter Cock (The James Hutton Institute), Pierre Lindenbaum (Institut du Thorax, INSERM U1087, Université de Nantes), Florian Breitwieser and the University of California, Los Angeles.
- BSD-3-Clause: any code in `samtools consensus` that remains from the Staden Package, by the Medical Research Council.
- BSD-2-Clause: the LZ4 compressor that `samtools markdup` uses for temporary files, by Yann Collet.
- CC0-1.0: `rANS_byte.h` and `rANS_word.h` in htscodecs, from Fabian Giesen's ryg_rans.
- Public domain: HTSlib's `md5.c` by Alexander Peslyak, the parts of `hts_time_funcs.h` taken from NetBSD's `localtime.c`, and `c_range_coder.h` in htscodecs by Eugene Shelwien.

`/usr/local/share/randomcontainers/samtools/licenses/` holds Samtools' `LICENSE`, HTSlib's `LICENSE` as `LICENSE.htslib`, htscodecs' `LICENSE.md` as `LICENSE.htscodecs.md`, the LZ4 license as `LICENSE.lz4`, the Staden Package license from `bam_consensus.c` as `LICENSE.staden`, and `NOTICES`, which the build fills with the first copyright comment of each C file outside the test suites when that comment names a holder other than Genome Research Ltd. The source tarball URL is in `/usr/local/share/randomcontainers/samtools/source`. The image's license label is `MIT AND BSD-3-Clause AND BSD-2-Clause AND CC0-1.0 AND LicenseRef-Public-Domain`; the default image adds BCFtools' terms, listed in its [README](https://github.com/randomcontainers/bcftools#licenses). The Ubuntu and Alpine packages in the image keep their own licenses, for example the curl license for libcurl and Apache-2.0 for OpenSSL. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
