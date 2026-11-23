@Library('jenkins-shared-library@main') _

// Backfill: mode 'next' and no schedule; run the job by hand for each release.
// After the backfill: mode 'all' and schedule 'TZ=UTC\n0 6 * * *' for the nightly run.
infisicalReleaseSyncPipeline(
    mode: 'next',
    schedule: ''
)
