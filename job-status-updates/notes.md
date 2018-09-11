# Job Status Updates

## Current Implementation Overview

### Job Status Update Sources

#### Road Runner/Interapps Runner

- Sends updates to RabbitMQ as the job is running.

#### Job Status Listener

- Listens for updates from OSG jobs and forwards them to RabbitMQ.

#### Agave

- Sends updates directly to apps via the UI nginx instance.

### Initial Consumers

#### Job Status Recorder

- Reads job status updates from RabbitMQ and records them in the `job_status_updates` table in the DE database.

### Intermediate Consumers

#### Job Status to Apps Adapter

- Reads job status updates from the `job_status_updates` table in the DE database and sends requests to inform apps of
  the update.

### Processors

#### Apps

- Receives updates from `job-status-to-apps-adapter` and from `agave` via two separate endpoints.
- Locks the job and job step as soon as possible after the request arrives.
- Ignores the status update if the job is in one of the terminal job states (`Failed`, `Completed`, or `Canceled`) or if
  the job state hasn't changed since the last update.
- Updates the status of the job step in the database.
- Updates the status of the job in the database.
- Updates the corresponding batch status if applicable.
- Sends the job status update if the overall job status changed.
- Updates for individual jobs happen quickly.
- Updates for jobs in a batch are somewhat slower because of the batch status check.

## Changes Made Last Week

- `apps` now processes all job status updates in the database for a single job step when it receives a status update.
- Because `apps` now processes all job status updates for a job step when it receives a status update, it can process
  all of the updates while only locking the job step and job once.
- Apps has been modified to skip job status updates that do not change the current job status earlier in the process,
  which allows it to bypass some of the overhead that it was encountering before.
- `job-status-to-apps-adapter` now groups job status updates by external ID, which should help to reduce contention for
  locks on the `jobs` and `job_steps` tables.
- `job-status-to-apps-adapter` now sends out job status update notifications in batches so that the DE can process
  multiple job status updates simultaneously.

## Potential Short-Term Improvement

- The apps service currently loads information about all of the child jobs in a batch in order to determine the status
  of the job. This is slow when the batch contains thousands of jobs.
- We can speed this up by adding an index to the jobs table on the `parent_id` and `status` columns and performing
  database queries to determine the batch status. Doing this on the batch that was giving us fits last week shaved about
  three seconds off of the processing time on my machine.

## Potential Long-Term Improvements

### Separate Notification Processing from Job Status Updates

I'd like to preface this by saying I would like to separate out some tasks. For example, submitting the next step in a
mixed pipeline currently happens when the job status update is being processed. This is suboptimal, and should probably
happen in a separate thread at the very least.

Doing this in a way that's still relatively fast without spamming the user will be tricky. The problem is also
exacerbated by the fact that the apps service currently filters notifications for mixed pipelines based on overall job
status, and it filters notifications for batches based on the overall batch status. I haven't been able to think of a
way to filter notifications in this manner reliably without locking database tables.

Note that we may still want to change the way that notifications are processed; it's just that we might not be able to
avoid having locks on the database tables.

### What I'd Like to Do
