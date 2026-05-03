# Power Automate: Create SharePoint Lists from Excel

Step-by-step instructions for building a Power Automate flow that reads an Excel file and automatically creates SharePoint Online lists with typed columns.

---

## Contents

| File | Description |
|------|-------------|
| `ListColumnDefinitions_Template.csv` | Sample Excel input template (convert to `.xlsx` before use) |

---

## Part 1 – Prepare the Excel File

1. Open `ListColumnDefinitions_Template.csv` in Excel.
2. Go to **File → Save As**, choose **Excel Workbook (.xlsx)**, and save it.
3. Click anywhere in the data, then press **Ctrl + T** to create a Table. Make sure **My table has headers** is checked, then click **OK**.
4. With the table selected, go to **Table Design** (ribbon tab) and set **Table Name** to `ColumnDefinitions`.
5. Upload the file to **OneDrive** or a **SharePoint Document Library**.

### Excel Column Reference

| Column | Required | Description |
|---|---|---|
| `ListName` | ✅ | Name of the SharePoint list to create. All rows with the same value belong to the same list. |
| `ColumnName` | ✅ | Internal (static) name. Use letters, numbers, and underscores only — no spaces. |
| `ColumnType` | ✅ | One of: `Text`, `Number`, `DateTime`, `Boolean`, `Choice`, `MultiChoice`, `Note`, `Currency`, `URL` |
| `ColumnDisplayName` | ✅ | The label shown in the SharePoint list view. |
| `Required` | ✅ | `Yes` or `No`. |
| `Choices` | ⬜ | Comma-separated values for `Choice` / `MultiChoice` columns. Leave blank for all other types. |

> **Tip:** Add a row with `ColumnName = Title` as the **first row** for each list. The flow skips creating it (SharePoint lists already have a Title field), but it acts as a clear visual separator in the Excel file.

---

## Part 2 – Create the Flow in Power Automate

### Step 1 – Create a new Instant cloud flow

1. Go to [make.powerautomate.com](https://make.powerautomate.com) and sign in.
2. Click **+ Create** in the left navigation, then choose **Instant cloud flow**.
3. Give the flow a name, e.g. `Create SharePoint Lists from Excel`.
4. Select **Manually trigger a flow** and click **Create**.

---

### Step 2 – Add trigger inputs

1. Click the **Manually trigger a flow** trigger card to expand it.
2. Click **+ Add an input** three times to add the following inputs:

| Input type | Name | Description |
|---|---|---|
| Text | `SharePointSiteURL` | Full URL of the target site, e.g. `https://contoso.sharepoint.com/sites/MySite` |
| Text | `ExcelFilePath` | OneDrive path to the Excel file, e.g. `/Documents/ListColumnDefinitions.xlsx` |
| Text | `ExcelTableName` | Name of the Excel table, e.g. `ColumnDefinitions` |

---

### Step 3 – Initialize variables

Click **+ New step** and add each of the following **Initialize variable** actions (search for *"Initialize variable"*):

| # | Variable Name | Type | Initial Value |
|---|---|---|---|
| 1 | `CurrentListName` | String | *(leave blank)* |
| 2 | `ListID` | String | *(leave blank)* |
| 3 | `ErrorLog` | Array | *(leave blank)* |

> Add them one after another. Each one appears as a separate action card.

---

### Step 4 – Get Excel rows

1. Click **+ New step**.
2. Search for **Excel Online (Business)** and select the action **List rows present in a table**.
3. Fill in the fields:
   - **Location**: `OneDrive for Business`
   - **Document Library**: `OneDrive`
   - **File**: click the folder icon and browse to your `.xlsx` file, or type the path from the trigger: use the expression `triggerBody()['text_2']` *(or click the lightning bolt and select the **ExcelFilePath** input)*.
   - **Table**: select your table name, or use the expression for the **ExcelTableName** trigger input.

---

### Step 5 – Apply to each row (sequential loop)

1. Click **+ New step**, search for **Apply to each**, and select it.
2. In the **Select an output from previous steps** field, click the expression editor and enter:

   ```
   body('List_rows_present_in_a_table')?['value']
   ```

3. **Important – disable parallel execution:**
   Click the `…` (ellipsis) menu on the **Apply to each** card → **Settings** → set **Degree of Parallelism** to `1` → click **Done**.
   This ensures rows are processed one at a time so list-name change detection works correctly.

---

### Step 6 – Compose row field values

Inside the **Apply to each** loop, add the following **Compose** actions (search for *"Compose"*). These extract and normalize each field from the current row.

| Action name | Inputs expression |
|---|---|
| `Compose_ListName` | `trim(items('Apply_to_each')?['ListName'])` |
| `Compose_ColumnName` | `trim(items('Apply_to_each')?['ColumnName'])` |
| `Compose_ColumnType` | `toLower(trim(items('Apply_to_each')?['ColumnType']))` |
| `Compose_ColumnDisplayName` | `trim(items('Apply_to_each')?['ColumnDisplayName'])` |
| `Compose_Required` | `toLower(trim(items('Apply_to_each')?['Required']))` |
| `Compose_Choices` | `trim(items('Apply_to_each')?['Choices'])` |

---

### Step 7 – Detect a new list (Condition)

1. Inside the loop, add a **Condition** action.
2. Configure it to check whether the current list name is different from the tracked one:
   - **Left value**: `@outputs('Compose_ListName')`
   - **Operator**: `is not equal to`
   - **Right value**: `@variables('CurrentListName')`

#### If YES branch – Create the SharePoint list

Inside the **Yes** branch, add these actions in order:

**a) Send an HTTP request to SharePoint** (search for *"Send an HTTP request to SharePoint"*):

