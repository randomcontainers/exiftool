# exiftool

Container images with [ExifTool](https://exiftool.org/), the command-line tool for reading, writing and removing metadata in images, video, audio and documents. ExifTool is installed from the release tarball on Ubuntu or Alpine and runs on the distro's Perl. The default image adds ImageMagick, so an image can be converted or resized and its metadata copied back in the same container. The images are rebuilt when ExifTool publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by Phil Harvey, who develops ExifTool. Report problems with the image in this repository and problems with ExifTool itself on the [ExifTool forum](https://exiftool.org/forum/).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/exiftool -a -G1 -s photo.jpg
```

The same images can also be pulled as `randomcontainers.com/exiftool`.

Remove the GPS position from every JPEG in the current directory:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/exiftool -overwrite_original '-gps*=' -ext jpg -ext jpeg .
```

`'-gps*='` deletes every GPS tag, including the XMP copies that photo editors often write and that `-gps:all=` leaves in place. Keep the quotes so the shell does not expand the `*`.

The entrypoint runs `exiftool` under `tini` in `/work`, so file names are relative to the directory you mount. Without arguments, the image prints the ExifTool version and the optional Perl modules it found (`exiftool -ver -v`). A few more commands:

```sh
# Rename photos after the date they were taken, for example 20260926_120000.jpg
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/exiftool '-FileName<DateTimeOriginal' -d '%Y%m%d_%H%M%S%%-c.%%e' .

# Write the metadata of every JPEG in a directory tree to a JSON file
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/exiftool -json -r -ext jpg -ext jpeg . > metadata.json

# Resize a photo with ImageMagick (default image only), then copy its metadata
# to the smaller copy, without the GPS position
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint magick \
  ghcr.io/randomcontainers/exiftool photo.jpg -resize 1600x1600 -strip small.jpg
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/exiftool -overwrite_original -tagsFromFile photo.jpg -all:all '-gps*=' small.jpg
```

ExifTool reads `.ExifTool_config` from the directory in `EXIFTOOL_HOME`, so `-e EXIFTOOL_HOME=/work` picks up a config file in the mounted directory. You can also pass `-config my.config` as the first argument. The [ExifTool documentation](https://exiftool.org/exiftool_pod.html) covers the options and tag names.

## What is in the image

| | slim | default |
|---|---|---|
| ExifTool, with the argument, config and format files from its tarball | yes | yes |
| The distro's Perl and Archive::Zip, which ExifTool needs for Office, OpenDocument and other ZIP-based files | yes | yes |
| ImageMagick | no | yes |

The `exiftool` script and its `lib` directory are installed in `/usr/local/lib/exiftool`, and `/usr/local/bin/exiftool` links to the script. The files for ExifTool's `-@` and `-p` options are in `arg_files` and `fmt_files` there, for example `-p /usr/local/lib/exiftool/fmt_files/gpx.fmt` to write a GPS track as GPX. `config_files` has example config files.

The other optional modules ExifTool uses ship with Perl and are in the image, among them Compress::Zlib for compressed PNG, PDF and DNG data, Digest::MD5 and Digest::SHA, and Time::Piece for parsing dates. Not included: Compress::Raw::Lzma (encoded 7z archives), IO::Compress::Brotli (compressed JPEG XL metadata), File::StatX (`FileCreateDate` on Linux), Unicode::LineBreak, and POSIX::strptime, for which Time::Piece stands in. Alpine has no packages for the first two; see [Extending the slim image](#extending-the-slim-image) to add them on Ubuntu.

ImageMagick in the default image is the [randomcontainers/imagemagick](https://github.com/randomcontainers/imagemagick) slim build. It has no Ghostscript, so it does not read PDF or PostScript files.

## Default or slim

The default image (`latest`) adds ImageMagick to ExifTool, for jobs that change the pixels and the metadata together, such as resizing photos and keeping their EXIF data. Most ExifTool jobs only read or write metadata, and `slim` has everything they need. Use `slim` for that, and as the base when you build your own image.

The default image is also published as `ghcr.io/randomcontainers/exiftool-imagemagick`, built in the [exiftool-imagemagick](https://github.com/randomcontainers/exiftool-imagemagick) repository with the same contents and a different digest.

## Tags

`<version>` is an ExifTool release such as `13.59`. `<major>` is its major version, `13`, and follows the newest release in that series.

| Default (with ImageMagick) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<major>` | `slim`, `<version>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current ExifTool version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are built natively on GitHub-hosted runners, without emulation. ExifTool is written in Perl, so the same ExifTool files are installed on every platform.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

When ExifTool changes a file, it writes the new version next to it and keeps the old one as `<name>_original`. `-overwrite_original` skips the copy. Either way, the container user needs write access to the directory, not only to the file.

Containers run in UTC unless you set `TZ`. ExifTool reads dates without a time zone, such as `DateTimeOriginal`, as local time, which matters when you set file dates from them (`-FileModifyDate<DateTimeOriginal`). A POSIX zone string works without a time zone database, for example `-e TZ=CET-1CEST,M3.5.0,M10.5.0/3`.

## Untrusted files

ExifTool reads more than 300 file types and has had code execution bugs, for example CVE-2021-22204 in its DjVu parser and CVE-2022-23935 with file names ending in `|`. For files from unknown sources, mount them read-only and take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --pids-limit 64 \
  ghcr.io/randomcontainers/exiftool -json upload.jpg
```

Mount only the directory the job needs. The images pick up a new ExifTool release about a day after it is tagged.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new ExifTool release and are rebuilt when the base image changes. ExifTool runs on the distro's `perl`, so Perl modules from distro packages work with it. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/exiftool:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends libcompress-raw-lzma-perl libio-compress-brotli-perl \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

The entrypoint is `["tini", "--", "exiftool"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new ExifTool releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/exiftool/` holds the version, the source URL, the build details, the license files and `runtime-deps`, the list of distro packages ExifTool needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/exiftool:latest \
  --repo randomcontainers/exiftool --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/exiftool-imagemagick` are built in that repository, so verify them with `--repo randomcontainers/exiftool-imagemagick` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/exiftool:latest --format '{{ json .SBOM }}'
```

The build checks the tarball against the SHA-256 recorded in `package.yml` and runs ExifTool's test suite on the distro's Perl before it installs anything.

## Updates

The project checks the [tags of exiftool/exiftool](https://github.com/exiftool/exiftool/tags) every 15 minutes. ExifTool tags every release, including those that [exiftool.org](https://exiftool.org/history.html) lists as development releases, and the images follow all of them. A release is picked up once its tag is 24 hours old. Its tarball is downloaded from [SourceForge](https://sourceforge.net/projects/exiftool/files/) and checked against the `checksums-<version>.txt` file published next to it, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new ImageMagick image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t exiftool:local .
```

Use `Dockerfile.alpine` for the Alpine image. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

ExifTool is free software under the same terms as Perl: the Artistic License or the GNU General Public License, version 1 or later (SPDX `Artistic-1.0-Perl OR GPL-1.0-or-later`). Its city database for reverse geocoding, `lib/Image/ExifTool/Geolocation.dat`, is derived from [GeoNames](https://www.geonames.org/) data and licensed under CC BY 4.0. `lib/Image/ExifTool/BZZ.pm`, the decompressor for DjVu files, is based on DjVuLibre code under the GNU GPL, version 2 or later, and is covered by that license; its copyright notices are in the file. The image's license label is `(Artistic-1.0-Perl OR GPL-1.0-or-later) AND GPL-2.0-or-later AND CC-BY-4.0`.

The license files are in `/usr/local/share/randomcontainers/exiftool/licenses/`: `COPYRIGHT`, ExifTool's copyright notice from the tarball's README; `Artistic` and `GPL-1`, the texts of the two Perl licenses; `GPL-2`, for `BZZ.pm`; `CC-BY-4.0`; and `GeoNames.NOTICE`, the attribution for the city database. The tarball does not include the license texts, so they come from [`licenses/`](licenses/) in this repository. `Artistic` and `GPL-1` are the `Artistic` and `Copying` files of the Perl 5.40.1 source, and `GPL-2` is the text published by the Free Software Foundation.

Every ExifTool version has a GitHub release in this repository, named `v<version>`, with the exact `Image-ExifTool-<version>.tar.gz` the image was built from. ExifTool runs from its source files, which the build copies without changes. The download URL is in `/usr/local/share/randomcontainers/exiftool/source`.

The default image also contains ImageMagick, under the ImageMagick license; see [randomcontainers/imagemagick](https://github.com/randomcontainers/imagemagick). The Ubuntu and Alpine packages in the images, Perl among them, keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
