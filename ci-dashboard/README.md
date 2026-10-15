# CI dashboard

Live, two-column view of GitHub Actions for **Theriak** and **VIMS**. It is a single
static `index.html` with no backend. It polls the GitHub REST API from the browser
every 30 s and uses ETag requests, so unchanged responses don't count against the
rate limit.

## Run it

Open `index.html` in a browser (`file://` works), or serve the folder with any static
server. On first load it asks for a **fine-grained personal access token** scoped to
`s-a-n-i-n/Theriak` and `s-a-n-i-n/VIMS` with these read-only repository permissions:

| Permission | Used for |
|---|---|
| Actions: Read | runs, jobs, queue, CI health, deploy history |
| Administration: Read | self-hosted runner status (idle / busy / offline) |
| Contents: Read | "N commits not deployed yet" (compare API) |
| Metadata: Read | required by GitHub |

The token is kept in this browser's `localStorage`, or only for the tab when
"Remember" is unticked. It is never written into the page or the repo.

## What each column shows

- **Runners**: busy / idle / offline counts. There is also one row per runner label
  pool, with the number of queued jobs that target it. A pool with jobs waiting and
  no runner online turns red.
- **Queue**: jobs waiting for a runner, how long the oldest has waited (red after
  10 min), what is running now, and runs awaiting environment approval.
- **Last deployment**: when the deploy *job* last succeeded. A run counts only if
  that job ran: a green run whose deploy job was skipped (a build-only dispatch, or
  a newer sha already deployed) is ignored. It also shows the number of commits on
  the default branch that are not deployed yet.
- **CI health**: pass count of the last 10 completed runs on the default branch,
  as dots (oldest → newest, each links to its run). It also shows the median
  duration and how long the branch has been red. Cancelled runs are left out.
- **Nightly**: last result of `health.yml` (Theriak) or `ci-nightly.yml` (VIMS).

Edit `CONFIG` at the top of the script to change repos, label pools, health
workflows, deploy jobs or the poll interval.
