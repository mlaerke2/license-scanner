# License Scanner for ServiceNow

This is shared for feedback and for people to try it out. It's possible to adapt the application to a specific setup by creating license definitions that fits your own purpose. This is not affiliated with ServiceNow. 

License Scanner is an investigative app that can be used to explore user statistics that is not available in a ServiceNow instance normally. It compares the licenses users **hold** with how they **actually use** the platform.

For each license definition it finds every user who has a licensed role. It then looks at what they did on the instance during the last 30 days and classifies each one as one of these:

| Acting as | Meaning |
|---|---|
| **Fulfiller** | Created or updated records that belong to someone else in the licensed tables. |
| **Business Stakeholder** | Approved records, or opened records that belong to someone else, without changing them. |
| **Requester** | Neither of the above. A paid role may not be needed. |

The results feed a dashboard. You can use it to find users who have more license than they need (savings) or less than they use (a compliance risk).

---

## Contents

```
.
├── README.md
├── sys_remote_update_set_afe7d86d877c5b50d59c206e8bbb3508.xml   # The application (update set)
└── examples/
    └── license_subscription_examples.xml                         # Example license subscriptions
```

---

## Requirements

- A ServiceNow instance with admin access, to install the update set.
- The role **`usage_admin`**, which every table, list and module in the app requires. Give it to the people who will run the scanner.
- **Auditing switched on** for the tables you want to monitor. Most out-of-the-box task tables are audited by default. Check custom tables and non-task tables.
- Navigator history (`sys_ui_navigator_history`) kept for at least the 30-day lookback.

---

## Installation

