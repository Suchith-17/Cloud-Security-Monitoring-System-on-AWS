# Cloud Security Monitoring System on AWS

## Overview

This project demonstrates a practical Cloud Security Monitoring System on AWS that detects access to secrets stored in AWS Secrets Manager and sends real-time notifications. The solution uses AWS CloudTrail, Amazon CloudWatch, metric filters, CloudWatch Alarms, and Amazon SNS to provide observability and alerting for secret access events.

Key goals:
- Track secret access events (for example, `GetSecretValue`) with CloudTrail.
- Centralize logs in CloudWatch Logs.
- Convert log patterns into actionable CloudWatch metrics.
- Trigger alarms and send notifications via SNS.
- Validate notification behavior and refine alarm configuration.

---

## Architecture

AWS Secrets Manager ➜ AWS CloudTrail ➜ Amazon CloudWatch Logs ➜ Metric Filter ➜ CloudWatch Alarm ➜ Amazon SNS ➜ Email Notification


<img width="1536" height="1024" alt="Cloud monitoring system" src="https://github.com/user-attachments/assets/a549ed46-076f-4800-afa2-0479eb602b2b" />

---

## Security Considerations

- An IAM user with administrative permissions was used only for demonstration and testing.
- The AWS root account was not used.
- In production, always enforce the principle of least privilege with well-scoped IAM roles and policies.
- Consider enabling KMS encryption for log files and long-term retention of logs for compliance and auditing.
- Ensure secure handling of test secrets and remove or rotate them after use.

---

## Prerequisites

- An AWS account with permissions to create and configure:
  - AWS Secrets Manager
  - AWS CloudTrail
  - CloudWatch Logs (log groups, metric filters, alarms)
  - Amazon SNS (topics and subscriptions)
  - IAM roles/policies
- AWS CLI (optional) or Console access for manual testing.

---

## Project Walkthrough

### Step 1 — Create a Secret in AWS Secrets Manager

1. Open AWS Secrets Manager.
2. Create a new secret and set the secret value.
3. (Optional) Configure rotation.
4. Store the secret and verify creation.

Example used in this project:
- Secret name: `mission_top_secret@07`
- Stored value (test): `there is no secret lol`

Screenshots:
- Secret creation and stored value  
  <img width="824" height="332" alt="Secrets Manager" src="https://github.com/user-attachments/assets/e84746bb-6c5e-4d14-8af8-d1a232f5f85d" />

---

### Step 2 — Configure AWS CloudTrail

Configure a CloudTrail trail to capture API activity, including Secrets Manager calls.

- Trail name: `secret_manager_trail`
- S3 bucket used for logs: `maxx-secrets-manager-trail-no`
- Management events: Enabled (Read and Write)
- Note: Server-side encryption for log files (SSE-KMS) was disabled for demo purposes — enable encryption in production.

This configuration ensures `GetSecretValue` and other API calls are recorded.

Screenshot:
<img width="779" height="315" alt="CloudTrail configuration" src="https://github.com/user-attachments/assets/1d0405f8-84ab-4c87-9946-69a3119a344c" />

---

### Step 3 — Generate Secret Access Events

Secrets can be accessed via:
- AWS Console (Secrets Manager)
- AWS CLI or SDKs
- Applications with the appropriate permissions

When `GetSecretValue` is called, CloudTrail records the event. Verify event details in CloudTrail → Event history by filtering for Secrets Manager.

Screenshots:
<img width="940" height="198" alt="CloudTrail event history" src="https://github.com/user-attachments/assets/473286c4-f7e0-43e4-92b0-83ff0fdaf314" />
<img width="940" height="192" alt="GetSecretValue event" src="https://github.com/user-attachments/assets/e9b4ab79-66e3-4d64-8401-e64bcd393e4d" />

---

### Step 4 — Forward CloudTrail Logs to CloudWatch

1. In CloudTrail, edit the trail and enable CloudWatch Logs delivery.
2. Create a new CloudWatch Log Group (for example: `maxx-secret-manger-loggroups`).
3. Create an IAM role that allows CloudTrail to publish logs to CloudWatch.

View logs in CloudWatch:
- Navigate to CloudWatch → Logs → Log Groups → select the log group → open the latest log stream → inspect events.

Screenshots:
<img width="940" height="373" alt="Enable CloudWatch Logs" src="https://github.com/user-attachments/assets/e7da2489-6555-4136-a7ce-23d301f987d9" />
<img width="840" height="337" alt="CloudWatch logs" src="https://github.com/user-attachments/assets/2665784a-c2e8-4955-b2ba-38ce45ff805a" />

---

### Step 5 — Create a Metric Filter

A metric filter converts matching log entries into metric data.

Recommended metric filter configuration:
- Filter pattern: "GetSecretValue"
- Metric namespace: `SecurityMetrics` (or `Security Metrics`)
- Metric name: `SecretAccessed`
- Metric value: `1`

Each matching event emits a metric datapoint with value 1.

Screenshots:
<img width="793" height="325" alt="Metric filter creation" src="https://github.com/user-attachments/assets/a5256659-278d-42cf-815c-6cde0071db91" />
<img width="815" height="332" alt="Metric filter details" src="https://github.com/user-attachments/assets/020e4a6e-4e13-41e2-932e-5fb2c323f87c" />

