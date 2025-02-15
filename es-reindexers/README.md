# ES Reindexing
Scripts to trigger es reindexing.

## **Prerequisites**
* Curl utility.
* AWS CLI.
* [ElasticDump](https://github.com/elasticsearch-dump/elasticsearch-dump) Tool.
* Requires an host within in the same network incase interacting with private elastic/opensearch instances.

## **Usage Examples**
This script is fairly simple to use and the options are self explanatory. Below are some of the example for usage.

```
Case 1: Reindex data
export SOURCE_ES='https://admin:xxxxxx@xxxxxx.us-west-2.es.amazonaws.com'
export SOURCE_INDEX="clair-test-index"
export DESTINATION_ES='https://ospst:xxxxxx@xxxxxx.intlab.redhat.com'
export DESTINATION_INDEX="ospst-clair-test-index-1"
sh elastic_reindex.sh

Case 2: Only backup the data but not reindex
export SOURCE_ES='https://admin:xxxxxx@xxxxxx.us-west-2.es.amazonaws.com'
export SOURCE_INDEX="clair-test-index"
export DESTINATION_ES='https://ospst:xxxxxx@xxxxxx.intlab.redhat.com'
export DESTINATION_INDEX="clair-test-index-2"
export BACKUP_ONLY=true
sh elastic_reindex.sh

Case 3: Backup and redindex all the data in an index along with its settings/configuration within given timerange
export START_TIME="2023-06-25T00:00:00Z" 
export END_TIME="2023-08-28T00:00:00Z" 
export SOURCE_ES='https://admin:xxxxxx@xxxxxx.us-west-2.es.amazonaws.com'
export SOURCE_INDEX="clair-test-index"
export DESTINATION_ES='https://ospst:xxxxxx@xxxxxx.intlab.redhat.com'
export DESTINATION_INDEX="clair-test-index-3"
export INITIAL_RUN=true
sh elastic_reindex.sh

Case 4: Trigger a backfill job to restore data from previous runs. Also has dry run option
export SOURCE_ES='https://admin:xxxxxx@xxxxxx.us-west-2.es.amazonaws.com'
export SOURCE_INDEX="ingress-performance"
export DESTINATION_ES='https://ospst:xxxxxx@xxxxxx.intlab.redhat.com'
export DESTINATION_INDEX="ingress-performance-baseline"
export BACKFILL=true
# Use this option when you want to only list backup files
export BACKFILL_DRY_RUN=true
# Use this option when actual indices are backed by a different name in alias
export BACKFILL_INDEX="baseline-ingress-performance"
sh elastic_reindex.sh

Case 5: Perform a direct reindex without any backup
export START_TIME="2023-06-25T00:00:00Z" 
export END_TIME="2023-08-28T00:00:00Z" 
export SOURCE_ES='https://admin:xxxxxx@xxxxxx.us-west-2.es.amazonaws.com'
export SOURCE_INDEX="clair-test-index"
export DESTINATION_ES='https://ospst:xxxxxx@xxxxxx.intlab.redhat.com'
export DESTINATION_INDEX="clair-test-index-5"
export REINDEX_ONLY=true
sh elastic_reindex.sh
```
## **How to setup a cron job for periodic runs**
To setup a cron job we need to have the below tools as prerequisites.
* Systemd (For setting up timers and services)
* logrotate (Optional: For log rotation)

### **Steps**
> **NOTE**: It is recommended to have separate set of systemd service, timer and a driver script for each index, which acts as a periodic sync job.

For example if we have a test index (i.e `estestindex`) for which we want a periodic sync to happen, below are steps.
### Setup a systemd timer
Setup a systemd timer `perfscale-es-estestindex.timer` with desired cadence for its execution (for example `8/12:00` here indicates execution for every 12 hours starting from 8th hour of a day). Cron expression can be decided based on individual's use.
```
[Unit]
Description=Perfscale ES estestindex Timer
Requires=perfscale-es-estestindex.service
[Timer]
OnCalendar=8/12:00
Unit=perfscale-es-estestindex.service
[Install]
WantedBy=timers.target
```
And also make sure the desired `.service` unit is mentioned here, basically it answers the question on which service unit needs to be triggered on the given cron expression.

### Setup a systemd service
Now setup a service `perfscale-es-estestindex.service` with the below details pointing out the above timer in unit section.
```
[Unit]
Description=Perfscale ES estestindex sync job
Wants=perfscale-es-estestindex.timer
[Service]
Type=simple
User=root
WorkingDirectory=/root
ExecStart=/bin/bash perfscale-es-estestindex.sh
[Install]
WantedBy=multi-user.target
```
Please make a note of `ExecStart` here which basically indicates which command to execute when this timer is started. Here we have a shell script being triggered in our case.

### Add your envs and driver logic in a shell script
Now create a shell script `perfscale-es-estestindex.sh` which was pointed above in the systemd service. This script calls the `elastic-reindex.sh` with required ENVs and notifies us about the result of the execution through slack.
```
#!/bin/bash
export START_TIME=$(date -d '3 days ago' +'%Y-%m-%dT%H:%M:%S');
export END_TIME=$(date +'%Y-%m-%dT%H:%M:%S');
export AWS_ACCESS_KEY_ID=XXXXXXX;
export AWS_SECRET_ACCESS_KEY=XXXXXXX;
export AWS_DEFAULT_REGION="REGION";
export S3_BUCKET=BUCKET_NAME";
export SOURCE_ES=ES_URL;
export SOURCE_INDEX=ES_INDEX;
export DESTINATION_ES=ES_URL;
export DESTINATION_INDEX=ES_INDEX;
export LOG_FILE="/var/log/perfscale-es-estestindex-$(date +'%Y%m%d%H%M%S').log";
export LOG_FILE_PREFIX="/var/log/perfscale-es-estestindex-$(date +'%Y%m%d')";
export WEBHOOK_URL=SLACK_WEBHOOK_URL;
export TOUCH_FILE=TOUCH_FILE.txt;
echo "Please tail for logs at $LOG_FILE";
/bin/bash /root/elastic-reindex.sh >> $LOG_FILE 2>&1;
exit_status=$?;
log_file_preview="$(tail -n 10 "$LOG_FILE")";

if [ "$exit_status" -eq 3 ]; then
  echo "Obvious success case, skipping the slack notification"
  exit 0
fi

if [ -z "$log_file_preview" ] || [ $exit_status -ne 0 ]; then
    message=":alert-siren: ES Reindexing Job *JOBNAME* ended with *failure*. Please review full logs on host:$(hostname) path:*$LOG_FILE* for more details :failed:";
else
    message=":success: ES Reindexing Job *JOBNAME* ended with *success*. Please take a look at logs prefixed with *$LOG_FILE_PREFIX* on host:$(hostname) for more details :white_check_mark:";
    message="$message \n\`\`\`\n$log_file_preview\n\`\`\`";
fi
curl -X POST -H 'Content-type: application/x-www-form-urlencoded' --data-urlencode "payload={\"text\": \"$message\"}" $WEBHOOK_URL;
exit $exit_status;
```
Now we are all set with the required assets to trigger a period job. Please execute the below commands to active the sync for `estestindex`

```
systemctl daemon-reload
systemctl restart perfscale-es-estestindex.timer
systemctl restart perfscale-es-estestindex.service
```
This should enable the specifc timer and service and make job run based on the cron expression specified.

Verify list of timers and their cadence and related service info.
```
systemctl list-timers
```

To check logs and status of a service execution, use the below commands
```
journalctl -u perfscale-es-estestindex
systemctl status perfscale-es-estestindex.service
```

## **Important Points To Note**
* While creating indices, always try to have unique aliases assigned to them to avoid collisions during querying.
* Never create an index with a name that is a substring and prefix of another index. This will create issues while applying index-patterns to those set of indices which are required for rollover activities.
  * Example for INVALID case: `ingress-performance` and `ingress-performance-baseline`
  * Example for VALID case: `ingress-performance` and `baseline-ingress-performance`

## Current list of configured jobs in jump host

### Perfscale ES Prod Instance Jobs
| Timer | Service | Driver Script | Source Index | Destination Index | Cadence |
| ------------------------------ | ---------------------- | ------------------ | -------------------------- | -------------------------- | -------------------------- |
| perfscale-es-ripsaw-kube-burner-prod.timer | perfscale-es-ripsaw-kube-burner-prod.service | perfscale-es-ripsaw-kube-burner-prod.sh | ripsaw-kube-burner | ospst-ripsaw-kube-burner | \*-\*-\* *:0/5:00 |
| perfscale-es-ingress-performance-prod.timer | perfscale-es-ingress-performance-prod.service | perfscale-es-ingress-performance-prod.sh | ingress-performance | ospst-ingress-performance | \*-\*-\* *:0/4:00 |
| perfscale-es-k8s-netperf-prod.timer | perfscale-es-k8s-netperf-prod.service | perfscale-es-k8s-netperf-prod.sh | k8s-netperf | ospst-k8s-netperf | \*-\*-\* *:0/1:00 |

### Perfscale ES OCP QE Instance Jobs
| Timer | Service | Driver Script | Source Index | Destination Index | Cadence |
| ------------------------------ | ---------------------- | ------------------ | -------------------------- | -------------------------- | -------------------------- |
| perfscale-es-ripsaw-kube-burner-ocp-qe.timer | perfscale-es-ripsaw-kube-burner-ocp-qe.service | perfscale-es-ripsaw-kube-burner-ocp-qe.sh | ripsaw-kube-burner | ospst-ripsaw-kube-burner | \*-\*-\* *:0/10:00 |
| perfscale-es-ingress-performance-ocp-qe.timer | perfscale-es-ingress-performance-ocp-qe.service | perfscale-es-ingress-performance-ocp-qe.sh | ingress-performance | ospst-ingress-performance | \*-\*-\* *:0/6:00 |
| perfscale-es-k8s-netperf-ocp-qe.timer | perfscale-es-k8s-netperf-ocp-qe.service | perfscale-es-k8s-netperf-ocp-qe.sh | k8s-netperf | ospst-k8s-netperf | \*-\*-\* *:0/3:00 |
| perfscale-es-perf-scale-ci-ocp-qe.timer | perfscale-es-perf-scale-ci-ocp-qe.service | perfscale-es-perf-scale-ci-ocp-qe.sh | perf_scale_ci | ospst-perf-scale-ci | \*-\*-\* *:0/7:00 |
| perfscale-es-prod-netobserv-datapoints-ocp-qe.timer | perfscale-es-prod-netobserv-datapoints-ocp-qe.service | perfscale-es-prod-netobserv-datapoints-ocp-qe.sh | prod-netobserv-datapoints | ospst-prod-netobserv-datapoints | \*-\*-\* \*:\*:00 |
| perfscale-es-prod-netobserv-operator-metadata-ocp-qe.timer | perfscale-es-prod-netobserv-operator-metadata-ocp-qe.service | perfscale-es-prod-netobserv-operator-metadata-ocp-qe.sh | prod-netobserv-operator-metadata | ospst-prod-netobserv-operator-metadata | \*-\*-\* \*:\*:00 |
| perfscale-es-prod-netobserv-jenkins-metadata-ocp-qe.timer | perfscale-es-prod-netobserv-jenkins-metadata-ocp-qe.service | perfscale-es-prod-netobserv-jenkins-metadata-ocp-qe.sh | prod-netobserv-jenkins-metadata | ospst-prod-netobserv-jenkins-metadata | \*-\*-\* \*:\*:00 |
| perfscale-es-prod-netobserv-baselines-ocp-qe.timer | perfscale-es-prod-netobserv-baselines-ocp-qe.service | perfscale-es-prod-netobserv-baselines-ocp-qe.sh | prod-netobserv-baselines | prod-netobserv-baselines | \*-\*-\* \*:\*:00 |

### Crontab Visualization

![alt text](crontab_schedule.png)