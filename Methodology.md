## Methodology

Our goal is to construct a comprehensive, accurate snapshot of the Stellar ecosystem by aggregating repositories and developer activity from multiple reliable sources. To achieve this, we combine data from official Stellar organization repositories, SCF awarded projects, and any codebases leveraging Soroban SDKs. Each step is designed to ensure that no repo is missed, that forks and dependencies are captured, and that all developer contributions are accounted for in a consistent, repeatable manner. Below, we outline the high-level process that powers our data pipeline.

1. **Enumerate Stellar Organization Repositories**

   * Retrieve all repositories under the Stellar organization on GitHub.
   * Exclude any external projects that have been forked into this organization.
   * Ensures only original Stellar repositories are considered.

2. **Identify Repository Forks**

   * For each repository obtained in Step 1, gather all associated forks.
   * Provides insight into derivative work and community-driven developments.

3. **Search for Soroban-Related Projects**

   * Use the GitHub API to find any repositories that import or depend on `rs-soroban-sdk` (Rust) or `js-soroban-client` (JavaScript).
   * These dependencies serve as indicators that a project is building on Soroban functionality.

4. **Incorporate Community-Funded Projects**

   * Query the Stellar Community Fund awarded projects to obtain a list of GitHub repositories backed by community grants.
   * Funded initiatives are included, even if they’re not linked to the organization or Soroban SDKs.

5. **Compose Unified Repository List**

   * Combine results from Steps 1–4.
   * Each repository is added to the database only once to prevent duplicate entries.
   * Produces a comprehensive list of all Stellar ecosystem repositories: organizational, forked, Soroban-dependent, and SCF awarded projects.

