# OpenIDM 7.1.2 Guide: Role-Based Provisioning & De-Provisioning (CSV Connector)

This guide provides step-by-step instructions for configuring **Role-Based Provisioning and De-Provisioning** in **OpenIdentityPlatform OpenIDM 7.1.2** using the **OpenICF CSV File Connector**, a **Dynamic Role (`Engineering_Access`)**, and a **`validSource` qualification filter** (`conf/sync.json`).

---

## Architecture Overview

1. **Target System (`provisioned_users.csv`):** An external CSV file managed through the OpenICF CSV File Connector (`HrCsv`).
2. **Dynamic Role (`Engineering_Access`):** Automatically evaluates user profiles (`department eq "Engineering"`) and grants the `Engineering Member` assignment.
3. **Outbound Mapping (`managedUser_systemHrcsv__ACCOUNT__`):** Maps `managed/user` attributes to the target CSV schema, correlates records by `__NAME__` (`uid`), and uses `validSource` (`source.department === 'Engineering'`) to provision new joiners and automatically delete rows when users leave the `Engineering` department.

---

## Step 1: Prepare the Target CSV File

Create the target CSV file on disk with the exact column headers expected by the connector, followed by a blank newline so records write onto separate lines.

1. Create the directory `C:\u01\app\openidm\data\` (if it does not already exist).
2. Create **`C:\u01\app\openidm\data\provisioned_users.csv`** with the following header row and a trailing newline:

```csv
uid,firstName,lastName,email,department,role

```

> **Note:** Omitting the `password` column from the header prevents an empty column (`,,`) from appearing in exported CSV rows when user passwords are not mapped.

---

## Step 2: Configure the Outbound CSV Connector (`HrCsv`)

1. Log in to the OpenIDM Admin UI (`http://localhost:8080/admin`) with administrator credentials (`openidm-admin` / `openidm-admin`).
2. Navigate to **Configure** → **Connectors** → **+ New Connector**.
3. Configure the connector properties:
   * **Connector Name:** `HrCsv`
   * **Enabled:** Toggle **ON**
   * **Connector Type:** **CSV File Connector - 2.0.4** (`org.forgerock.openicf.connectors.csvfile.CSVFileConnector`)
   * **csvFile:** `C:/u01/app/openidm/data/provisioned_users.csv`
   * **headerUid:** `uid`
   * **headerPassword:** *(leave blank)*
   * **quoteCharacter:** `"`
   * **fieldDelimiter:** `,`
   * **syncFileRetentionCount:** `3`
4. Click **Save** to generate `conf/provisioner.openicf-hrCsv.json`.

### Set `newlineString` in `conf/provisioner.openicf-hrCsv.json`
The Admin UI strips escape sequences such as `\n` on save. Set `newlineString` directly in the JSON configuration file so each provisioned user writes to a separate line:

1. Open **`C:\u01\app\openidm\conf\provisioner.openicf-hrCsv.json`**.
2. Ensure `"enabled" : true` is set and add `"newlineString" : "\n"` inside `"configurationProperties"`:

```json
{
    "enabled" : true,
    "configurationProperties" : {
        "csvFile" : "C:/u01/app/openidm/data/provisioned_users.csv",
        "fieldDelimiter" : ",",
        "headerPassword" : "",
        "headerUid" : "uid",
        "newlineString" : "\n",
        "quoteCharacter" : """,
        "syncFileRetentionCount" : 3
    }
}
```

3. Save the file and verify under **Configure** → **Connectors** that **`HrCsv`** displays a green **Active** status.

---

## Step 3: Create the Outbound Mapping (`managedUser_systemHrcsv__ACCOUNT__`)

1. In the Admin UI, navigate to **Configure** → **Mappings** → **+ New Mapping**.
2. Select the source and target objects:
   * **Source:** **Managed Objects** → **`user`** (`managed/user`)
   * **Target:** **Connectors** → **`HrCsv`** → **`__ACCOUNT__`** (`system/hrCsv/__ACCOUNT__`)
   * **Mapping Name:** `managedUser_systemHrcsv__ACCOUNT__`
3. Click **Create Mapping**.

### Step 3.1: Configure Property Mappings
On the **Properties** tab, map the `managed/user` attributes to the CSV columns:

| Source (`managed/user`) | Target (`system/hrCsv/__ACCOUNT__`) | Purpose |
| :--- | :--- | :--- |
| **`userName`** | **`__NAME__`** | Maps username to the CSV `uid` column |
| **`givenName`** | **`firstName`** | Maps user's first name |
| **`sn`** | **`lastName`** | Maps user's last name |
| **`mail`** | **`email`** | Maps user's email address |
| **`department`** | **`department`** | Maps custom `department` attribute |

*(Leave `role` unmapped here; it is populated dynamically by the Role Assignment in Step 4).*

### Step 3.2: Configure Correlation Query (`__NAME__`)
1. Open the **Association** tab and click **+ Add Correlation Query**.
2. Select **Link Qualifier:** `default` and **Expression Builder**.
3. Add **`__NAME__`** (matching the target attribute mapped to `userName`—do not use `__UID__` if `userName` is mapped to `__NAME__`, as an unmapped `__UID__` evaluates to an empty string and throws `Uid value must not be blank!`).
4. Click **Submit**, then click **Save and Don't Reconcile**.

