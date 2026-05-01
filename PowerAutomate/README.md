# Power Automate: Create SharePoint Lists from Excel

This folder contains a Power Automate flow that reads an Excel file containing column definitions and automatically creates SharePoint Online lists with the specified columns.

---

## Contents

| File | Description |
|------|-------------|
| `CreateSharePointListsFromExcel.json` | Importable Power Automate flow definition |
| `ListColumnDefinitions_Template.csv` | Sample Excel input template (convert to `.xlsx` before use) |

---

## Excel File Setup

1. Convert `ListColumnDefinitions_Template.csv` to an `.xlsx` file (open in Excel, **Save As → Excel Workbook**).
2. Select all data including the header row and press **Ctrl+T** to format it as a **Table**.
3. Name the table (e.g. `ColumnDefinitions`) via **Table Design → Table Name**.
4. Upload the file to **OneDrive** or a **SharePoint Document Library**.

### Column Reference

| Column | Required | Description |
|---|---|---|
| `ListName` | ✅ | Name of the SharePoint list to create. All rows with the same value are grouped into one list. |
| `ColumnName` | ✅ | Internal (static) name of the column. Use alphanumeric characters and underscores only. |
| `ColumnType` | ✅ | One of: `Text`, `Number`, `DateTime`, `Boolean`, `Choice`, `MultiChoice`, `Note`, `Currency`, `URL` |
| `ColumnDisplayName` | ✅ | The label shown in the SharePoint list view. |
| `Required` | ✅ | `Yes` or `No` — whether the field is mandatory. |
| `Choices` | ⬜ | Comma-separated values for `Choice` / `MultiChoice` columns. Leave blank for other types. |

> **Important:** Always include a row with `ColumnName = Title` as the first row for each list. The flow automatically skips creating a duplicate Title field (SharePoint lists already have one), but it serves as a clear marker for each new list group.

---

## Importing the Flow

1. Go to [Power Automate](https://make.powerautomate.com) → **My flows** → **Import** → **Import Package (Legacy)**.
2. Upload `CreateSharePointListsFromExcel.json`.
   > If the import option requires a `.zip`, wrap the JSON in a zip: `zip CreateSharePointListsFromExcel.zip CreateSharePointListsFromExcel.json`
3. During import, configure the two **connections**:
   - **Excel Online (Business)** — sign in with your Microsoft 365 account.
   - **SharePoint** — sign in with an account that has at least **Manage Lists** permission on the target site.
4. Click **Import**.

---

## Running the Flow

The flow uses a **manual trigger** with three inputs:

| Input | Example |
|---|---|
| SharePoint Site URL | `https://contoso.sharepoint.com/sites/MySite` |
| Excel File Location (OneDrive path) | `/Documents/ListColumnDefinitions.xlsx` |
| Excel Table Name | `ColumnDefinitions` |

1. Open the flow → click **Run**.
2. Fill in the three input fields and click **Run flow**.
3. Monitor the run in **Flow run history** to check for any errors.

---

## How the Flow Works

```
Trigger (Manual)
  │
  ├─ Initialize variables: CurrentListName, ListID, ErrorLog
  │
  ├─ List rows from Excel table
  │
  └─ Apply to each row (sequential):
       │
       ├─ Compose row values (ListName, ColumnName, ColumnType, …)
       │
       ├─ Condition: Is ListName different from CurrentListName?
       │    └─ Yes → Create SharePoint list → Set ListID & CurrentListName
       │
       ├─ Condition: Is ColumnName = "Title"?
       │    └─ Yes → Skip (Title field already exists on every list)
       │
       └─ Switch on ColumnType:
            Text / Number / DateTime / Boolean / Choice /
            MultiChoice / Note / Currency / URL
              └─ POST field to SharePoint list via connector
```

### Supported Column Types

| Excel value | SharePoint type | `FieldTypeKind` |
|---|---|---|
| `Text` | Single line of text | 2 |
| `Number` | Number | 9 |
| `DateTime` | Date and Time | 4 |
| `Boolean` | Yes/No | 8 |
| `Choice` | Choice (single) | 6 |
| `MultiChoice` | Choice (multi-select) | 15 |
| `Note` | Multiple lines of text | 3 |
| `Currency` | Currency | 10 |
| `URL` | Hyperlink or Picture | 11 |

---

## Error Handling

- An **ErrorLog** array variable collects warnings for unknown column types and confirmation messages for each created list.
- The flow processes rows **sequentially** (`concurrency: 1`) to avoid race conditions when detecting list-name changes.
- If the SharePoint connector action fails (e.g. list already exists), the run will fail at that step. Check the run history for the failed action and its error message.

---

## Throttling / Large Files

SharePoint Online has API rate limits. If you are creating many lists or columns:

1. Add a **Delay** action (1–2 seconds) after each field-creation action.
2. Consider splitting the Excel file by list and running the flow multiple times.

---

## Permissions Required

The SharePoint connection account must have one of the following:
- **Site Collection Administrator**
- **Full Control** on the site
- **Manage Lists** permission at minimum (to create lists and fields)

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| *"Resource not found"* on Excel step | Verify the OneDrive file path and table name are correct. |
| *"Access denied"* on SharePoint step | Ensure the connected account has Manage Lists permission. |
| List created but columns missing | Check the ErrorLog output for unknown ColumnType values. |
| Duplicate list error | Remove or rename the existing list in SharePoint before re-running. |
| Choices column has extra spaces | Trim spaces in the Excel Choices cell, or they will appear in SharePoint options. |
