# Kubernetes CronJob Guide

## What is a CronJob?

A Kubernetes `CronJob` is a resource used to run jobs on a scheduled basis. It is similar to the Unix `cron` service, which runs commands at fixed times.

In Kubernetes, a `CronJob` creates a `Job` at the scheduled time, and that `Job` runs one or more Pods to perform the task.

Example:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              args:
                - /bin/sh
                - -c
                - date; echo Hello from the Kubernetes cluster!
          restartPolicy: OnFailure
```

This CronJob runs every minute because the schedule is `* * * * *`.

## What does a CronJob do?

A CronJob does the following:

- checks the configured schedule
- creates a Kubernetes `Job` when the scheduled time matches
- starts one or more Pods for the task
- runs the command or script inside the container
- tracks the Job until it completes or fails

A CronJob is useful for repeating background tasks such as:

- database cleanup
- log rotation
- report generation
- backup jobs
- health checks
- API polling
- sending scheduled notifications

## Where is it used?

CronJobs are commonly used in:

- DevOps automation
- system maintenance
- backup and restore workflows
- reporting systems
- cleanup routines
- monitoring and alerting jobs
- periodic synchronization tasks

Typical real-world examples:

- run a backup at 2:00 AM every day
- delete old log files every Sunday night
- generate sales reports every Monday morning
- check a service endpoint every 5 minutes

## Why is it useful?

CronJobs are useful because they let you automate repetitive work without manual intervention.

Benefits:

- automation of routine tasks
- reliable periodic execution
- reduced manual effort
- consistent scheduled maintenance
- easy integration with Kubernetes workloads
- built-in job tracking and retries

Instead of running a script manually every day, you can let Kubernetes do it automatically at the exact time you define.

## How to schedule a CronJob

The scheduling field in Kubernetes is:

```yaml
spec:
  schedule: "* * * * *"
```

The value follows the standard cron format:

```text
* * * * *
| | | | |
| | | | +-- Day of the week (0-7)  Sunday=0 or 7
| | | +---- Month (1-12)
| | +------ Day of the month (1-31)
| +-------- Hour (0-23)
+---------- Minute (0-59)
```

So the five stars mean:

- first `*` = minute
- second `*` = hour
- third `*` = day of month
- fourth `*` = month
- fifth `*` = day of week

The entire schedule is read as:

`minute hour day-of-month month day-of-week`

## What each `* * * * *` means

### 1. Minute

```text
* * * * *
^
```

This means every minute.

Example:

```yaml
schedule: "0 * * * *"
```

This runs at minute 0 of every hour.

### 2. Hour

```text
* * * * *
  ^
```

This means every hour.

Example:

```yaml
schedule: "*/30 * * * *"
```

This runs every 30 minutes.

### 3. Day of month

```text
* * * * *
    ^
```

This means every day of the month.

Example:

```yaml
schedule: "0 9 * * *"
```

This runs every day at 9:00 AM.

### 4. Month

```text
* * * * *
      ^
```

This means every month.

Example:

```yaml
schedule: "0 0 1 * *"
```

This runs at midnight on the first day of every month.

### 5. Day of week

```text
* * * * *
        ^
```

This means every day of the week.

Example:

```yaml
schedule: "0 9 * * 1"
```

This runs every Monday at 9:00 AM.

## Common schedule examples

### Every minute

```yaml
schedule: "* * * * *"
```

### Every 15 minutes

```yaml
schedule: "*/15 * * * *"
```

### Every hour at minute 30

```yaml
schedule: "30 * * * *"
```

### Every day at 2:00 AM

```yaml
schedule: "0 2 * * *"
```

### Every Monday at 9:00 AM

```yaml
schedule: "0 9 * * 1"
```

### The first day of every month at midnight

```yaml
schedule: "0 0 1 * *"
```

## Important notes

- CronJob schedules use the timezone of the Kubernetes control plane unless configured otherwise.
- `schedule` values are strings, so they must be quoted as shown in YAML.
- If a Job is still running when the next scheduled time arrives, Kubernetes may skip or delay the next run depending on the Job behavior.
- The `restartPolicy` inside the Pod template is usually `OnFailure` or `Never` for Job-based workloads.

## Summary

A `CronJob` in Kubernetes is a scheduled task runner. It is used to automate jobs that must happen repeatedly, such as backups, cleanup, health checks, or reports. The `schedule` field uses a five-field cron expression:

```text
minute hour day-of-month month day-of-week
```

This makes CronJobs a powerful tool for periodic automation in Kubernetes clusters.

## Example command to apply a CronJob

```powershell
kubectl apply -f cronjob.yaml
```

To view CronJobs:

```powershell
kubectl get cronjobs
```

To see Job history or Pods created by the CronJob:

```powershell
kubectl get jobs
kubectl get pods
```

This gives you a good starting point for using CronJobs in your Kubernetes cluster.
