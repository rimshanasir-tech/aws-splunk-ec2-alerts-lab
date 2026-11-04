# AWS EC2 Alerts to Splunk Lab

This lab sends EC2 alerts from AWS to Splunk Enterprise and raises a Splunk alert when a CloudWatch alarm fires. All AWS work was done in one region (US East, N. Virginia, `us-east-1`) on a new AWS Free Tier account.

Every step has a screenshot (folder `images/`). Account IDs, access key IDs and user names were blacked out in the screenshots.

## Architecture

CloudWatch alarm / EC2 state change -> EventBridge rule `ec2-alerts-to-logs` -> CloudWatch log group `/aws/events/ec2-alerts` -> Splunk Add-on for AWS (CloudWatch Logs input) -> Splunk index `aws` -> Splunk alert `AWS EC2 alarm fired`

## Repository structure

```
.
├── README.md
├── iam/SplunkAddonReadOnly.json          # read-only IAM policy for Splunk
├── eventbridge/ec2-alerts-pattern.json  # EventBridge event pattern
├── splunk/alarm-search.spl              # Splunk search used for the alert
└── images/                              # screenshots used below
```

## Resources used

| Item | Name / value |
|---|---|
| EC2 instance | `lab-ec2` (t3.micro, us-east-1) |
| IAM policy | `SplunkAddonReadOnly` |
| IAM user | `splunk-reader` (no console access, one access key) |
| CloudWatch alarms | `ec2-high-cpu`, `ec2-status-check` |
| EventBridge rule | `ec2-alerts-to-logs` (default event bus) |
| Log group | `/aws/events/ec2-alerts` |
| Splunk account | `lab-aws` |
| Splunk input | `ec2-alerts` (CloudWatch Logs, interval 60 s) |
| Splunk index | `aws` |
| Splunk alert | `AWS EC2 alarm fired` |

## Walkthrough

### Part 1: Prepare Splunk

**Step 1.** Open Splunk Enterprise at `http://localhost:8000` and log in as the admin user.

![Step 1 screenshot](images/01-open-splunk-enterprise-at-and.png)

**Step 2.** On Splunkbase, open **Splunk Add-on for Amazon Web Services**, accept the license agreements and download the add-on.

![Step 2 screenshot](images/02-on-splunkbase-open-splunk-add.png)

**Step 3.** In Splunk, install the add-on with **Apps > Manage Apps > Install app from file**, then restart Splunk. The add-on now appears in the Apps list.

![Step 3 screenshot](images/59-in-splunk-install-the-add.png)

**Step 4.** Create an index named `aws` (Settings > Indexes > New Index, App: Search & Reporting).

### Part 2: Create the EC2 instance

**Step 5.** Launch a free-tier-eligible instance named `lab-ec2` (t3.micro) in `us-east-1`. Wait until the state is **Running** and the status checks have passed.

![Step 5 screenshot](images/03-launch-a-free-tier-eligible.png)

### Part 3: IAM policy and user for Splunk

**Step 6.** Open **IAM > Policies > Create policy**.

![Step 6 screenshot](images/04-open-iam-policies-create-policy.png)

![Step 6 screenshot](images/05-open-iam-policies-create-policy.png)

![Step 6 screenshot](images/06-open-iam-policies-create-policy.png)

**Step 7.** Choose the **JSON** editor and paste the read-only policy (logs, cloudwatch, ec2, sns, lambda, elasticloadbalancing, s3:GetBucketLocation, sts:GetCallerIdentity).

![Step 7 screenshot](images/07-choose-the-json-editor-and.png)

**Step 8.** Name the policy `SplunkAddonReadOnly` and create it.

![Step 8 screenshot](images/08-name-the-policy-splunkaddonreadonly-and.png)

![Step 8 screenshot](images/09-name-the-policy-splunkaddonreadonly-and.png)

**Step 9.** Open **IAM > Users > Create user**. Name it `splunk-reader` and leave console access off.

![Step 9 screenshot](images/10-open-iam-users-create-user.png)

![Step 9 screenshot](images/11-open-iam-users-create-user.png)

