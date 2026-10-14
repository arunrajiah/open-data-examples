# WildNetwork open wildlife detections (WDX) on AWS

Open snapshots of [WildNetwork](https://wildnetwork.arunrajiah.com), a free, open live map of wildlife detections from community acoustic stations, AI camera traps and citizen science. Every record is a [WDX](https://github.com/arunrajiah/wildlife-detection-exchange) (Wildlife Detection Exchange) event: one machine or community detection with the species, time, place, the AI model and version that made it, its confidence score, the station's listening effort, and a human review status.

> **Status: draft, in onboarding with the AWS Open Data Sponsorship Program (October 2026).** The bucket name, region and file layout below are the planned design and may change before launch. Until the bucket is live, the same data can be read from the WildNetwork public API (see "Access without AWS" below).

- **Planned location:** `s3://wildnetwork-open-data/` in `us-east-1` (public, no AWS account or credentials needed)
- **Format:** Apache Parquet (GeoParquet for point data), partitioned by date; the WDX JSON Schema alongside
- **Update frequency:** weekly snapshot (every Monday, UTC), plus a full re-export when the WDX schema version changes
- **Scale today:** about 18,800 sensors, 311 million detections and 44,000 species represented on WildNetwork (live figures at `https://wildnetwork.arunrajiah.com/api/v1/status`)
- **Maintainer:** Arun Rajiah, Chennai, India. Contact: arunrajiah@gmail.com or [GitHub issues](https://github.com/arunrajiah/wildnetwork/issues)

## Why this dataset exists

Tens of thousands of low-cost acoustic stations (BirdNET-Pi, BirdNET-Go, BirdWeather) and AI camera traps now run continuously in backyards, farms and reserves. Their detections rarely leave the device, and when they do, each system writes a different record with the model, confidence and listening effort often lost. WildNetwork normalises them into one open format so researchers can study species presence, migration timing and range shifts at a scale and time resolution that point surveys cannot reach, while keeping the information needed to judge each record's reliability.

## Bucket layout

```
s3://wildnetwork-open-data/
├── README.md                          this document
├── LICENSE.md                         licence terms per source (see below)
├── CHANGELOG.md                       snapshot history and schema changes
├── manifest.json                      latest snapshot date, row counts, file list, checksums
├── schema/
│   └── wdx-0.2.schema.json            WDX JSON Schema (also on Zenodo, DOI 10.5281/zenodo.22200864)
├── events/                            individual detection events (GeoParquet), CC0 and CC BY only
│   └── year=2026/month=10/day=12/
│       └── part-000.parquet
├── aggregates/                        CC0 and CC BY only
│   ├── species_daily/                 one row per species, 5-degree cell, day and source
│   │   └── year=2026/part-000.parquet
│   └── species_weekly/                one row per species, 5-degree cell, ISO week and source
│       └── year=2026/part-000.parquet
└── noncommercial/                     CC BY-NC records only, same layout and columns
    ├── events/year=YYYY/month=MM/day=DD/
    └── aggregates/species_daily/ and species_weekly/
```

**Open and non-commercial data are kept apart.** Everything under `events/` and `aggregates/` is CC0 or CC BY and can be used for any purpose, including commercial use. Records whose observers chose CC BY-NC (mostly from iNaturalist) are published only under `noncommercial/`, with the same layout and columns, so you can include or exclude them with a single path.

Partitions use Hive-style `key=value` folder names, so DuckDB, Polars, PyArrow, Spark and Amazon Athena can prune by date automatically.

## Data dictionary

### `events/` (one row per detection event)

| Column | Type | Description |
|---|---|---|
| `event_id` | string | Globally unique, stable identifier of the detection (prefixed by source, e.g. `inaturalist:123`) |
| `wdx_version` | string | WDX version the record conforms to (`0.1` or `0.2`) |
| `event_start` / `event_end` | timestamp (UTC) | Start and end of the detection window |
| `scientific_name` | string | Accepted scientific name |
| `vernacular_name` | string | Common name in English, when known |
| `taxon_rank` | string | `species`, `genus`, `family`, `class` or `unranked` |
| `taxon_id` | string | Checklist-prefixed identifier, e.g. `gbif:2493091` |
| `taxon_group` | string | `avian`, `bat`, `amphibian`, `insect`, `mammal` or `other` |
| `confidence` | double | Classifier score from 0 to 1. **Not a probability**: scores are not calibrated across species or models |
| `classifier_name` / `classifier_version` | string | Model that produced the label, e.g. `BirdNET` `2.4`, `Perch` `2.0`, `SpeciesNet` `4.0` |
| `review_status` | string | `unreviewed`, `confirmed`, `rejected` or `uncertain` |
| `reviewed_by` / `reviewed_at` | string / timestamp | Who reviewed the record and when (institution or pseudonymous reviewer id) |
| `deployment_id` | string | Station identifier (stable, not personally identifying) |
| `sensor_type` | string | `acoustic-recorder`, `camera-trap` or `other` |
| `sensor_model` | string | e.g. `BirdNET-Pi`, `WildNetwork Base Pi`, `AudioMoth` |
| `geometry` | point (WKB, EPSG:4326) | Station location, **generalised**: private stations rounded to about 1 km; sensitive species generalised further |
| `coordinate_uncertainty_m` | double | Radius covering the generalisation and GPS error |
| `source_system` | string | `wdx` (contributed devices), `inaturalist`, `gbif` |
| `source_record_id` | string | Identifier in the source system |
| `license` | string | SPDX licence of the record: `CC-BY-4.0` or `CC0-1.0` (`CC-BY-NC-4.0` only under `noncommercial/`) |

### `aggregates/species_daily/` and `aggregates/species_weekly/`

| Column | Type | Description |
|---|---|---|
| `day` / `week` | date | Day, or ISO week starting Monday |
| `source_system` | string | Source of the counts |
| `scientific_name` / `vernacular_name` | string | Species |
| `cell_lat` / `cell_lon` | int16 | South-west corner of the 5-degree grid cell |
| `count` | int32 | Detections of the species in the cell and period |
| `high_conf_count` | int32 | Detections with confidence 0.85 or higher (or the source's equivalent "very likely" class) |
| `sites` | int32 | Distinct stations contributing (daily only; null when unknown) |

Aggregates are the right input for movement and timing analyses: WildNetwork's published indices use a species' **share** of all detections in a cell, never raw counts, so that more stations do not look like more animals. Methods: <https://wildnetwork.arunrajiah.com/methods>.

## Licensing and sources

Only records whose licence allows redistribution are published, and each row carries its own `license`. CC0 and CC BY records are under `events/` and `aggregates/`; CC BY-NC records are only under `noncommercial/`.

| Source | What is included | Licence |
|---|---|---|
| WDX contributions (WildNetwork Base, BirdNET-Pi, BirdNET-Go and other stations sending through wdx-agent) | events and aggregates | CC BY 4.0 by default, as chosen by each contributor |
| iNaturalist (via its API) and GBIF | events and aggregates | CC0 or CC BY in the main prefixes; CC BY-NC records only under `noncommercial/`. Credit observers as their licences require |
| BirdWeather community stations | aggregates only, and only with BirdWeather's written permission for redistribution | to be confirmed before launch |

Please cite the dataset (see below) and credit the underlying sources when you publish.

## Data quality: read this before analysis

- **Most records are unreviewed machine output.** Check `review_status`. Records marked `confirmed` have been checked by a person; `unreviewed` records are raw classifier output.
- **Confidence scores are not probabilities.** Choose thresholds per species and per study goal, ideally from a reviewed sample (Wood and Kahl 2024, *J. Ornithol.*; Pérez-Granados et al. 2025, *Ibis*).
- **Effort varies.** Stations go on and off line; use the per-cell share of detections, or the `sites` column, rather than raw counts.
- **Locations are generalised** to protect station owners' homes and sensitive species.

## Access

With no AWS account (anonymous):

```bash
aws s3 ls --no-sign-request s3://wildnetwork-open-data/
```

DuckDB, straight from S3:

```sql
INSTALL httpfs; LOAD httpfs;
SET s3_region='us-east-1';
SELECT scientific_name, sum(count) AS detections
FROM read_parquet('s3://wildnetwork-open-data/aggregates/species_weekly/*/*.parquet', hive_partitioning=true)
WHERE cell_lat = 10 AND cell_lon = 75          -- 10-15 N, 75-80 E: Kerala, central Tamil Nadu, southern Karnataka
GROUP BY 1 ORDER BY 2 DESC LIMIT 20;
```

A guided tour is in the notebook [get-to-know-a-dataset.ipynb](get-to-know-a-dataset.ipynb).

### Access without AWS

The same data is served by the WildNetwork public API (no key): `https://wildnetwork.arunrajiah.com/api/v1/` with `events`, `species`, `arrivals` (CSV available), `insights` and `status`. See <https://wildnetwork.arunrajiah.com/developers>.

## Citation

> Rajiah, A. (2026). WildNetwork open wildlife detections (WDX). Accessed [date] from https://registry.opendata.aws/wildnetwork-wdx. WDX format: doi:10.5281/zenodo.22200864.

## Related

- WDX format and JSON Schema: <https://github.com/arunrajiah/wildlife-detection-exchange>
- WildNetwork platform code (AGPL-3.0): <https://github.com/arunrajiah/wildnetwork>
- wdx-agent, for sending your own station's detections (Apache-2.0): <https://github.com/arunrajiah/wdx-agent>