| Field | Value |
|---|---|
| Site Address | `@{triggerBody()['text']}` *(the SharePointSiteURL input)* |
| Method | `POST` |
| Uri | `_api/web/lists` |
| Headers | `Content-Type`: `application/json;odata=verbose` and `Accept`: `application/json;odata=verbose` |
| Body | (see below) |

Body:
```json
{
  "__metadata": { "type": "SP.List" },
  "BaseTemplate": 100,
  "Title": "@{outputs('Compose_ListName')}"
}
```

**b) Set variable – `ListID`**

- **Name**: `ListID`
- **Value**: `@{body('Send_an_HTTP_request_to_SharePoint')?['d']?['Id']}`

**c) Set variable – `CurrentListName`**

- **Name**: `CurrentListName`
- **Value**: `@{outputs('Compose_ListName')}`

**d) Append to array variable – `ErrorLog`**

- **Name**: `ErrorLog`
- **Value**: `✅ Created list: @{outputs('Compose_ListName')}`

---

### Step 8 – Skip the Title column (Condition)

After the list-detection Condition (still inside the main loop), add another **Condition**:

- **Left value**: `@toLower(outputs('Compose_ColumnName'))`
- **Operator**: `is equal to`
- **Right value**: `title`

Leave the **Yes** branch empty (do nothing — SharePoint already has a Title field on every list).

---

### Step 9 – Add columns by type (Switch)

Inside the **No** branch of the Title-skip condition, add a **Switch** action:

1. Search for **Switch** and add it.
2. Set the **On** expression to: `@outputs('Compose_ColumnType')`

Add a **Case** for each supported column type below. In each case, add a **Send an HTTP request to SharePoint** action with the body shown.

> **Common fields for every HTTP request action:**
> - **Site Address**: `@{triggerBody()['text']}`
> - **Method**: `POST`
> - **Uri**: `_api/web/lists(guid'@{variables('ListID')}')/fields`
> - **Headers**: `Content-Type: application/json;odata=verbose`, `Accept: application/json;odata=verbose`

---

#### Case: `text`

```json
{
  "__metadata": { "type": "SP.FieldText" },
  "FieldTypeKind": 2,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)}
}
```

---

#### Case: `number`

```json
{
  "__metadata": { "type": "SP.FieldNumber" },
  "FieldTypeKind": 9,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)}
}
```

---

#### Case: `datetime`

```json
{
  "__metadata": { "type": "SP.FieldDateTime" },
  "FieldTypeKind": 4,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)}
}
```

---

#### Case: `boolean`

```json
{
  "__metadata": { "type": "SP.FieldBoolean" },
  "FieldTypeKind": 8,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)}
}
```

---

#### Case: `choice`

Before the HTTP request, add a **Compose** action inside this case to split the choices:

- **Name**: `Compose_ChoicesArray`
- **Inputs**: `@split(outputs('Compose_Choices'), ',')`

Then add the HTTP request:

```json
{
  "__metadata": { "type": "SP.FieldChoice" },
  "FieldTypeKind": 6,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)},
  "Choices": { "results": @{outputs('Compose_ChoicesArray')} }
}
```

---

#### Case: `multichoice`

Same structure as `choice` but use `FieldTypeKind: 15` and `"type": "SP.FieldMultiChoice"`:

- Compose action: `@split(outputs('Compose_Choices'), ',')`

```json
{
  "__metadata": { "type": "SP.FieldMultiChoice" },
  "FieldTypeKind": 15,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)},
  "Choices": { "results": @{outputs('Compose_ChoicesArray')} }
}
```