---

### Step 6 — Create CloudWatch Alarm and SNS Topic

Alarm configuration (example):
- Metric namespace: `SecurityMetrics`
- Metric name: `SecretAccessed`
- Threshold type: Static
- Condition: `>= 1` (trigger when at least one event occurs)
- Statistic: Use `Sum` (recommended for counting occurrences) with an appropriate evaluation period (for example, 1 minute) to ensure consistent alerting.

Create an SNS topic and subscribe an email endpoint:
1. Create SNS Topic.
2. Add an email subscription and confirm it via the received confirmation email.

Screenshots:
- Alarm creation  
  <img width="940" height="235" alt="CloudWatch Alarm" src="https://github.com/user-attachments/assets/0759750e-8851-4ee8-b931-8056745e58f8" />
- SNS Topic and subscription  
  <img width="940" height="373" alt="SNS topic" src="https://github.com/user-attachments/assets/6814b36b-0a64-478b-93cd-def5bd1ccee0" />
  <img width="940" height="409" alt="SNS subscription confirmation" src="https://github.com/user-attachments/assets/7d05a8e2-fa1b-4719-9d95-7d65920da2c5" />

---

### Step 7 — Test Email Notifications and Alarm Behavior

Testing steps:
- Access the secret via Console or CLI to generate a `GetSecretValue` event.
- Confirm the event appears in CloudTrail and CloudWatch Logs.
- Confirm the metric is incremented by the metric filter.
- Verify the CloudWatch Alarm transitions to `ALARM` and SNS sends an email notification.

Manual testing notes:
- Published a test message to SNS to verify email delivery:
  - Subject/message example: "yooo wassup maxx how u doing!"
- Observed that notifications were not delivered when SNS notifications were disabled at the CloudTrail level — logs continued to be delivered to CloudWatch and CloudTrail.
- Alarm statistic was initially set to `Average`, which caused inconsistent behavior; updating the statistic to `Sum` (with a suitable period) produced reliable alerts.

Screenshots:
<img width="817" height="358" alt="Test metric filter" src="https://github.com/user-attachments/assets/9c3c6632-a5c3-4f76-8a03-8b7907513dba" />
<img width="811" height="328" alt="Test metric results" src="https://github.com/user-attachments/assets/639bf9ba-563a-4cd3-b6d8-20d63a454ee2" />
<img width="940" height="147" alt="CLI alarm set state" src="https://github.com/user-attachments/assets/0664d098-666d-439e-ba8f-d011526a3d2f" />
<img width="940" height="147" alt="CLI alarm set state 2" src="https://github.com/user-attachments/assets/0e02f25d-bfe6-4559-aee1-075f489caca9" />
<img width="940" height="173" alt="SNS email received" src="https://github.com/user-attachments/assets/1470aeec-0205-4e61-a639-4d82c825adbd" />
<img width="940" height="311" alt="Alarm triggered and SNS message" src="https://github.com/user-attachments/assets/412400fe-09ff-4470-82ae-1ccca8d856d5" />
<img width="940" height="223" alt="Email notification sample" src="https://github.com/user-attachments/assets/9fa6a427-ef54-409c-810c-fde8cc448480" />
<img width="940" height="518" alt="Final event and alarm" src="https://github.com/user-attachments/assets/cfd9cecd-4660-49da-8ec9-8eca15e0c6d3" />
<img width="940" height="310" alt="Alarm notification delivered" src="https://github.com/user-attachments/assets/442eb047-1e67-4edd-bf1e-975f1a26d017" />

---

## Troubleshooting & Recommendations

- If alarms do not trigger:
  - Verify the metric filter pattern matches the log events (try the Test Pattern feature).
  - Check that metric data points are being emitted to the chosen namespace.
  - Ensure the alarm uses an appropriate statistic (Sum for counts) and evaluation period.
- If SNS notifications are not received:
  - Confirm the subscription via the confirmation email.
  - Check that the SNS topic policy allows publishing from CloudWatch.
  - Review any email provider spam or filtering.
- Production hardening:
  - Enable SSE-KMS for S3 logs and apply retention policies for long-term storage.
  - Replace administrative credentials with least-privilege IAM roles.
  - Consider cross-account or multi-region CloudTrail and CloudWatch for broader visibility.

---

## Conclusion

This project validates a working security monitoring workflow on AWS:
- Secret access events are tracked via CloudTrail.
- Logs are centralized in CloudWatch.
- Metric filters convert logs into actionable metrics.
- CloudWatch Alarms trigger reliably on secret access.
- Amazon SNS delivers real-time email notifications.

This setup provides a strong foundation for cloud observability and security telemetry focused on secret access.

---

## Future Enhancements

- Integrate with Slack, PagerDuty, or Microsoft Teams for richer alerting and incident management.
- Use Amazon EventBridge for advanced routing and correlation of security events.
- Implement least-privilege IAM roles and more granular auditing.
- Enable encryption for logs and implement long-term retention and archival.
- Expand monitoring to multi-region and cross-account scenarios.

---

## Contact / Author

Project maintained by: Suchith-17

For questions or contributions, please open an issue or submit a pull request on the repository.
