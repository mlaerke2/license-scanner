# License Scanner for ServiceNow

This is shared for feedback and for people to try it out. It's possible to adapt the application to a specific setup by creating license definitions that fits your own purpose. This is not affiliated with ServiceNow. 

License Scanner is an investigative app that can be used to explore user statistics that is not available in a ServiceNow instance normally. It compares the licenses users **hold** with how they **actually use** the platform.

For each license definition it finds every user who has a licensed role. It then looks at what they did on the instance during the last 30 days (configurable) and classifies each one as one of these:

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
├── license_scanner_update_set.xml        # The application as an update set (version 1.1.0)
├── 1220c0668775f6509b22a7980cbb35a0/     # The application source, linked to ServiceNow Source Control
├── sn_source_control.properties          # Tells ServiceNow where the application lives in this repo
└── examples/
    └── license_subscription_examples.xml # Example license subscriptions
```

---

## Requirements

- A ServiceNow instance with admin access to install the application.
- The role **`usage_admin`**, which the tables, lists and dashboard in the app require. Give it to the people who will run the scanner.
- **Auditing switched on** for the tables you want to monitor. Most out-of-the-box task tables are audited by default. Check custom tables and non-task tables.
- Navigator history (`sys_ui_navigator_history`) kept for at least the lookback period.
- The **Platform Analytics** dashboard experience, to use the included dashboard.

---

## Installation

Choose one of the two options.

### Option A: update set

1. Go to **System Update Sets → Retrieved Update Sets**.
2. Click **Import Update Set from XML** and upload `license_scanner_update_set.xml`.
3. Open the retrieved update set and click **Preview Update Set**. Resolve any problems the preview finds.
4. Click **Commit Update Set**.

### Option B: source control

1. Open **ServiceNow Studio** and choose **Import from source control**.
2. Enter this repository's URL and pick the `main` branch. A public repository needs no credential for importing.

### After installing

1. Give `usage_admin` to the users who will run the scanner.
2. Create the license subscriptions (see [Configuring a license subscription](#configuring-a-license-subscription)) or import the examples.
3. Review the [system properties](#system-properties) under **License Scanner → Administration → Properties**.

The dashboard is included and opens from **License Scanner → Dashboard**.

The **Overview** tab shows the latest scan of every subscription:

| Widget | Shows |
|---|---|
| Subscriptions overview | Users per subscription. |
| Acting as compared to Role type | Heatmap of the license users hold against how they behave. |
| Acting as per subscription | The same split for each subscription, so you can see which module the savings are in. |
| Fulfiller – review required | Users with a Fulfiller license who act as Business Stakeholder or Requester (possible savings). |
| Business Stakeholder – review required | Users with a Business Stakeholder license who act as Requester or Fulfiller. |

The **Trend** tab shows the weekly development from [Scan History](#scan-history-x_delde_license_scan_history):

| Widget | Shows |
|---|---|
| Users by behaviour over time | Fulfiller, Business Stakeholder and Requester across all subscriptions, per week. |
| Unused licenses (Requester) per subscription | Users who hold a license but act as Requester, per subscription and week. |

### Employee Center widget

The app includes a portal page, `/esc?id=x_delde_license_usage`, with the **License usage** widget (`x-delde-license-usage`). It shows the same results as the dashboard in a form meant for license and process owners:

- a module filter (modules are found the same way as in the scanner: subscriptions that share tables),
- four key figures: licensed users, users at license level, possible downgrades (Fulfiller license, Business Stakeholder use) and licenses not used (Requester),
- one stacked bar per subscription,
- the weekly trend of unused licenses from Scan History,
- the first users to review, with the scanner's evidence, and a link to the full list.

Everything is read live from Scan History and License Monitor. The page and the widget's data require `usage_admin`. The widget is built for the Employee Center (`esc`) portal and moves clear of its left navigation rail. On another portal, add the widget to a page there.

### Menu

| Module | Opens |
|---|---|
| License definition | License subscriptions. |
| License Monitors | One record per user per scan. |
| Scan History | One record per subscription per scan. |
| Dashboard | The License Scanner dashboard. |
| License usage (portal) | The license usage page on the Employee Center portal, in a new tab. |
| Administration → Properties | The app's system properties (admin only). |
| Administration → ScanUtils, Schedule Job, Business rule | The app's code (admin only). |

---

## Upgrading from 1.0.0

Version 1.1.0 has the same application sys_id and scope (`x_delde_license`) as 1.0.0, so the update set upgrades the app in place. License subscriptions and role assignments are kept. Try the upgrade on a sub-production instance first.

### Before you install

- **Back up the current version.** Open the application record (*System Applications → My Company Applications → License Scanner*), click **Publish to Update Set**, and export that update set as XML.
- **Note local changes.** 1.1.0 replaces the dashboard, the business rules, the scheduled job and `ScannerUtils`. Changes made to them on the instance are overwritten.
- **Do not link the installed app to this repository.** Linking an existing application pushes the instance's version into the repository. *Import from source control* only works when the app is not installed yet. Use the update set.

### Install

1. Import `license_scanner_update_set.xml` under **System Update Sets → Retrieved Update Sets**.
2. Click **Preview Update Set**. In 1.0.0 the dashboard was in the Global scope with the same sys_ids. If the preview reports problems on the `par_dashboard_*` records, choose **Accept remote update** so the records move into the application scope.
3. Click **Commit Update Set**. The application record now shows version 1.1.0.

### Clean up data from 1.0.0

Monitor records from 1.0.0 have no *Scan run* and no *Latest* value. They are not shown on the dashboard, and they are not retired or purged. Scans that 1.0.0 left *In Progress* never run again. Run this once under **System Definition → Scripts - Background**, with the scope set to **x_delde_license**:

```javascript
// Monitor records from 1.0.0 become history (purged after history_retention_days)
var mon = new GlideRecord('x_delde_license_license_monitor');
mon.addNullQuery('scan_run');
mon.setValue('latest', false);
mon.setWorkflow(false);
mon.updateMultiple();

