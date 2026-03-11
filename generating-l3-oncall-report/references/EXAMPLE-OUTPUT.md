# Example Output Reference

Real report from November 10, 2025 rotation. Use this as the quality bar for generated reports.

Source: https://www.notion.so/2a7eace5de2b80dca65afbd606566790

## Contents
- [Properties](#properties) — Notion page properties
- [Page Content](#page-content) — full rendered report
- [Key formatting patterns](#key-formatting-patterns) — Deep Dive alerts, actionable items, pending cases, significant events

## Properties

| Property | Value |
|----------|-------|
| Title | `Engineering L3 On-Call @November 10, 2025` |
| Start rotation | `2025-11-03` |
| Rotation L3 | Bryan Condor |
| Next Rotation L3 | (next engineer) |
| Team | `Merchant SMB`, `Merchant Product` |

## Page Content

```
# On-Call Hand-off
---

### Squad Report {toggle="true"}

  **Owner:** @Bryan Condor

  ### Relevant Incidents from the last week

  - [Incident page title](https://www.notion.so/INCIDENT_PAGE_ID)

  ### Relevant remediation actions update

  - https://app.asana.com/1/705028621614854/project/1201765457329654/task/1211757270632698?focus=true (in progress)
  - https://app.asana.com/1/705028621614854/project/1201765457329654/task/1211757270632690?focus=true (in progress)

  ### Alert Groups

  [screenshot: alerts overview]

  - Pending cases to resolve (to hand over to the next on-call engineer):
    - Support Cases
      - https://app.asana.com/1/.../task/1211866903799237?focus=true
      - 5 additional not started cases has been re-assign through Asana.
    - Marketplace Cancellation
      - https://addico.slack.com/archives/C06PV441GLA/p1762287061418369
      - https://addico.slack.com/archives/C06PV441GLA/p1762292976781689
      - https://addico.slack.com/archives/C06PV441GLA/p1762193427687899
  - Most frequent alerts triggered:
    - **High Severity**
      [screenshot: high severity alerts]
    - **Low Severity**
      [screenshot: low severity alerts]
    - **Deep Dive Relevant Alerts**
      - **Corbeta Intermittence errors:**
        - [SMB] Corbeta Orders Integration | Channels In-store system internal API
        - [SMB] Corbeta API | Get Orders
        - [SMB] Corbeta API | Create Orders
        ⏩ Solution: Follow existing run-book.
      - **Merchant Discounts API errors:**
        - [SMB] Benefit Merchant Discounts | Promotions Internal API
        ⏩ Solution: Following run-book. RUN EVERY DAY AT NIGHT [Runbook link](https://www.notion.so/...)
        🟢 Enhancement: Run-book updated with semi-automate process through Python scripts.
        🟢 Enhancement: Isolated merchant discount API. It will be rollout @November 10.
      - **Intermittence errors:**
        - [Merchant Platform] Channels | Virtual Card OTP Generation | CO
        - [Merchant Platform] Channels | API Errors | Sodimac | CO
        ⏩ Solution: Follow existing run-book.
      - **Ledger Errors**
        - [Merchant Platform] Ally Payment Conditions | Failed updating bank account information on ally activation
        - [Merchant Platform] Merchant Transactions | Ledger | Create Cancellation | Failed to create transaction
        - [Merchant Platform] Ally Payment Job | Ledger | Error generating report on the ledger
        ⏩ Solution: Follow existing run-book. [Runbook link](https://www.notion.so/...)
        🟢 Enhancement: Run-book updated to have lightweight version for ally management jobs.
      - **Bad configuration alert (pending to adjust alert)**
        - Identity Management System.
        - Ally Activation Request Absence.

  ### Bugs Backlog

  [screenshot: Bugs Backlog Merchant Acquisition]
  [screenshot: Bugs Backlog Merchant Success]

  ### Accepted bugs update

  - https://app.asana.com/1/705028621614854/project/1200680916517745/task/1211823920345811?focus=true

  ### Outstanding Postmortems

  - None

  ### Significant Events

  > Please provide a summary of the key insights and events from your on-call experience.

  1. Grafana Degradation:
     1. Alert False Positives about absence alert.
     2. Lack of visibility operational work.
  2. Execute Merchant Transaction Operational Tasks Every day IN THE MORNING (Monday to Sunday) [Runbook](https://www.notion.so/...)
  3. N8N General Error Handling in #merchant-platform-alert has been created.

  ### Actionable items

  > What are the actionable items you found during the on-call?

  1. Deep Review about affected cases due discounts alert not solving in the previous week. https://app.asana.com/1/.../task/1211838059357862?focus=true
  2. Python Jobs locally to check affecting discount applications https://app.asana.com/1/.../task/1211838059357866?focus=true
  3. N8N Workflow Error Handling: Track whatever uncontrolled errors. https://app.asana.com/1/.../task/1211878100482805?focus=true
  4. Benefit API run-book update to include Python Scripts https://app.asana.com/1/.../task/1211901080706668?focus=true
  5. Run-book updated: Merchant Transactions Operation - Ally Management Lightweight Revision https://app.asana.com/1/.../task/1211902715052307?focus=true
  6. [LOW] N8N On-call Marketplace cancellation: Google Sheet Fix https://app.asana.com/1/.../task/1211775837611759?focus=true

### Meetings notes {toggle="true"}

  > Schedule a meeting for the handover and use this space for your notes

  - Most frequently escalated cases:
    - Manual input for the L2 cases escalated from Support Engineers.
  - Alerts created or updated:
    - Manual check for a GitHub link per squad represents the new and updated alerts.
    - Manual check for other alerts that are not in GitHub, like LogRocket.
  - Runbooks created or updated
    - Manual check.
  - Bugs, enhancements, and roadmap tasks created and Implemented
    - Related to escalated cases and alerts
    - Manual check.
```

## Key formatting patterns

### Deep Dive Relevant Alerts — group by theme

Alerts are NOT listed individually. Instead, **group related alerts by theme** with a bold group name, then list child alert names and a shared Solution/Enhancement:

```
- **[Theme name]:**
  - [Alert name 1]
  - [Alert name 2]
  ⏩ Solution: [description]
  🟢 Enhancement: [optional improvement made]
```

### Actionable items — numbered with description + link

Each item has a **brief description of what was done**, followed by the Asana task URL:

```
1. [Description of work done] [ASANA_URL]
2. [Description of work done] [ASANA_URL]
```

### Pending cases — categorized

Group pending cases by source/type:

```
- Support Cases
  - [link]
  - [additional context if needed]
- Marketplace Cancellation
  - [link]
```

### Significant events — numbered with nested detail

```
1. [Event title]:
   1. [Detail]
   2. [Detail]
2. [Event title] [optional link]
```
