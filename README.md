# YouTube Trending Data — AWS ETL Pipeline

Serverless data lake on AWS that cleans and analyses the Kaggle
[Trending YouTube Video Statistics](https://www.kaggle.com/datasets/datasnaek/youtube-new) dataset
(daily trending videos for 10 regions + per-region category JSON).

**Stack:** S3 · Lambda · Glue (Crawler, Catalog, Spark ETL) · Athena · QuickSight · IAM · Terraform

## Architecture

```
 Local CSV / JSON
       │  aws s3 cp
       ▼
┌──────────── RAW bucket ─────────────────────────────────────────┐
│ youtube/raw_statistics/region=ca|de|us|...   (CSV, partitioned)  │
│ youtube/raw_statistics_reference_data/       (category JSON)     │
└───────┬───────────────────────────────┬─────────────────────────┘
        │ S3 ObjectCreated event         │ Glue Crawler
        ▼                                ▼
 Lambda (JSON → Parquet)         Glue Catalog: db_youtube_raw
        │                                │
        │                                ▼
        │                        Glue Spark ETL job
        │                        (map types, drop nulls, Parquet)
        ▼                                ▼
┌──────────── CLEAN bucket ───────────────────────────────────────┐
│ cleansed_statistics_reference_data/   raw_statistics/region=...  │
└───────┬─────────────────────────────────────────────────────────┘
        │ Glue Crawler → db_youtube_cleansed
        ▼
   Athena (join stats + categories)  →  QuickSight dashboard
```

## Flow

1. **Ingest** — CSVs are copied to `raw_statistics/region=xx/` (Hive-style partitions); category JSONs go to `raw_statistics_reference_data/` ([upload commands](Project_Code/Project_Code/Code/video8%20Amazon%20S3%20CLI%20copy%20commands.sh)).
2. **Reference data** — the JSON upload triggers a [Lambda](Project_Code/Project_Code/Code/video17%20lambda%20function.py) that flattens `items` with `awswrangler` and writes Parquet + a Glue table to the clean bucket.
3. **Statistics data** — a Glue crawler catalogs the raw CSVs; a [Glue Spark job](Project_Code/Project_Code/Code/video22%20Spark%20code%20Glue%20job.py) applies the schema, drops nulls and writes region-partitioned Parquet to the clean bucket. The [pushdown-predicate version](Project_Code/Project_Code/Code/video24%20Spark%20code%20Glue%20job%20with%20pushdown%20predicate.py) reads only selected regions (`ca`, `gb`, `us`).
4. **Analyse** — Athena joins the cleansed tables on `category_id`; QuickSight visualises the result.

## Repo layout

| Path | Contents |
|------|----------|
| `Data/` | Kaggle CSV + category JSON files (10 regions) |
| `Project_Code/Project_Code/Code/` | Lambda, Glue job scripts, S3 CLI commands |
| `Project_Code/Project_Code/Terraform/` | IaC for buckets, Glue DBs/crawlers/job, Lambda, IAM, S3 trigger |
| `Solution Methodology.pdf` | Project write-up |

## Deploy

```bash
cd Project_Code/Project_Code/Terraform
aws configure          # use your own credentials
terraform init
terraform apply
```

Then manually: attach the **AWS Wrangler** layer to the Lambda, run the crawlers, run the Glue job, and create the QuickSight dashboard (not in Terraform). See [Instructions.md](Project_Code/Project_Code/Terraform/Instructions.md).

## Known issues to fix before running

- Lambda reads lowercase env vars (`s3_cleansed_layer`) but Terraform sets uppercase (`S3_CLEANSED_LAYER`).
- Glue scripts hardcode the output bucket `bigdata-on-youtube-cleansed-euwest1-14317621-dev`; Terraform creates `s3-bucket-youtube-clean-data`.
- Glue job resource is named `flight-performance-processing-job` (copy-paste leftover).
- Lambda runtime `python3.8` is deprecated.
- IAM uses broad policies (`AmazonS3FullAccess`, `glue:*`/`s3:*` on `*`) — tighten for production.
