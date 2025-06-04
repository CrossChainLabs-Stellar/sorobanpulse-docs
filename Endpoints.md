## Endpoints definition

Below are the three endpoint definitions with detailed inline comments to explain each step of the logic, the purpose of functions, and how they ensure accuracy.

---

### 1. `commits_weekly`

```sql
-- Create a materialized view that aggregates the total number of commits per week across all repositories.
-- Using a materialized view allows us to precompute and store results, improving query performance for downstream analytics.

CREATE MATERIALIZED VIEW IF NOT EXISTS commits_weekly_view_v2 AS

WITH
    -- CTE: commits_data
    -- 1. Truncates each commit’s timestamp to the start of its week (Monday by default).
    -- 2. Computes an “epoch” value for that week to support easy numeric comparisons.
    -- 3. Counts how many commits occurred in each truncated week interval.
    commits_data AS (
        SELECT
            -- Truncate commit_date to the beginning of its week, then cast to DATE for readability.
            date_trunc('week', commit_date)::date AS commit_week,
            
            -- Extract the Unix epoch (seconds since 1970-01-01) from the truncated week timestamp.
            -- This numeric representation can be used for sorting or joining if needed.
            EXTRACT(EPOCH FROM date_trunc('week', commit_date)) AS week_epoch,
            
            -- Count all commit_hash values within each truncated week bucket.
            -- commit_hash is assumed to be unique per commit, so counting them yields total commits.
            count(commit_hash) AS commits
        FROM
            commits
        GROUP BY
            commit_week,    -- Group by the date representing the start of each week
            week_epoch      -- Also group by the numeric epoch for that same week
    )

-- Final SELECT: Return the week and commit totals, ordering by most recent week first.
SELECT
    commit_week,                                           -- The DATE corresponding to the Monday (or week start) of that week
    week_epoch,                                            -- Unix epoch of the same week for numeric sorting/comparison
    COALESCE(commits_data.commits, 0) AS total_commits     -- Use COALESCE to ensure zero if no commits, though grouping should eliminate NULLs
FROM
    commits_data
ORDER BY
    commit_week DESC                                       -- Sort descending so the newest weeks appear first
WITH DATA;                                                 -- Populate the materialized view immediately with this data
```

---

### 2. `new_developers_weekly`

```sql
-- Create a materialized view that counts how many developers made their first commit in each week.
-- This helps track growth in the number of active contributors.

CREATE MATERIALIZED VIEW IF NOT EXISTS new_devs_weekly_view AS

WITH
    -- CTE: dev_first_commit
    -- 1. Finds the minimum commit_date per developer across all repos.
    -- 2. Truncates that first commit date to the start of the week and computes its epoch.
    dev_first_commit AS (
        SELECT
            c.dev_id,
            
            -- Find the earliest commit_date for this developer across all repositories,
            -- then truncate to the beginning of that week and cast to DATE.
            DATE_TRUNC('week', MIN(c.commit_date))::date AS first_commit_week,
            
            -- Compute the Unix epoch for the week containing their first commit
            EXTRACT(EPOCH FROM DATE_TRUNC('week', MIN(c.commit_date))) AS week_epoch
        FROM
            commits c
        GROUP BY
            c.dev_id    -- Group by developer ID to get one record per developer
    ),

    -- CTE: weekly_new_dev
    -- 1. Counts how many distinct developers have their first_commit_week equal to each given week.
    weekly_new_dev AS (
        SELECT
            first_commit_week,                   -- The week-start date when these new developers first contributed
            week_epoch,                          -- Corresponding numeric epoch for that week
            COUNT(*) AS new_devs                 -- Total number of developers whose first commit was in that week
        FROM
            dev_first_commit
        GROUP BY
            first_commit_week,                   -- Group by week to count developers per week
            week_epoch                           -- Also group by epoch to retain numeric week value
    )

-- Final SELECT: Return week and count of new developers, sorted with newest weeks first.
SELECT
    first_commit_week AS commit_week,                 -- Rename to “commit_week” for consistency with other views
    week_epoch,                                        -- Numeric representation of that week
    new_devs                                           -- Number of developers whose first commit fell in that week
FROM
    weekly_new_dev
ORDER BY
    commit_week DESC                                   -- Show the most recent weeks first
WITH DATA;                                             -- Immediately materialize and store results
```

---

### 3. `cumulative_commits` and `cumulative_developer_count`

```sql
-- Create a materialized view that returns two overall metrics:
-- 1. cumulative_commits: the sum of all contributions (commit counts) across every developer and repo.
-- 2. cumulative_developer_count: the total number of distinct developers in our dataset.
-- We use a CROSS JOIN to combine these two aggregated values into a single-row result.

CREATE MATERIALIZED VIEW IF NOT EXISTS statistics_commits_developers AS

WITH
    -- CTE: total_developers_cte
    -- Counts the number of unique developer usernames in the devs table.
    total_developers_cte AS (
        SELECT
            COUNT(DISTINCT dev_name) AS cumulative_developer_count
            -- DISTINCT dev_name ensures each developer is counted only once, even if they have multiple repo entries.
        FROM
            devs
    ),

    -- CTE: commits_data
    -- Sums up all contributions from the devs_contributions table.
    -- Each row in devs_contributions represents how many commits (contributions) a given dev made to a specific repo.
    commits_data AS (
        SELECT
            SUM(contributions) AS cumulative_commits
            -- Summing “contributions” across all rows yields the total number of commits in the system.
        FROM
            devs_contributions
    )

-- Final SELECT: Combine both aggregated values into a single row.
SELECT
    commits_data.cumulative_commits,                     -- Total commit count across all developers/repos
    total_developers_cte.cumulative_developer_count      -- Total distinct developer count
FROM
    commits_data
CROSS JOIN
    total_developers_cte                                 -- CROSS JOIN guarantees a Cartesian product that yields one row
WITH DATA;                                               -- Immediately populate the materialized view
```

---

#### Notes on Commenting Style and Accuracy

* **Purpose of Materialized Views**: Each `WITH DATA` ensures that the view is populated immediately.
* **`date_trunc('week', …)`**: By default, PostgreSQL treats weeks as starting on Monday. This is consistent across all queries to ensure weekly buckets align exactly.
* **`EXTRACT(EPOCH FROM …)`**: Converting timestamps to epoch seconds allows efficient numeric comparisons, sorting, and potential joins to other time-series data that also use epoch fields.
* **`COUNT(DISTINCT dev_name)`**: Guarantees that repeated contributions by the same developer do not inflate the “distinct developer count.”
* **`COALESCE(…, 0)`**: Although grouping and counting normally yield zero rows, using `COALESCE` is a defensive measure in case of future schema changes or external joins that might introduce `NULL` values.
* **Ordering**: Each view orders by `commit_week DESC` to ensure that queries against the view will naturally return the most recent week first, simplifying downstream reporting.