6. **Extract Developer Metrics**
   For each repository, collect information about its contributors—such as commit counts—to build a detailed profile of developer engagement within the Stellar ecosystem. The technical sub-steps are as follows:

   **a. Determine Default Branch and Enumerate All Branches**

   1. Query the database to retrieve the repository’s `default_branch` from the **`repos`** table.
   2. Call `getRepoBranches(repo, org, default_branch)`, which internally hits the GitHub API endpoint:

      ```
      GET /repos/{org}/{repo}/branches?per_page=100&page=N
      ```

      * Begin by adding the `default_branch` to our branch list.
      * Paginate through any additional branches, capturing both the branch name and its most recent commit date.
   3. Persist each branch’s metadata—`repo`, `organisation`, `branch`, and `latest_commit_date`—into the **`branches`** table via:

      ```js
      saveBranch({
        repo,
        organisation,
        branch,
        latest_commit_date
      });
      ```
   4. This allows future runs to fetch only new commits per branch in subsequent executions.

   **b. Retrieve and Store New Commits per Branch**

   1. For each branch, check if a `latest_commit_date` exists in **`branches`**. If it does, include `since={ISO8601}` in the commits request to limit results.
   2. Invoke `getRepoCommits(repo, org)`, which performs:

      1. Fetch the branch list (from step 6a).
      2. For each branch, paginate through:

         ```
         GET /repos/{org}/{repo}/commits?sha={branch}&per_page=100&page=N[&since=...]
         ```
      3. Track the maximum `commit_date` encountered for that branch—this becomes the updated `latest_commit_date`.
      4. Deduplicate commits by SHA and build an array of commit objects:

         ```js
         {
           commit_hash:   "<string>",
           dev_id:        <author.id>,
           dev_name:      "<author.login>",
           repo:          "<repo>",
           organisation:  "<org>",
           branch:        "<branch>",
           commit_date:   "<committer.date>"
         }
         ```
      5. Insert these commit records into the **`commits`** table via:

         ```js
         saveCommits(commitsArray);
         ```
      6. Update **`branches`** with the new `latest_commit_date` for each branch.
   3. Return the total number of new commits fetched (for logging).

   **c. Aggregate Contributor Statistics**

   1. After inserting new commits, call `getRepoContributors(repo, org)` to fetch the contributor list via:

      ```
      GET /repos/{org}/{repo}/contributors?per_page=100&page=N
      ```
   2. For each contributor where `type === "User"`, construct an object:

      ```js
      {
        id:            <contributor.id>,
        dev_name:      "<contributor.login>",
        repo:          "<repo>",
        organisation:  "<org>",
        avatar_url:    "<contributor.avatar_url>",
        contributions: <contributor.contributions> // total for this repo
      }
      ```
   3. Upsert developer profiles into **`devs`** using:

      ```js
      saveDevs([
        {
          id: <contributor.id>,
          dev_name: "<contributor.login>",
          avatar_url: "<contributor.avatar_url>"
        }
      ]);
      ```
   4. Upsert per-repo contribution counts into **`devs_contributions`** via:

      ```js
      saveContributions([
        {
          dev_name: "<contributor.login>",
          repo: "<repo>",
          organisation: "<org>",
          contributions: <contributor.contributions>
        }
      ]);
      ```
   5. Return the count of processed contributor records.

   **d. Persist Repository Metadata**

   1. Call `getRepoInfo(repo, org, dependencies, repo_type)` to retrieve high-level repository metadata via:

      ```
      GET /repos/{org}/{repo}
      ```
   2. Extract the following fields:

      * `stargazers_count` (stars)
      * `default_branch`
      * `created_at`
      * `updated_at`
      * `pushed_at`
      * `owner.type`
      * `forks`
      * Combine these with `dependencies` (JSON array of parent projects/SDKs) and `repo_type` (e.g., “whitelisted”, “dependent”, or “fork”).
   3. Upsert into the **`repos`** table via:

      ```js
      saveRepoInfo({
        repo,
        organisation,
        stargazers_count,
        default_branch,
        created_at,
        updated_at,
        pushed_at,
        owner_type,
        forks,
        dependencies,
        repo_type
      });
      ```
   4. Storing `updated_at` and `pushed_at` timestamps allows the system to skip unchanged repositories in future crawls.

   **e. Database Schema Supporting Developer Metrics**

   * **`repo_type`** (ENUM): `whitelisted`, `dependent`, `fork`, `whitelisted-fork`.
   * **`commits`**: Records each commit’s SHA, author ID/name, repository, branch, and timestamp.
   * **`devs`**: Stores unique developer metadata (ID, username, avatar URL).
   * **`devs_contributions`**: Logs per-developer contribution totals for each `(repo, organisation)`.
   * **`branches`**: Tracks the latest commit timestamp for each `(repo, organisation, branch)`.
   * **`repos`**: Contains summary metadata (stars, default branch, creation/push dates) for each repository.
   * **`repos_list`**: `(repo TEXT, organisation TEXT, repo_type REPO_TYPE, dependencies JSONB)` – raw input list.

   **f. Building Aggregate Views for Metrics**
   The following SQL views are defined to compute higher-level developer metrics across the dataset:

   1. **`overview_view_v2`** (`00014_create-overview-view-v2.sql`)

      * Ecosystem-wide summary: total commits, total repos, total contributors, etc.
   2. **`new_devs_weekly_view`** (`00015_create-new-devs-weekly-view.sql`)

      * Weekly new developer counts based on first commit.
   3. **`commits_weekly_view_v2`** (`00016_create-commits-weekly-view-v2.sql`)

      * Weekly total commit counts across all repositories.

7. **End-to-End Workflow**

   * **Main Loop (`run()`)** iterates through `repos_list`. For each `(repo, organisation, repo_type, dependencies)`:

     1. If `repo_type = 'whitelisted-fork'`, skip metrics extraction.
     2. Else, call `getRepoStatus(...)` to compare GitHub’s `updated_at`/`pushed_at` vs. the database.

        * **If unchanged**: skip directly to refreshing materialized views.
        * **If changed**:

          * Call `getRepoCommits(...)`, `getRepoContributors(...)`, and `getRepoInfo(...)` to ingest new commits, contributors, and metadata.
          * After inserts/updates, invoke `db.refreshView(...)` on all materialized views concurrently.
   * **Pause** for 60 seconds between iterations.
   * On startup, run migrations (`00001_` through `00016_`) to create/alter tables, types, indexes, and views.

Through these sub-steps, Step 6 ensures that commit activity and contributor data are captured in normalized tables, and that a comprehensive set of materialized views computes developer engagement metrics—ranging from monthly active contributors and commit breakdowns to weekly new developer counts and an overarching ecosystem health dashboard.