---

#### Case: `note`

```json
{
  "__metadata": { "type": "SP.FieldMultiLineText" },
  "FieldTypeKind": 3,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)}
}
```

---

#### Case: `currency`

```json
{
  "__metadata": { "type": "SP.FieldCurrency" },
  "FieldTypeKind": 10,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)}
}
```

---

#### Case: `url`

```json
{
  "__metadata": { "type": "SP.FieldUrl" },
  "FieldTypeKind": 11,
  "Title": "@{outputs('Compose_ColumnDisplayName')}",
  "StaticName": "@{outputs('Compose_ColumnName')}",
  "Required": @{if(equals(outputs('Compose_Required'),'yes'),true,false)}
}
```

---

#### Default case (unknown type)

In the **Default** case of the Switch, add an **Append to array variable** action:

- **Name**: `ErrorLog`
- **Value**: `⚠️ Unknown type '@{outputs('Compose_ColumnType')}' for column '@{outputs('Compose_ColumnName')}' in list '@{outputs('Compose_ListName')}' — skipped.`

---

### Step 10 – Save and test the flow

1. Click **Save** in the top-right corner.
2. Click **Test** → select **Manually** → click **Test**.
3. Fill in the three trigger inputs:
   - **SharePointSiteURL**: `https://contoso.sharepoint.com/sites/MySite`
   - **ExcelFilePath**: `/Documents/ListColumnDefinitions.xlsx`
   - **ExcelTableName**: `ColumnDefinitions`
4. Click **Run flow**.
5. Watch the run progress. Green ticks indicate success. Click any failed action to see the error details.

---

## Complete Flow Structure (reference)

```
Manually trigger a flow
  │  inputs: SharePointSiteURL, ExcelFilePath, ExcelTableName
  │
  ├─ Initialize variable: CurrentListName (String, blank)
  ├─ Initialize variable: ListID (String, blank)
  ├─ Initialize variable: ErrorLog (Array, blank)
  │
  ├─ List rows present in a table  [Excel Online (Business)]
  │
  └─ Apply to each (sequential, concurrency = 1)
       │  items: body('List_rows_present_in_a_table')?['value']
       │
       ├─ Compose: Compose_ListName
       ├─ Compose: Compose_ColumnName
       ├─ Compose: Compose_ColumnType
       ├─ Compose: Compose_ColumnDisplayName
       ├─ Compose: Compose_Required
       ├─ Compose: Compose_Choices
       │
       ├─ Condition: ListName != CurrentListName
       │    └─ YES:
       │         ├─ Send HTTP to SharePoint → POST _api/web/lists
       │         ├─ Set variable: ListID
       │         ├─ Set variable: CurrentListName
       │         └─ Append to ErrorLog: "✅ Created list: …"
       │
       ├─ Condition: ColumnName == "title"
       │    ├─ YES: (do nothing)
       │    └─ NO:
       │         └─ Switch on ColumnType
       │              ├─ text     → POST field (FieldTypeKind: 2)
       │              ├─ number   → POST field (FieldTypeKind: 9)
       │              ├─ datetime → POST field (FieldTypeKind: 4)
       │              ├─ boolean  → POST field (FieldTypeKind: 8)
       │              ├─ choice   → POST field (FieldTypeKind: 6)
       │              ├─ multichoice → POST field (FieldTypeKind: 15)
       │              ├─ note     → POST field (FieldTypeKind: 3)
       │              ├─ currency → POST field (FieldTypeKind: 10)
       │              ├─ url      → POST field (FieldTypeKind: 11)
       │              └─ default  → Append to ErrorLog: "⚠️ Unknown type …"
```

---

## Permissions Required

The account used in the **Send an HTTP request to SharePoint** action must have one of:
- **Site Collection Administrator**
- **Full Control** on the site
- **Manage Lists** permission at minimum

---

## Throttling Tips

If you are creating many lists or columns, add a **Delay** action (1–2 seconds) after each HTTP request to SharePoint to avoid hitting API rate limits.

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| *"Resource not found"* on Excel step | Verify the OneDrive file path and table name match exactly. |
| *"Access denied"* on SharePoint HTTP request | Ensure the connected account has Manage Lists permission. |
| List created but columns are missing | Check the ErrorLog variable output for unknown ColumnType warnings. |
| *"The list already exists"* | Remove or rename the existing list in SharePoint before re-running. |
| Choices contain extra spaces | Trim spaces from the Choices cell in Excel, or they will appear in the SharePoint dropdown options. |
| Run fails mid-way | Check the run history → click the failed action → expand **Outputs** to read the SharePoint error message. |