**Step 10.** Choose **Attach policies directly**, search `spl` and tick `SplunkAddonReadOnly`.

![Step 10 screenshot](images/12-choose-attach-policies-directly-search.png)

![Step 10 screenshot](images/13-choose-attach-policies-directly-search.png)

**Step 11.** Review the permissions summary and click **Create user**.

![Step 11 screenshot](images/14-review-the-permissions-summary-and.png)

![Step 11 screenshot](images/15-review-the-permissions-summary-and.png)

**Step 12.** Open the new user. It shows console access Disabled and no access key yet.

![Step 12 screenshot](images/16-open-the-new-user-it.png)

**Step 13.** Click **Create access key** and choose the use case **Third-party service**.

![Step 13 screenshot](images/17-click-create-access-key-and.png)

**Step 14.** Copy the access key ID and the secret access key to a safe place. The secret is shown only once.

![Step 14 screenshot](images/18-copy-the-access-key-id.png)

### Part 4: CloudWatch alarms

**Step 15.** Open **CloudWatch > Alarms** (0 alarms) and click **Create alarm**.

![Step 15 screenshot](images/19-open-cloudwatch-alarms-and-click.png)

![Step 15 screenshot](images/20-open-cloudwatch-alarms-and-click.png)

![Step 15 screenshot](images/21-open-cloudwatch-alarms-and-click.png)

**Step 16.** Click **Select metric**, then **EC2 > Per-Instance Metrics**.

![Step 16 screenshot](images/22-click-select-metric-then-ec2.png)

![Step 16 screenshot](images/23-click-select-metric-then-ec2.png)

![Step 16 screenshot](images/24-click-select-metric-then-ec2.png)

![Step 16 screenshot](images/25-click-select-metric-then-ec2.png)

**Step 17.** Search `CPUUtilization` and tick the row of `lab-ec2` (the instance with a name). Click **Select metric**.

![Step 17 screenshot](images/26-search-cpuutilization-and-tick-the.png)

**Step 18.** Set Static, Greater, threshold `80`, period 5 minutes. Click **Next**.

![Step 18 screenshot](images/27-set-static-greater-threshold-80.png)

**Step 19.** On **Configure actions** do not add anything. Click **Next** and confirm **Proceed without actions**.

![Step 19 screenshot](images/28-on-configure-actions-do-not.png)

![Step 19 screenshot](images/29-on-configure-actions-do-not.png)

**Step 20.** Name the alarm `ec2-high-cpu`, review and click **Create alarm**.

![Step 20 screenshot](images/30-name-the-alarm-ec2-high.png)

![Step 20 screenshot](images/31-name-the-alarm-ec2-high.png)

![Step 20 screenshot](images/32-name-the-alarm-ec2-high.png)

**Step 21.** Create the second alarm: **Select metric > EC2 > Per-Instance Metrics**, choose `StatusCheckFailed` for `lab-ec2`.

![Step 21 screenshot](images/33-create-the-second-alarm-select.png)

![Step 21 screenshot](images/34-create-the-second-alarm-select.png)

**Step 22.** Set Static, Greater/Equal, threshold `1` (the screenshot shows the Greater option with value 1). No actions.

![Step 22 screenshot](images/35-set-static-greater-equal-threshold.png)

![Step 22 screenshot](images/36-set-static-greater-equal-threshold.png)

![Step 22 screenshot](images/37-set-static-greater-equal-threshold.png)

**Step 23.** Name it `ec2-status-check`, review and create. Both alarms show state **OK**.

![Step 23 screenshot](images/38-name-it-ec2-status-check.png)

![Step 23 screenshot](images/39-name-it-ec2-status-check.png)

![Step 23 screenshot](images/40-name-it-ec2-status-check.png)

### Part 5: EventBridge rule to a log group

**Step 24.** Open **EventBridge** and go to the **default** event bus.

![Step 24 screenshot](images/41-open-eventbridge-and-go-to.png)

![Step 24 screenshot](images/42-open-eventbridge-and-go-to.png)

![Step 24 screenshot](images/43-open-eventbridge-and-go-to.png)