1. Go to **System Update Sets → Retrieved Update Sets**.
2. Click **Import Update Set from XML** and upload `sys_remote_update_set_afe7d86d877c5b50d59c206e8bbb3508.xml`.
3. Open the retrieved update set and click **Preview Update Set**. Resolve any problems the preview finds.
4. Click **Commit Update Set**.
5. Give `usage_admin` to the users who will run the scanner.
6. Create the license subscriptions (see [Configuring a license subscription](#configuring-a-license-subscription)) or import the examples.

> **Note:** The dashboard is in the Global scope and is **not** part of the update set. Build it in Platform Analytics on the **License Monitor** table. Filter on *Subscription* and group by *Acting as*.

---

## Data model

### License Subscription (`x_delde_license_subscription`, prefix `LISUB`)

Each record is one license definition to monitor.

| Field | Type | Description |
|---|---|---|
| Name | String | Name of the license definition. |
| Role type | Reference → `license_role_type` | The license type being measured: *Fulfiller* or *Business Stakeholder*. |
| Application entitlement | List → `license_family` | The license families that make up the license. |
| Associated roles | String | Licensed roles, filled in from *Application entitlement* and *Role type*. |
| Associated tables | List → `sys_db_object` | Tables whose activity counts as using the license. |
| Table names | String | Table names, filled in from *Associated tables*. |
| Subscription entitlement | Reference | Optional link to the platform subscription record. |
| Scan status | Choice | 100 Not Initiated · 200 Initiated · 300 In Progress · 400 Skipped · 500 Completed · 600 Error |
| Last scanned | Date/time | When the last scan finished. |
| Active | True/False | Only active subscriptions are scanned by the scheduled job. |
| Description | HTML | The rules used to classify users. |

### License Monitor (`x_delde_license_license_monitor`, prefix `LMON`)

Each record is one user in one subscription.

| Field | Description |
|---|---|
| Subscription | The license subscription that was scanned. |
| User | The user who holds one or more of the associated roles. |
| Role type | The license type the user holds. |
| Acting as | How the user actually behaves: Fulfiller, Business Stakeholder or Requester. |
| Status | Status of the record. |

---

## How a scan works

1. **Configure.** When you set *Application entitlement* and *Role type* on a subscription, a business rule looks up the licensed roles in `license_role` and stores them in *Associated roles*. When you set *Associated tables*, another business rule stores the table names in *Table names*.
2. **Start.** A scan starts in one of two ways:
   - the **Scan license** UI action on the subscription form, or
   - the scheduled job, which runs weekly on Monday at 00:00 for every active subscription.

   Both set *Scan status* to **Initiated (200)**.
3. **Scan.** An async business rule sets the status to **In Progress (300)** and calls `ScannerUtils.processScan()`, which:
   - finds active, interactive users who are granted the associated roles directly,
   - leaves out users with `admin`,
   - deletes the previous monitor records for the subscription,
   - creates one monitor record per user and sets the subscription to **Completed (500)**.
4. **Classify.** When each monitor record is inserted, the business rule *Check users Acting as status* calls `ScannerUtils.isActingAsFulfiller()`. It works out the user's *Acting as* value from:
   - `sys_audit`: writes in the associated tables, within the last 30 days, on records created by someone else (gives **Fulfiller**),
   - `sysapproval_approver`: approvals (gives **Business Stakeholder**),
   - `sys_ui_navigator_history`: records the user opened that were created by someone else (gives **Business Stakeholder**).

---

## Configuring a license subscription

1. Go to **License Scanner → License Subscriptions → New**.
2. Fill in **Name**, **Role type**, **Application entitlement** (the license families) and **Associated tables**.
3. Save. Check that *Associated roles* and *Table names* have been filled in.
4. Tick **Active** and click **Scan license**.

### Example subscriptions

[`examples/license_subscription_examples.xml`](examples/license_subscription_examples.xml) holds seven example definitions:

| Subscription | Role type | Main tables |
|---|---|---|
| Customer Service Management - Fulfiller | Fulfiller | `sn_customerservice_case`, `sn_customerservice_task`, `sn_customerservice_escalation`, `customer_account`, `customer_contact`, `csm_consumer`, `interaction` |
| Customer Service Management - Business Stakeholder | Business Stakeholder | CSM case, task, escalation, account, contact and consumer tables |
| Integrated Risk Management - Fulfiller | Fulfiller | Policy & Compliance, Risk, Advanced Risk, Issues, Audit, Operational Resilience, Regulatory Change and Continuous Authorization tables (24 in total) |
| Integrated Risk Management - Business Stakeholder | Business Stakeholder | Policy exception, risk, risk response, issue, remediation task, compliance case, audit engagement |
| Vulnerability Response - Fulfiller | Fulfiller | `sn_vul_vulnerable_item`, `sn_vul_vulnerability`, application vulnerability tables, `sn_vulc_result_group` |
| Field Service Management - Fulfiller | Fulfiller | `wm_order`, `wm_task` |
| Agile Development (SPM) - Fulfiller | Fulfiller | `rm_story`, `rm_epic`, `rm_defect`, `rm_sprint`, `rm_scrum_task`, `rm_release_scrum` |

To import them, open the **License Subscription** list, right-click a column header, choose **Import XML** and upload the file. The records arrive as active, with scan status *Not Initiated*.

> **Important: the examples are tied to the instance they were built on.**
> - *Application entitlement*, *Associated tables* and *Role type* hold sys_ids from the source instance. On another instance, check each record and select the license families and tables again if needed. Saving the record then fills in *Associated roles* and *Table names* again.
> - On the source instance, the main GRC roles (`sn_compliance.*`, `sn_risk.*`, `sn_grc.user/manager/admin`) and some CSM roles (family `crm_customer_and_case_core`) are not linked to a license family in `license_role`. They were therefore **added to *Associated roles* by hand**. If you change *Application entitlement* on those records later, the hand-added roles are overwritten.
> - *Subscription entitlement* is left empty. Link it to the right subscription on your instance.

---

## Known limitations

- **Business Stakeholder subscriptions never detect Fulfiller behaviour.** If the subscription name contains "Stakeholder", the audit check is skipped.
- **Approvals have no date limit.** One old approval is enough to make a user a Business Stakeholder.
- **No history is kept.** Every scan deletes the previous results for the subscription, so trends over time are not available.
- **A failed scan can stay *In Progress* (300) forever.** Neither the UI action nor the scheduled job picks it up again until someone resets the status by hand.
- **Only Service Operations Workspace URLs are understood.** Records opened in other workspaces are not counted.
- **Inherited roles are ignored.** Users who get a licensed role only through another role are not counted.
- **Admins are always left out** of the scan.
- **Monitor *Status* stays *In progress*.** It is never set to *Completed*.
- **The 30-day lookback is hardcoded** in `ScannerUtils`.
- **Cross-scope privileges** cover only some of the monitored tables. The app depends on *runtime access tracking* being set to *permissive*.

---

## Roadmap

- Keep a history of scan runs so trends can be shown.
- Add error handling that sets status *Error (600)*, and reset scans that are stuck.
- Run the Fulfiller check for every subscription type, and limit approvals to the scan window.
- Make the lookback period, excluded roles and workspace URL patterns configurable through system properties.
- Move the dashboard into the application scope.
- Add ATF tests for the classification logic.
- Show the potential saving per subscription, based on the license price.
