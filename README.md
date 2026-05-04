# smo_scottish_lidar

A pure Ruby gem for listing and downloading Scottish Public Sector LiDAR data from the [Registry of Open Data on AWS](https://registry.opendata.aws/scottish-lidar/). Built by Sebastian Madrid Ontiveros to support hydraulic modellers in Scotland working on 1D-2D flood risk assessments and model build workflows.

No external dependencies. No AWS CLI. No credentials. Uses only Ruby stdlib (`net/http`, `uri`, `fileutils`). Compatible with InfoWorks ICM 2027 embedded Ruby.

---

## What is this data?

The Scottish Government has made LiDAR survey data publicly available through an S3 bucket (`srsp-open-data`). The dataset covers most of Scotland across five survey phases plus a dedicated Outer Hebrides survey. Each phase includes:

- **DSM** - Digital Surface Model (includes buildings, trees, structures)
- **DTM** - Digital Terrain Model (bare earth, vegetation removed)
- **LAZ** - Raw LiDAR point cloud in compressed LAS format

Files are organised by OS National Grid square (e.g. NS, NT, NO, NN) and are free to download.

| Phase | Area covered |
|---|---|
| phase-1 | Central Scotland |
| phase-2 | South and East Scotland |
| phase-3 | North and West Scotland |
| phase-4 | Additional coverage |
| phase-5 | Latest survey phase |
| outer-hebrides | Western Isles (25cm and 50cm resolution) |

---

## Installation

```sh
gem install smo_scottish_lidar
```

```ruby
require "smo_scottish_lidar"
```

---

## Quick start

```ruby
require "smo_scottish_lidar"

# List all Phase 1 DSM tiles in the NS grid square
lister = SmoScottishLidar::Lister.new
lister.summary("phase-1", "dsm", grid_square: "NS")

# Download a single tile
downloader = SmoScottishLidar::Downloader.new
downloader.download_file("phase-1", "dsm", "NS56_1M_DSM_PHASE1.tif",
  destination: "/tmp/lidar"
)

# Batch download all NS tiles for Phase 1 DSM
downloader.download("phase-1", "dsm",
  destination: "/tmp/lidar/phase-1/dsm",
  grid_square: "NS"
)
```

---

## API reference

### `SmoScottishLidar::Lister`

Lists available files from the S3 bucket. All filtering is done client-side after fetching the S3 listing.

```ruby
lister = SmoScottishLidar::Lister.new(verbose: false)
```

#### `lister.list(phase, type, grid_square: nil, resolution: nil)`

Returns an `Array<Hash>` of matching files. Each hash contains:

| Key | Type | Description |
|---|---|---|
| `:key` | String | Full S3 object key |
| `:filename` | String | Bare filename (e.g. `NS56_1M_DSM_PHASE1.tif`) |
| `:size` | Integer | File size in bytes |
| `:last_modified` | String | ISO 8601 timestamp |

```ruby
files = lister.list("phase-1", "dsm", grid_square: "NS")
files.each { |f| puts "#{f[:filename]}  #{f[:size]} bytes" }
```

#### `lister.summary(phase, type, grid_square: nil, resolution: nil)`

Prints a formatted table to stdout and returns the same `Array<Hash>`.

```ruby
lister.summary("phase-2", "dtm", grid_square: "NT")
lister.summary("outer-hebrides", "dsm", resolution: "50cm")
```

---

### `SmoScottishLidar::Downloader`

Downloads files from the S3 bucket. Streams in chunks to avoid loading large files into memory.

```ruby
downloader = SmoScottishLidar::Downloader.new(verbose: false)
```

#### `downloader.download(phase, type, destination:, ...)`

Downloads all tiles matching the given filters. Returns a summary hash `{ downloaded: [...], skipped: [...], failed: [...] }`.

```ruby
downloader.download(
  "phase-1", "dsm",
  destination:   "/tmp/lidar/phase-1/dsm",
  grid_square:   "NS",          # optional. nil downloads everything
  skip_existing: true,          # skip files already on disk at the correct size
  dry_run:       false          # set true to preview without downloading
)
```

| Option | Default | Description |
|---|---|---|
| `destination:` | required | Local directory to save files into |
| `grid_square:` | `nil` | OS grid square filter, e.g. `"NS"`, `"NT"` |
| `resolution:` | `nil` | Outer Hebrides only, e.g. `"25cm"`, `"50cm"`, `"4ppm"`, `"16ppm"` |
| `skip_existing:` | `true` | Skip files that already exist locally at the correct size |
| `dry_run:` | `false` | Print what would be downloaded without downloading |

#### `downloader.download_file(phase, type, filename, destination:, resolution: nil)`

Downloads a single tile by exact filename.

```ruby
downloader.download_file(
  "phase-1", "dsm", "NS56_1M_DSM_PHASE1.tif",
  destination: "/tmp/lidar"
)
```

---

### `SmoScottishLidar.prefix_for(phase, type, resolution: nil)`

Returns the S3 prefix string for a given phase and type. Useful if you need to build custom queries.

```ruby
SmoScottishLidar.prefix_for("phase-1", "dsm")
# => "lidar/phase-1/dsm/27700/gridded/"

SmoScottishLidar.prefix_for("outer-hebrides", "dtm", resolution: "50cm")
# => "lidar/outer-hebrides/2019/dtm/50cm/27700/gridded/"
```

---

## Valid phases and types

```ruby
SmoScottishLidar::PHASES
# => ["phase-1", "phase-2", "phase-3", "phase-4", "phase-5", "outer-hebrides"]

SmoScottishLidar::DATASET_TYPES
# => ["dsm", "dtm", "laz"]
```

### Outer Hebrides resolutions

| Type | Available resolutions |
|---|---|
| dsm | `"25cm"` (default), `"50cm"` |
| dtm | `"25cm"` (default), `"50cm"` |
| laz | `"4ppm"` (default), `"16ppm"` |

---

## Examples

The `examples/` directory contains ready-to-run scripts:

| Script | Description |
|---|---|
| `demo.rb` | Full walkthrough of all features |
| `phase_1_dsm.rb` | List Phase 1 DSM tiles |
| `phase_1_dtm.rb` | List Phase 1 DTM tiles |
| `phase_1_laz.rb` | List Phase 1 LAZ tiles |
| `phase_2_dsm.rb` ... | One script per phase and type |
| `outer_hebrides_dsm.rb` | Outer Hebrides DSM |
| `outer_hebrides_dtm.rb` | Outer Hebrides DTM |
| `outer_hebrides_laz.rb` | Outer Hebrides LAZ |
| `download_individual_tile.rb` | Download a single named tile |
| `download_batch_tiles.rb` | Batch download with grid square filter |

Run any script with:

```sh
ruby examples/phase_1_dsm.rb
```

---

## Typical workflow for hydraulic modelling

```ruby
require "smo_scottish_lidar"

downloader = SmoScottishLidar::Downloader.new(verbose: true)

# 1. Check what is available for your catchment (e.g. NS and NS grid squares)
lister = SmoScottishLidar::Lister.new
lister.summary("phase-1", "dtm", grid_square: "NS")

# 2. Dry run first to confirm file sizes and count
downloader.download("phase-1", "dtm",
  destination: "/projects/my_catchment/lidar/dtm",
  grid_square: "NS",
  dry_run: true
)

# 3. Download for real
downloader.download("phase-1", "dtm",
  destination: "/projects/my_catchment/lidar/dtm",
  grid_square: "NS",
  dry_run: false
)

# 4. If the download is interrupted, re-run the same command.
# Files already on disk at the correct size are skipped automatically.
```

---

## Support

If this gem saves you time on a project, you can buy me a coffee.

[![Buy Me a Coffee](https://github.com/Sebasmadridmx/SMO-WGS84-TO-BNG/blob/main/temp_png/buymecoffeeqr.png)](https://buymeacoffee.com/smadrid)

---

## License

MIT. Copyright (c) 2024 Sebastian Madrid Ontiveros.