// Scans left "In Progress" by 1.0.0 can run again
var sub = new GlideRecord('x_delde_license_subscription');
sub.addQuery('scan_status', '300');
sub.setValue('scan_status', '100');
sub.setValue('scan_message', 'Reset after upgrade to 1.1.0');
sub.updateMultiple();
```

### Check the configuration

- Review the [system properties](#system-properties) under **License Scanner → Administration → Properties**. The defaults match 1.0.0: a 30-day lookback and `admin` users left out.
- Subscriptions where roles were added to *Associated roles* by hand keep them. Saving a new *Application entitlement* recalculates the roles from `license_role` and removes the hand-added ones.

### Scan and verify

Click **Scan license** on each active subscription, or wait for the weekly job. Then check:

- every subscription has *Scan status* **Completed** and a *Scan message*,
- **Scan History** has a record for each subscription,
- the dashboard's **Overview** tab shows data. The **Trend** tab builds up one week at a time.

### Why the numbers change

Results from 1.1.0 differ from 1.0.0. These are the fixes at work, not errors:

- Users whose *Web service access only* flag is empty are now included. 1.0.0 left them out.
- A user is counted once per module, under the highest license they hold. 1.0.0 counted fulfillers again under Business Stakeholder.
- Roles that come with a licensed role, such as `sn_incident_read` inside `sn_incident_write`, no longer count as a separate license.
- Approvals only count within the lookback period. In 1.0.0 one old approval was enough.
- Records opened in any workspace or in the classic UI now count. 1.0.0 only recognised Service Operations Workspace URLs.
- Changes to comments, watch list and tags no longer make a user a Fulfiller.

To compare before and after, export the License Monitor list before you upgrade.

---

## Data model

### License Subscription (`x_delde_license_subscription`, prefix `LISUB`)

Each record is one license definition to monitor.

| Field | Type | Description |
|---|---|---|
| Name | String | Name of the license definition. |
| Role type | Reference → `license_role_type` | The license type being measured, for example *Fulfiller* or *Business Stakeholder*. |
| Application entitlement | List → `license_family` | The license families that make up the license. |
| Associated roles | String | Licensed roles, filled in from *Application entitlement* and *Role type*. Kept as they are if no roles are found in `license_role`. |
| Associated tables | List → `sys_db_object` | Tables whose activity counts as using the license. |
| Table names | String | Table names, filled in from *Associated tables*. |
| Subscription entitlement | Reference | Optional link to the platform subscription record. |
| Scan status | Choice | 100 Not Initiated · 200 Initiated · 300 In Progress · 400 Skipped · 500 Completed · 600 Error |
| Scan message | String | Result of the last scan (counts per classification) or the error that stopped it. |
| Last scanned | Date/time | When the last scan finished. |
| Active | True/False | Only active subscriptions are scanned by the scheduled job. |
| Description | HTML | The rules used to classify users. |

### License Monitor (`x_delde_license_license_monitor`, prefix `LMON`)

Each record is one user in one scan of one subscription.

| Field | Description |
|---|---|
| Subscription | The license subscription that was scanned. |
| User | The user who holds one or more of the associated roles. |
| Role type | The license type the user holds. |
| Acting as | How the user actually behaves: Fulfiller, Business Stakeholder or Requester. |
| Evidence | Why the user got that classification, for example *Updated incident INC0007002 owned by someone else*. |
| Scan run | Identifies the scan the record belongs to. |
| Latest | True for the records that count now. The dashboard shows only these. |
| Status | *Completed*, or *Skipped* when the user is counted under another subscription in the same module. |

### Scan History (`x_delde_license_scan_history`)

One record per subscription per scan, used for trends.

| Field | Description |
|---|---|
| Subscription | The license subscription that was scanned. |
| Scan run | The same run identifier as on the monitor records. |
| Scanned on | When the scan finished. |
| Users | Users in the scan. |
| Fulfiller, Business Stakeholder, Requester | Users counted in this subscription with that behaviour. |
| Counted under higher license | Users counted under another subscription in the same module. |
| Latest in week | True for the newest scan of the subscription in that week. Trend charts use only these, so extra scans in a week are not counted twice. |

The counts are updated if a later scan in the same module moves users between subscriptions.

---

## How a scan works

1. **Configure.** When you set *Application entitlement* and *Role type*, a business rule looks up the licensed roles in `license_role` and stores them in *Associated roles*. When you set *Associated tables*, another business rule stores the table names in *Table names*.
2. **Start.** A scan starts from the **Scan license** UI action or from the scheduled job (weekly, Monday 00:00, every active subscription). Both set *Scan status* to **Initiated (200)**. The scheduled job first resets scans that have been *In Progress* for too long.
3. **Find users.** `ScannerUtils.runScan()` sets **In Progress (300)** and finds active, interactive users who hold the associated roles, leaving out users with an excluded role (`admin` by default). A role does not count when the user also holds a licensed role that contains it: `sn_incident_read` that comes with `sn_incident_write` does not make a fulfiller a Business Stakeholder. Roles granted through groups count as normal.
4. **Classify.** Each user is checked in this order, within the lookback period, and the first match wins:
   - **Fulfiller:** changed (`sys_audit`) a record in the associated tables that belongs to someone else. A record belongs to the person in *Caller*, *Requested for*, *Opened for*, *Contact*, *Consumer* or *Requested by*, or to its creator when none of those is filled in. Changes to the fields in `x_delde_license.ignored_audit_fields` (comments, watch list and tags by default) are ignored.
   - **Business Stakeholder:** approved or rejected something, or opened a record in the associated tables that belongs to someone else (classic UI and any workspace URL).
   - **Requester:** none of the above.

   The Fulfiller check runs only for subscriptions at Fulfiller level. Business Stakeholder roles are read-only, and ACLs stop those users from doing fulfilling work, so their subscriptions classify users as Business Stakeholder or Requester. Set `x_delde_license.check_fulfiller_below_fulfiller` to true to run the check anyway, for example to test for gaps in custom ACLs.
5. **Store.** One monitor record per user is created with the evidence. Earlier results are kept as history (*Latest* = false) and deleted after the retention period.
6. **One classification per module.** See below.
7. **Finish.** The subscription gets **Completed (500)** and a summary in *Scan message*, and a Scan History record is written. If anything fails it gets **Error (600)** with the error.

### One classification per user per module

A user who holds a Fulfiller role almost always holds Business Stakeholder roles too: in ITSM, `sn_incident_write` contains `sn_incident_read`, and the same pattern exists for change, problem and request, and in CSM and IRM. Without a rule, the same person would be counted against both licenses, although they consume only the higher one.

Subscriptions that share at least one associated table are treated as the same module. Within a module each user is counted **once**:

- in the subscription with the **highest license type** they hold (Fulfiller, then Business Stakeholder or Approver, then Requester),
- with the **highest behaviour** seen in any of the module's subscriptions.

The user's records in the other subscriptions get *Status* = *Skipped*, *Latest* = false, and evidence such as *Counted under LISUB0001002 (higher license in the same module)*. A user who holds licenses in two different modules, with no shared tables, is still counted in both.

### System properties

| Property | Default | Meaning |
|---|---|---|
| `x_delde_license.lookback_days` | 30 | Days of activity the scanner looks at. |
| `x_delde_license.excluded_roles` | admin | Users with any of these roles are left out. |
| `x_delde_license.include_inherited_roles` | false | Also count roles that a user holds only because a licensed role they hold contains them. |
| `x_delde_license.check_fulfiller_below_fulfiller` | false | Also look for Fulfiller activity in subscriptions below Fulfiller level. Useful to test for ACL gaps. |
| `x_delde_license.ignored_audit_fields` | comments,watch_list,sys_tags | Audited fields that do not count as fulfilling work. |
| `x_delde_license.history_retention_days` | 365 | How long earlier monitor records are kept. 0 keeps them forever. Scan History is kept. |
| `x_delde_license.stuck_scan_hours` | 6 | After this many hours *In Progress*, the scheduled job sets the scan to *Error* so it can run again. |

---

## Configuring a license subscription

1. Go to **License Scanner → License Subscriptions → New**.
2. Fill in **Name**, **Role type**, **Application entitlement** (the license families) and **Associated tables**.
3. Save. Check that *Associated roles* and *Table names* have been filled in.
4. Tick **Active** and click **Scan license**. The result is shown in *Scan message* when the scan finishes.

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

- **Modules are found from shared tables.** Two subscriptions with no table in common are treated as separate modules, even if they cover the same license.
- **Approvals are not limited to the associated tables.** Any approval in the lookback period counts towards Business Stakeholder.
- **The Business Stakeholder checks for approvals and views have only been tested with little data.** The Fulfiller path has been verified on a test instance.
- **Cross-scope privileges** are included for the ITSM, CSM, IRM and Agile tables used on the test instance. Other modules need *runtime access tracking* set to *permissive*, or privileges added for their tables.
- **The example subscriptions hold sys_ids from the instance they were built on.** See the note under [Example subscriptions](#example-subscriptions).

---

## Roadmap

- Add ATF tests for the classification logic.
- Show the potential saving per subscription, based on the license price.
- Limit approvals to records in the associated tables.
- Add an explicit *Module* field as an alternative to grouping by shared tables.
