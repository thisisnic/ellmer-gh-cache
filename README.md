# ellmer-gh-cache

Parquet cache of `tidyverse/ellmer` GitHub activity. Published as assets on the
rolling [`cache-latest`](../../releases/tag/cache-latest) release — the latest
snapshot only, no history.

## Usage

Each table has a stable public URL. No clone, no auth, no LFS required.

```r
library(arrow)
library(dplyr)

base_url <- "https://github.com/thisisnic/ellmer-gh-cache/releases/download/cache-latest"

open_prs      <- read_parquet(file.path(base_url, "open_prs.parquet"))
closed_prs    <- read_parquet(file.path(base_url, "closed_prs.parquet"))
open_issues   <- read_parquet(file.path(base_url, "open_issues.parquet"))
closed_issues <- read_parquet(file.path(base_url, "closed_issues.parquet"))

# Example: PRs merged in the last 30 days
closed_prs |>
  filter(!is.na(merged_at), merged_at >= Sys.time() - as.difftime(30, units = "days"))

# Example: open issues by label
open_issues |>
  tidyr::unnest(labels) |>
  count(labels, sort = TRUE)
```

## Available tables

| Asset | Contents | Detail level |
|---|---|---|
| `open_prs.parquet` | Open pull requests | Full (includes mergeable, additions/deletions, etc. from per-PR detail endpoint) |
| `closed_prs.parquet` | Closed pull requests | List-level (no per-PR detail fetch; includes merged_at and merge_commit_sha) |
| `open_issues.parquet` | Open issues (PRs excluded) | List-level |
| `closed_issues.parquet` | Closed issues (PRs excluded) | List-level |
| `issue_comments.parquet` | Comments on issues and PRs (conversation tab) | Full |
| `review_comments.parquet` | PR review comments (inline on diffs) | Full |

### Column differences

Open PRs include extra fields from the per-PR detail endpoint that are not
available for closed PRs: `mergeable`, `mergeable_state`, `rebaseable`,
`additions`, `deletions`, `changed_files`, `commits`, `review_comments_count`,
`merged`, `merged_by`.

## How it works

A scheduled GitHub Action (`.github/workflows/update_cache.yaml`) runs every 3 hours,
calls the GitHub REST API, writes parquet files to `artifacts/`, and uploads
them to the `cache-latest` release with `gh release upload --clobber`.

On each run, existing assets are downloaded first. Missing tables are
backfilled from scratch; existing tables get a delta update using the
`since` parameter, so only items changed since the last run are re-fetched.