**Step 25.** Click **Create rule** and switch to **Advanced builder** (step-by-step).

![Step 25 screenshot](images/43-open-eventbridge-and-go-to.png)

**Step 26.** Step 1: name `ec2-alerts-to-logs`, event bus `default`, rule type "Rule with an event pattern".

![Step 26 screenshot](images/44-step-1-name-ec2-alerts.png)

**Step 27.** Step 2: choose a **Custom pattern (JSON editor)** and paste the pattern below.

![Step 27 screenshot](images/45-step-2-choose-a-custom.png)

![Step 27 screenshot](images/46-step-2-choose-a-custom.png)

**Step 28.** Step 3: target type **AWS service**, target **CloudWatch log group**, **New log group**. The prefix `/aws/events/` is fixed, so type only `ec2-alerts`.

![Step 28 screenshot](images/47-step-3-target-type-aws.png)

**Step 29.** Step 4: skip tags. Step 5: review the rule and click **Create rule**.

![Step 29 screenshot](images/48-step-4-skip-tags-step.png)

![Step 29 screenshot](images/49-step-4-skip-tags-step.png)

**Step 30.** The rule `ec2-alerts-to-logs` is created and **Enabled**.

![Step 30 screenshot](images/50-the-rule-ec2-alerts-to.png)

Event pattern used:

```json
{
  "source": ["aws.cloudwatch", "aws.guardduty", "aws.ec2"],
  "detail-type": [
    "CloudWatch Alarm State Change",
    "GuardDuty Finding",
    "EC2 Instance State-change Notification"
  ]
}
```

### Part 6: Generate and verify the first events

**Step 31.** In **EC2 > Instances** select `lab-ec2` and choose **Instance state > Stop instance**. Confirm **Stop**.

![Step 31 screenshot](images/51-in-ec2-instances-select-lab.png)

![Step 31 screenshot](images/52-in-ec2-instances-select-lab.png)

**Step 32.** When the state is **Stopped**, choose **Instance state > Start instance**.

![Step 32 screenshot](images/53-when-the-state-is-stopped.png)

**Step 33.** In **CloudWatch**, check that the alarms are **OK**.

![Step 33 screenshot](images/54-in-cloudwatch-check-that-the.png)

**Step 34.** Open **CloudWatch > Log groups**. The group `/aws/events/ec2-alerts` exists.

![Step 34 screenshot](images/55-open-cloudwatch-log-groups-the.png)

![Step 34 screenshot](images/56-open-cloudwatch-log-groups-the.png)

**Step 35.** Open the group. Four log streams were created from the stop/start events.

![Step 35 screenshot](images/57-open-the-group-four-log.png)

**Step 36.** Open a log stream. The event shows `detail-type` "EC2 Instance State-change Notification", `source` `aws.ec2`, region `us-east-1` and the instance state.

![Step 36 screenshot](images/58-open-a-log-stream-the.png)

### Part 7: Connect Splunk to AWS

**Step 37.** In Splunk open **Splunk Add-on for AWS > Configuration > Account**. It has 0 accounts.

![Step 37 screenshot](images/60-in-splunk-open-splunk-add.png)

**Step 38.** Click **Add**. Name `lab-aws`, paste the access key ID and secret key of `splunk-reader`, region category **Global**, then **Add**.

![Step 38 screenshot](images/61-click-add-name-lab-aws.png)

**Step 39.** Open **Inputs > Create New Input > Custom Data Type > CloudWatch Logs**.

![Step 39 screenshot](images/62-open-inputs-create-new-input.png)

![Step 39 screenshot](images/63-open-inputs-create-new-input.png)

![Step 39 screenshot](images/64-open-inputs-create-new-input.png)

**Step 40.** Fill in the form: name `ec2-alerts`, account `lab-aws`, region US East (N. Virginia), log group `/aws/events/ec2-alerts`, stream matching regex `.*`, only after `2026-10-30T00:00:00`, source type `aws:cloudwatchlogs`, index `aws`, query window size `10`, interval `60`. Click **Add**.

![Step 40 screenshot](images/65-fill-in-the-form-name.png)