---

## Step 4: Configure `validSource` & Policies for Dynamic De-Provisioning

To ensure **only** users in the `Engineering` department are provisioned—and that users transferred out of `Engineering` are automatically deleted from `provisioned_users.csv`—configure `validSource` and sync policies in `conf/sync.json`.

1. Open **`C:\u01\app\openidm\conf\sync.json`**.
2. Locate the `"managedUser_systemHrcsv__ACCOUNT__"` mapping object and add the **`"validSource"`** script condition:

```json
        "validSource" : {
            "type" : "text/javascript",
            "source" : "source.department === 'Engineering'"
        },
```

3. Verify the key **`"policies"`** inside `"managedUser_systemHrcsv__ACCOUNT__"` (or set **Current Policy** to **Default** on the **Behaviors** tab):
   * **`ABSENT` → `CREATE`:** Provisions new `Engineering` users into the CSV file.
   * **`CONFIRMED` / `FOUND` → `UPDATE`:** Updates existing `Engineering` user attributes in the CSV file.
   * **`UNQUALIFIED` → `DELETE`:** Automatically removes the CSV row when a user fails `validSource` (e.g., `department` changes from `Engineering` to `Marketing`).
   * **`UNASSIGNED` → `IGNORE`:** Prevents creating CSV rows for unqualified users who lack a target link.
4. Save **`conf/sync.json`**.

---

## Step 5: Create the Dynamic Role & Assignment

### Step 5.1: Create the Role Assignment (`Engineering Member`)
1. Navigate to **Manage** → **Assignments** → **+ New Assignment**.
2. Configure the assignment:
   * **Assignment Name:** `Engineering Member`
   * **Mapping:** `managedUser_systemHrcsv__ACCOUNT__`
3. Click **Save**.
4. Open the **Attributes** tab on the assignment, click **+ Add Attribute**, select **`role`**, set its value to **`Engineering Member`**, and click **Save**.

### Step 5.2: Create the Dynamic Role (`Engineering_Access`)
1. Navigate to **Manage** → **Role** → **+ New Role**.
2. Enter **Name:** `Engineering_Access` and **Description:** `Automatic CSV provisioning for Engineering department`.
3. Open the **Condition** tab:
   * Toggle **Enable conditional assignment** **ON**.
   * Set the filter rule to: **`department`** **`equals`** **`Engineering`** (`/department eq "Engineering"`).
4. Click **Save**.
5. Open the **Managed Assignments** tab on `Engineering_Access`, click **+ Add Assignment**, select **`Engineering Member`**, and click **Save**.

---

## Step 6: Verification & Lifecycle Testing

### 1. Test Joiner (Provisioning)
1. Create or register a user with **Department** set to **`Engineering`** (e.g., `abhishek9195` and `james`).
2. Open **`C:\u01\app\openidm\data\provisioned_users.csv`** and confirm each qualifying user appears on a separate line with `"Engineering Member"` in the `role` column:

```csv
"uid","firstName","lastName","email","department","role"
"abhishek9195","Abhishek","Kumar","abhishek.km2010@gmail.com","Engineering","Engineering Member"
"james","James","Smith","james.smith@openidm.com","Engineering","Engineering Member"
```

### 2. Test Mover (Dynamic De-Provisioning)
1. In the Admin UI, navigate to **Manage** → **User** → select **`james`**.
2. Change **Department** from **`Engineering`** to **`Marketing`** and click **Save**.
3. Open **`C:\u01\app\openidm\data\provisioned_users.csv`**—OpenIDM automatically revokes `Engineering_Access`, evaluates `validSource` as `false` (`UNQUALIFIED`), and deletes `james`'s row while keeping `abhishek9195` intact:

```csv
"uid","firstName","lastName","email","department","role"
"abhishek9195","Abhishek","Kumar","abhishek.km2010@gmail.com","Engineering","Engineering Member"
```

---

## Troubleshooting Reference

| Error / Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`IllegalArgumentException: Uid value must not be blank!`** | Correlation query searches `__UID__` while `userName` is mapped to `__NAME__`. | Update `"expressionTree"` in `conf/sync.json` to match **`"__NAME__"`**. |
| **Multiple CSV rows merge onto one line** | `newlineString` is empty or stripped by the Admin UI form. | Set `"newlineString" : "\n"` directly in `conf/provisioner.openicf-hrCsv.json`. |
| **User row updates instead of deleting when department changes** | Mapping lacks a `validSource` condition to mark non-Engineering users `UNQUALIFIED`. | Add `"validSource" : { "type": "text/javascript", "source": "source.department === 'Engineering'" }` in `conf/sync.json`. |
| **`Cannot reconcile from an empty data source`** | `managed/user` has zero records when running outbound reconciliation. | Create or register at least one user in **Manage → User** before clicking **Reconcile**. |
| **`SQLIntegrityConstraintViolationException: Duplicate entry` on `links`** | Stale link records remain in MySQL after manually deleting rows from the CSV file. | Delete the stale rows from `openidm.links` (or via `DELETE /openidm/repo/links/<id>` with exact `If-Match: <rev>`) before reconciling. |
