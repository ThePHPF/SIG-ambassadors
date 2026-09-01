# Lists of possible sources for raw data extraction

> This is a first draft and should not be considered an exhaustive list.

## Scope

This document aims to list the possible data sources and the ways to access them and the extractions that can be made for the various sources.

## 1 Tools/Sources

* Google BigQuery: [Google BigQuery](https://cloud.google.com/bigquery)

### 1.1 Google BigQuery

Google BigQuery is usage-based: you pay only for the volume of data your queries actually read, and the first terabyte per month is free, which is enough for exploratory work and for analyses of moderate size.

## 2 Extractions

* `bigquery-public-data.github_repos`: Contents, commit history and metadata for
  millions of public GitHub repositories, including full source files for a
  licensed subset. Used to measure the presence and usage patterns of a library
  or API across real-world codebases. Note that the snapshot covers only
  open-licensed repositories and is not a complete mirror of GitHub.

* `bigquery-public-data.stackoverflow`: Full archive of Stack Overflow questions,
  answers, comments, tags and user activity, with timestamps. Used as a proxy for
  developer interest and for the practical problems encountered when adopting a
  technology. Loaded as a periodic dump rather than continuously, and the copy on
  BigQuery has not been refreshed since late 2022, so recent years are missing.

* `httparchive.crawl`: The monthly HTTP Archive crawl, which loads millions of
  sites in both desktop and mobile configurations and records what each one
  serves. Two tables: `pages`, one row per page tested, carrying detected
  technologies, Lighthouse results and a page-level summary; and `requests`,
  one row per resource loaded. Used to observe what is actually deployed on the
  public web and how it performs. Both tables are partitioned by `date` and run
  to tens or hundreds of terabytes per crawl, so queries must filter on the
  partition and select only the fields needed.

### 2.1 `bigquery-public-data.github_repos` delivered by [Google BigQuery](https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=github_repos&t=files&page=table)

**data freshness: 2025-10-14 05:11:35.999 UTC, last check 2026-09-01 16:00:00.000 UTC**

A snapshot of the open source contents of GitHub, loaded into BigQuery and queryable in SQL. It is not a complete mirror of GitHub: only repositories that GitHub classifies as open source through its License API are included, and for file contents only text files under 10 MB.

It is suited to questions such as "how many projects import this library", "how is this API actually used in real code", "what is the licence distribution by language".

### Table structure

| Table | Granularity | Main fields |
|---|---|---|
| `files` | one row per file path | `repo_name`, `ref`, `path`, `mode`, `id`, `symlink_target` |
| `contents` | one row per unique content | `id`, `size`, `content`, `binary`, `copies` |
| `commits` | one row per commit | `commit`, `tree`, `parent`, `author`, `committer`, `subject`, `message`, `difference`, `repo_name` |
| `languages` | one row per repository | `repo_name`, `language` (array of `{name, bytes}`) |
| `licenses` | one row per repository | `repo_name`, `license` |
| `sample_*` | reduced subsets of the tables above | same schemas |

> [!important]
> the query reported here needs to be reasoned and refined

```sql
WITH
-- All repos GitHub Linguist tags as containing PHP.
php_repos AS (
  SELECT DISTINCT repo_name
  FROM `bigquery-public-data.github_repos.languages`, UNNEST(language) AS lang
  WHERE lang.name = 'PHP'
),

-- Single scan of the files table: keep only Composer-related paths.
-- vendor/autoload.php is the reliable marker of a committed vendor directory.
composer_files AS (
  SELECT
    repo_name,
    path,
    (path = 'composer.json' OR ENDS_WITH(path, '/composer.json')) AS is_manifest,
    (STARTS_WITH(path, 'vendor/') OR CONTAINS_SUBSTR(path, '/vendor/')) AS is_in_vendor
  FROM `bigquery-public-data.github_repos.files`
  WHERE path = 'composer.json'
     OR ENDS_WITH(path, '/composer.json')
     OR ENDS_WITH(path, 'vendor/autoload.php')
),

-- Collapse to one row per repo with the three signals we care about.
repo_signals AS (
  SELECT
    repo_name,
    LOGICAL_OR(path = 'composer.json')             AS has_root_manifest,
    LOGICAL_OR(is_manifest AND NOT is_in_vendor)   AS has_own_manifest,
    LOGICAL_OR(is_in_vendor)                       AS has_vendor_committed
  FROM composer_files
  GROUP BY repo_name
),

-- Left join so repos with no Composer footprint at all survive as NULLs,
-- then normalise those NULLs to FALSE so every repo lands on one side.
classified AS (
  SELECT
    p.repo_name,
    IFNULL(s.has_root_manifest,    FALSE) AS has_root_manifest,
    IFNULL(s.has_own_manifest,     FALSE) AS has_own_manifest,
    IFNULL(s.has_vendor_committed, FALSE) AS has_vendor_committed
  FROM php_repos AS p
  LEFT JOIN repo_signals AS s ON s.repo_name = p.repo_name
)

SELECT
  COUNT(*) AS total_php_repos,

  -- Bottom line: does the project ship a manifest of its own?
  COUNTIF(has_own_manifest)       AS uses_composer,
  COUNTIF(NOT has_own_manifest)   AS no_composer,

  -- Manifest present but not at the repo root: likely a layout problem,
  -- or a legitimate monorepo. Cannot be told apart from paths alone.
  COUNTIF(has_own_manifest AND NOT has_root_manifest) AS manifest_not_at_root,

  -- The classic mistake: vendor/ committed to version control.
  COUNTIF(has_vendor_committed)                          AS vendor_committed,
  COUNTIF(has_vendor_committed AND NOT has_own_manifest) AS vendor_committed_without_manifest,

  ROUND(100 * COUNTIF(has_own_manifest) / COUNT(*), 2)                          AS pct_uses_composer,
  ROUND(100 * COUNTIF(has_vendor_committed) / NULLIF(COUNTIF(has_own_manifest), 0), 2)
                                                                                AS pct_vendor_among_users
FROM classified
```

Assumed metrics

| Metric | Count | Share | What it means |
|---|---:|---:|---|
| PHP repositories (total) | 339,426 | 100% | Every repo Linguist tags as containing PHP, including those where PHP is a minor part of the codebase |
| Ships a `composer.json` | 159,237 | 46.91% of total | Declares its own manifest somewhere outside `vendor/`, the project participates in the Composer ecosystem |
| No `composer.json` | 180,189 | 53.09% of total | No manifest of its own anywhere in the tree |
| Manifest at repository root | 142,299 | 89.36% of Composer users | The conventional layout: `composer.json` sits next to the README |
| Manifest outside the root | 16,938 | 10.64% of Composer users | Manifest lives in a subdirectory. Deliberate in monorepos, accidental in projects whose PHP code is buried under `src/`, `api/` or `web/` |
| `vendor/` committed to Git | 17,443 | 10.95% of Composer users | Dependencies checked into version control instead of being installed from the lock file |
| `vendor/` committed, no manifest | 2,067 | 1.15% of repos without a manifest | Dependencies committed without the file that describes them, these repos use the ecosystem but are invisible to any manifest-based search |

### 2.2 `bigquery-public-data.stackoverflow` delivered by [Google BigQuery](https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=stackoverflow&t=posts_questions&page=table)

**data freshness: 2022-11-25 01:03:23.684 UTC, last check 2026-09-01 16:00:00.000 UTC**

A full copy of the public Stack Overflow data dump loaded into BigQuery: questions, answers, comments, users, votes, tags and edit history, with original timestamps. It covers the site from 2008 to Q3 2022.

It suits measuring developer interest in a technology over time, identifying recurring problems during adoption, and gauging the health of an ecosystem through the volume and quality of discussion.

Compared with `github_repos` this is a **small** dataset, tens of gigabytes rather than terabytes, and therefore far cheaper to query.

### Table structure

| Table | Granularity | Main fields |
|---|---|---|
| `posts_questions` | one row per question | `id`, `title`, `body`, `tags`, `score`, `view_count`, `answer_count`, `accepted_answer_id`, `creation_date`, `owner_user_id` |
| `posts_answers` | one row per answer | `id`, `parent_id`, `body`, `score`, `creation_date`, `owner_user_id` |
| `stackoverflow_posts` | all post types in one table | as above, plus `post_type_id` |
| `comments` | one row per comment | `id`, `post_id`, `text`, `score`, `creation_date`, `user_id` |
| `users` | one row per user | `id`, `display_name`, `reputation`, `location`, `about_me`, `creation_date`, `last_access_date`, `up_votes`, `down_votes` |
| `votes` | one row per vote | `id`, `post_id`, `vote_type_id`, `creation_date` |
| `badges` | one row per badge awarded | `id`, `name`, `user_id`, `date`, `class`, `tag_based` |
| `tags` | one row per tag | `id`, `tag_name`, `count`, `excerpt_post_id`, `wiki_post_id` |
| `post_history` | one row per revision | `id`, `post_id`, `post_history_type_id`, `creation_date`, `text` |
| `post_links` | one row per link between posts | `id`, `post_id`, `related_post_id`, `link_type_id` |

> [!important]
> the query reported here needs to be reasoned and refined

```sql
WITH totals AS (
  -- Denominator: all questions. Normalising matters, site-wide volume
  -- shifted a lot over the years, so absolute counts mislead.
  SELECT DATE_TRUNC(DATE(creation_date), QUARTER) AS quarter,
         COUNT(*) AS total_questions
  FROM `bigquery-public-data.stackoverflow.posts_questions`
  GROUP BY quarter  -- DATE() needed: creation_date is a TIMESTAMP
),

php AS (
  -- Numerator: questions tagged 'php'.
  SELECT DATE_TRUNC(DATE(q.creation_date), QUARTER) AS quarter,
         COUNT(*) AS php_questions
  FROM `bigquery-public-data.stackoverflow.posts_questions` AS q,
       -- tags is a '|'-joined string, not an ARRAY
       UNNEST(SPLIT(q.tags, '|')) AS tag
  WHERE tag = 'php'  -- exact match: LIKE '%php%' would catch 'phpunit'
  GROUP BY quarter
)

SELECT
  t.quarter,
  p.php_questions,
  t.total_questions,
  ROUND(100 * p.php_questions / t.total_questions, 2) AS share_pct
FROM totals AS t
JOIN php AS p USING (quarter)  -- INNER: fine for a common tag, not a rare one
WHERE t.quarter >= DATE '2012-01-01'  -- early years are noisy
ORDER BY t.quarter;
```

Assumed metrics

| Quarter | PHP questions | Total | Share % |
|-------------|--------------:|-------------:|---:|
| 2012 Q1     |        30,153 |      376,853 | 8.00 |
| 2012 Q2     |        31,925 |      397,370 | 8.03 |
| 2012 Q3     |        34,599 |      418,704 | 8.26 |
| 2012 Q4     |        34,579 |      436,459 | 7.92 |
| 2013 Q1     |        39,453 |      487,682 | 8.09 |
| 2013 Q2     |        39,119 |      499,706 | 7.83 |
| 2013 Q3     |        42,801 |      516,364 | 8.29 |
| 2013 Q4     |        44,575 |      529,938 | 8.41 |
| **2014 Q1** |    **51,559** |  **588,822** | **8.76** |
| 2014 Q2     |        45,358 |      537,546 | 8.44 |
| 2014 Q3     |        40,840 |      511,556 | 7.98 |
| 2014 Q4     |        38,960 |      499,511 | 7.80 |
| 2015 Q1     |        42,045 |      528,559 | 7.95 |
| 2015 Q2     |        43,935 |      568,055 | 7.73 |
| 2015 Q3     |        43,152 |      558,809 | 7.72 |
| 2015 Q4     |        41,014 |      541,253 | 7.58 |
| 2016 Q1     |        44,647 |      575,649 | 7.76 |
| 2016 Q2     |        42,547 |      574,969 | 7.40 |
| 2016 Q3     |        37,927 |      533,886 | 7.10 |
| 2016 Q4     |        35,675 |      516,298 | 6.91 |
| 2017 Q1     |        39,153 |      557,366 | 7.02 |
| 2017 Q2     |        37,172 |      547,215 | 6.79 |
| 2017 Q3     |        34,081 |      523,400 | 6.51 |
| 2017 Q4     |        30,524 |      488,231 | 6.25 |
| 2018 Q1     |        29,043 |      490,342 | 5.92 |
| 2018 Q2     |        26,806 |      488,334 | 5.49 |
| 2018 Q3     |        24,653 |      465,469 | 5.30 |
| 2018 Q4     |        21,415 |      444,844 | 4.81 |
| 2019 Q1     |        22,210 |      459,576 | 4.83 |
| 2019 Q2     |        20,086 |      443,263 | 4.53 |
| 2019 Q3     |        18,013 |      427,610 | 4.21 |
| 2019 Q4     |        17,344 |      436,484 | 3.97 |
| 2020 Q1     |        17,224 |      451,332 | 3.82 |
| 2020 Q2     |        19,091 |      545,781 | 3.50 |
| 2020 Q3     |        15,714 |      459,909 | 3.42 |
| 2020 Q4     |        14,020 |      414,673 | 3.38 |
| 2021 Q1     |        13,902 |      424,970 | 3.27 |
| 2021 Q2     |        12,678 |      403,943 | 3.14 |
| 2021 Q3     |        11,620 |      376,487 | 3.09 |
| 2021 Q4     |        11,943 |      424,180 | 2.82 |
| 2022 Q1     |        11,412 |      436,766 | 2.61 |
| 2022 Q2     |        11,673 |      427,400 | 2.73 |
| 2022 Q3     |        11,261 |      404,622 | 2.78 |

Yearly assumed metrics

| Year      | PHP questions |         Total | Share % |
|-----------|---------------:|--------------:|---:|
| 2012      |        131,256 |     1,629,386 | 8.06 |
| 2013      |        165,948 |     2,033,690 | 8.16 |
| **2014**  |    **176,717** | **2,137,435** | **8.27** |
| 2015      |        170,146 |     2,196,676 | 7.75 |
| 2016      |        160,796 |     2,200,802 | 7.31 |
| 2017      |        140,930 |     2,116,212 | 6.66 |
| 2018      |        101,917 |     1,888,989 | 5.39 |
| 2019      |         77,653 |     1,766,933 | 4.39 |
| 2020      |         66,049 |     1,871,695 | 3.53 |
| 2021      |         50,143 |     1,629,580 | 3.08 |
| 2022*     |         34,346 |     1,268,788 | 2.71 |

\* Partial year: first three quarters only.

### 2.3 `httparchive.crawl` delivered by [Google BigQuery](https://console.cloud.google.com/bigquery?p=httparchive&d=crawl&t=pages&page=table)

**data freshness: 2026-08-17 23:14:05.000 UTC, last check 2026-09-01 16:00:00.000 UTC**

The main HTTP Archive dataset, from the project that loads millions of sites with WebPageTest every month and archives their performance, detected technologies, structure and resources. It is the data behind the Web Almanac and the standard source for analyses of the state of the web.

The `crawl` dataset is the current structure: it replaces the old dated tables (`httparchive.pages.2023_06_01_desktop` and similar) and the `runs`, `har` and `all` datasets. Pre-2024 queries found online no longer work as written.

**Note the project**: this is not in `bigquery-public-data` but in the separate `httparchive` project. It is still part of the Public Dataset Program, so storage is free and only queries are billed.

### The two main tables

| Table | Granularity | Monthly size | History since |
|---|---|---|---|
| `crawl.pages` | one row per page tested | ~30 TB | June 2011 |
| `crawl.requests` | one row per resource loaded | ~199 TB | June 2011 |

Both are partitioned on `date`. `pages` is clustered on `client`, `is_root_page`, `rank`, `page`; `requests` on `client`, `is_root_page`, `type`, `rank`.

Each page is tested in two configurations, desktop and mobile, and since April 2022 both the origin's root page and one secondary page are tested.

### `crawl.pages` schema

| Field | Type | Contents |
|---|---|---|
| `date` | DATE | Monthly crawl date, always the first of the month |
| `client` | STRING | `desktop` or `mobile` |
| `page` | STRING | URL of the page tested |
| `is_root_page` | BOOLEAN | Whether the page is the root of the origin |
| `root_page` | STRING | URL of the root page, the origin followed by `/` |
| `rank` | INTEGER | Site popularity bucket, from CrUX |
| `wptid` | STRING | Identifier of the WebPageTest results |
| `payload` | JSON | Full WebPageTest results |
| `summary` | JSON | Summarisation of page-level metrics |
| `technologies` | repeated RECORD | Technologies detected by Wappalyzer |
| `custom_metrics` | RECORD | Custom metrics collected during the test |
| `lighthouse` | JSON | Lighthouse report |
| `features` | repeated RECORD | Blink features used by the page |
| `metadata` | JSON | Additional metadata about the test |

> [!important]
> the query reported here needs to be reasoned and refined

```sql
WITH sites AS (
  SELECT
    date,
    root_page,
    -- EXISTS avoids UNNEST in the FROM clause, which would emit one row
    -- per technology and break the site count.
    EXISTS(
      SELECT 1 FROM UNNEST(technologies) AS t
      WHERE t.technology = 'PHP'
    ) AS uses_php
  FROM `httparchive.crawl.pages`
  -- Each extra date multiplies the cost: date is the partitioning column.
  WHERE date IN ('2023-06-01', '2024-06-01', '2025-06-01')
    AND is_root_page       -- without this every origin counts twice
    AND client = 'mobile'  -- one config is enough for a share; both would double-count
)

SELECT
  date,
  COUNT(*) AS total_sites,
  COUNTIF(uses_php) AS php_sites,
  ROUND(100 * COUNTIF(uses_php) / COUNT(*), 2) AS share_pct
FROM sites
GROUP BY date
ORDER BY date;
```

Assumed metrics

| Crawl | Total sites | Sites with PHP | Share % |
|---|---:|---:|---:|
| 2023-06 | 16,563,413 | 9,099,650 | 54.94 |
| 2024-06 | 16,129,455 | 8,773,007 | 54.39 |
| 2025-06 | 15,545,137 | 8,208,463 | 52.80 |