**Step 41.** The input `ec2-alerts` is listed with status **Active**.

![Step 41 screenshot](images/66-the-input-ec2-alerts-is.png)

### Part 8: Fire the alarm

**Step 42.** CloudShell was not available on the new account (account verification in progress), so the alarm was forced from the console. Edit `ec2-high-cpu` and change the threshold to `0.1`.

![Step 42 screenshot](images/67-cloudshell-was-not-available-on.png)

**Step 43.** Click **Update alarm** on the last page.

![Step 43 screenshot](images/68-click-update-alarm-on-the.png)

**Step 44.** After a few minutes `ec2-high-cpu` goes to **In alarm**.

![Step 44 screenshot](images/69-after-a-few-minutes-ec2.png)

**Step 45.** A new log stream appears in `/aws/events/ec2-alerts` (five streams now).

![Step 45 screenshot](images/70-a-new-log-stream-appears.png)

**Step 46.** Open it. The event has `detail-type` "CloudWatch Alarm State Change", `alarmName` `ec2-high-cpu` and state value `ALARM`.

![Step 46 screenshot](images/71-open-it-the-event-has.png)

![Step 46 screenshot](images/72-open-it-the-event-has.png)

### Part 9: Search and alert in Splunk

**Step 47.** In Splunk Search run the query below (time range: last 60 minutes). It returns the `ec2-high-cpu` ALARM event.

![Step 47 screenshot](images/73-in-splunk-search-run-the.png)

**Step 48.** Click **Save As > Alert** and set: title `AWS EC2 alarm fired`, Scheduled, cron `*/5 * * * *`, time range last 15 minutes, trigger when number of results is greater than 0, **For each result**, throttle on `detail.alarmName` for 30 minutes, action **Add to Triggered Alerts** with severity **High**.

![Step 48 screenshot](images/74-click-save-as-alert-and.png)

**Step 49.** The alert is saved: Enabled, trigger condition "Number of Results is > 0", one action. It has not fired yet.

![Step 49 screenshot](images/75-the-alert-is-saved-enabled.png)

**Step 50.** To trigger it again, edit `ec2-high-cpu`: set the threshold back to `80` so the alarm returns to **OK**.

![Step 50 screenshot](images/76-to-trigger-it-again-edit.png)

![Step 50 screenshot](images/77-to-trigger-it-again-edit.png)

**Step 51.** Edit the alarm again and set the threshold to `0.1`.

![Step 51 screenshot](images/78-edit-the-alarm-again-and.png)

**Step 52.** The alarm is **OK**, then goes **In alarm** again.

![Step 52 screenshot](images/79-the-alarm-is-ok-then.png)

![Step 52 screenshot](images/80-the-alarm-is-ok-then.png)

**Step 53.** In Splunk, **Activity > Triggered Alerts** shows `AWS EC2 alarm fired` (severity High, mode Per Result).

![Step 53 screenshot](images/81-in-splunk-activity-triggered-alerts.png)

Search used:

```
index=aws sourcetype=aws:cloudwatchlogs* "CloudWatch Alarm State Change"
| spath
| search "detail.state.value"=ALARM
| table _time, detail.alarmName, detail.state.reason, region
```

## Result

The CloudWatch alarm event travelled from AWS to Splunk, and the Splunk alert fired and appeared in Triggered Alerts with High severity.

## Cleanup (done after the lab)

1. Disable the `ec2-alerts` input in the Splunk add-on.
2. Delete the alarms `ec2-high-cpu` and `ec2-status-check`.
3. Delete the EventBridge rule `ec2-alerts-to-logs` and the log group `/aws/events/ec2-alerts`.
4. Deactivate and delete the access key of `splunk-reader`, then delete the user and the policy `SplunkAddonReadOnly`.
5. Terminate the EC2 instance `lab-ec2`.

## Notes

- GuardDuty was skipped. It is optional and was not needed for this lab.
- The CloudWatch metrics input (optional) was skipped to avoid extra cost.
- Alarm actions were left empty on purpose. EventBridge picks up the alarm state-change events without an SNS notification.
