# P_AP_ANZ_003_056_JobTimesheetProcessing

# Job Timesheet Processing

# Process Technical Design Document (PTDD)

| Process ID | P_AP_ANZ_003_056_JobTimesheetProcessing |
| --- | --- |
| Business Unit | ANZ |
| Operation Unit | IT |
| Process Name | Job Timesheet Processing |

## Document Version Control

| Date Issued | Version | Description/Remark | Attachment | Author |
| --- | --- | --- | --- | --- |
| 2026-06-24 | 1.0 | Create the document: 1. Introduce the background of the process 2. Introduce the solution of the process 3. Introduce specific functions of the process | TKE RPA - Job Timesheet Processing Maintain PDD V1.0 (ITCM 1837430).pdf | DHC - Yang Jin-Yu |

## Contents

1. Background
2. Process Flowchart
3. Process Technical Design
   1. Process Analysis
   2. API Design
   3. Specific Design
      1. Business Impact Assessment
      2. Process Structure
      3. Process Workflows
      4. Process Activity
      5. Remind Mail
4. Assets and Constants
   1. Assets
   2. Constants
5. QA
6. Approvals

## 1. Background

The solution will automate the Job Timesheet Processing activities for ANZ. The robot downloads timesheet data from VIEW, prepares and validates the Excel upload file, uploads timesheet records to SAP, approves working times, transfers approved timesheets to cost objects, stores supporting files in SharePoint, and sends execution result emails.

The robot will:

1. Download the Timesheet Report from VIEW for the weekly processing period.
2. Create and maintain the working Excel file from the standard template.
3. Filter and clean CATS report data before SAP upload.
4. Upload valid timesheet lines to SAP with T-code `/TKET/FOT_UPLOADCATS`.
5. Move invalid, zero-hour, or unresolved rows to the Unallocated tab.
6. Save uploaded and unallocated data to SharePoint.
7. Approve working times in SAP with T-code `CATS_APPR_LITE`.
8. Transfer approved timesheets to cost objects with T-code `CATA`.
9. Run and export the `COFC` report.
10. Generate the execution summary report.
11. Send success, business exception, or system exception remind mails.

## 2. Process Flowchart

Start -> Initialize process assets and local folders -> Check current execution step -> Login to VIEW -> Download weekly Timesheet Report -> Build working workbook -> Validate and filter CATS data -> Login to SAP -> Upload CATS data by test run -> Resolve known upload errors -> Execute live upload -> Save output files to SharePoint -> Approve working times -> Transfer to cost object -> Export COFC report -> Generate execution summary report -> Send remind mail -> End.

Exception path: Business exception -> record item details in execution summary report -> continue with the next transaction item or process stage where applicable -> send business exception mail at completion. System exception -> retry the failed item or stage 2 times with a 1 minute interval -> record details in execution summary report -> send system exception mail when unresolved.

## 3. Process Technical Design

### 3.1 Process Analysis

| Area | Design |
| --- | --- |
| Trigger | Scheduled at 7:00 AM AEST each Thursday. |
| Primary transaction | Each timesheet line in the CATS report is treated as one transaction item for validation, upload status, exception logging, and final reporting. |
| Batch stages | VIEW download, workbook preparation, SAP upload, SharePoint save, SAP approval, SAP transfer, COFC report, and mail notification are tracked as process stages. |
| Target systems | VIEW, SAP, and SharePoint. |
| Processing window | Last Monday to last Sunday. |
| Countries | AU and NZ. |
| SAP company codes | AU = 2223, NZ = 2225. |
| SAP T-codes | `/TKET/FOT_UPLOADCATS`, `CATS_APPR_LITE`, `CATA`, and `COFC`. |
| Main output | `Job Timesheet Processing Execution Summary Report (yyyyMMdd).xlsx`. |

### 3.2 API Design

No custom API integration is required. The automation uses UI automation for VIEW and SAP, Excel activities for workbook processing, SharePoint file operations for data saving, and mail activities for notifications.

| Integration | Method | Purpose |
| --- | --- | --- |
| VIEW | Browser UI automation | Download weekly Timesheet Report. |
| SAP | SAP GUI automation | Upload CATS data, approve working times, transfer to cost object, and run COFC report. |
| Excel | Excel workbook automation | Prepare upload file, filter data, maintain uploaded and unallocated tabs, and generate reports. |
| SharePoint | File upload/download operation | Store uploaded data, unallocated data, and execution evidence. |
| Outlook/Exchange | Mail activity | Send success, business exception, and system exception notifications. |

### 3.3 Specific Design

#### 3.3.1 Business Impact Assessment

| Assessment Area | Question | Answer |
| --- | --- | --- |
| Confidentiality | What is the highest classification of the data processed? | Timesheet, employee/service resource, labor hour, network, activity, SAP upload, and cost object data. |
| Confidentiality | Does this process contain any personal data? | Y. Timesheet records can identify employees or service resources and their working hours. |
| Confidentiality | Does this process contain any sensitive data e.g., commercial, security? | Y. Labor hours, costing information, SAP cost object postings, and exception details are commercially sensitive. |
| Integrity | Does this process affect any external / operational services? | Y. The robot modifies SAP timesheet and cost object data and stores data files in SharePoint. |
| Availability | Are there any SLAs in place internally or externally? | Internally scheduled weekly execution at 7:00 AM AEST each Thursday. |
| Availability | Does this process have a defined Recovery Time Objective? | Retry 2 times per failed item or stage with a 1 minute interval; unresolved failures are reported to RPA support. |
| Availability | Does this process have a defined Recovery Point Objective? | No formal RPO. Source data can be re-downloaded from VIEW and execution outputs are saved to SharePoint after processing. |

#### 3.3.2 Process Structure

The architecture of process is REF (Robotic Enterprise Framework).

Set each timesheet line from the CATS report as one transaction item.

If the process has any business exceptions, the robot records the exception information in the execution summary report and log message. The process should not be interrupted and should proceed with the next transaction item or next eligible process stage.

If the process has any system exceptions, the robot records the execution information in the execution summary report and log message. The robot retries 2 times per failed transaction item or stage. The process should not be interrupted when the error can be isolated to a transaction item.

Define a DataTable to store the execution results of transaction items. The fields are:

| Field Name | Storage Data | Comment |
| --- | --- | --- |
| Process Name | `Job Timesheet Processing` | Mandatory |
| Country | AU or NZ | Mandatory |
| Company Code | AU = 2223, NZ = 2225 | Mandatory |
| Processing Period Start | Last Monday of the weekly period | Mandatory |
| Processing Period End | Last Sunday of the weekly period | Mandatory |
| Source System | VIEW | Mandatory |
| Source Report Name | Downloaded CATS report file name | Optional |
| Transaction Row No. | Row number from working workbook | Mandatory |
| Employee / Resource ID | Identifier from CATS report where available | Optional |
| Work Date | Date from CATS report where available | Optional |
| Network | Network value from upload data | Mandatory for upload; must be 10 digits when populated |
| Activity | Activity value from upload data | Mandatory for upload; must be 4 digits when populated |
| CATSHOURS | Hours from CATS report | Mandatory |
| ACTTYPE | Activity type used in SAP upload | Optional; auto-updated for known `CG5K322200` error |
| Upload Status | SAP upload result | Success, Warning, Failed |
| Approval Status | SAP approval result | Success, Warning, Failed, or N/A |
| Transfer Status | SAP transfer result | Success, Warning, Failed, or N/A |
| SharePoint Save Status | File save result | Success, Warning, Failed |
| Error Type | Business or system exception category | Optional |
| Execution Result | Robot generated | Success, Warning, Failed |
| Comment | Execution details and exception messages | Optional |

##### VIEW Download

1. Get VIEW URL and credentials from assets.
2. Login to VIEW (`https://au.ap.tkelevator.com` or `https://nz.ap.tkelevator.com`).
3. Navigate to Service Database -> Timesheet -> Timesheet Report.
4. Select `Report to CATS`.
5. Select last Monday to last Sunday as the time period.
6. Click `last update` to refresh cache.
7. Export Excel and open the report by selecting `Don't Convert` when prompted.
8. If no report data exists, record a business exception and send the configured mail.

##### Excel Data Processing

1. Create a new workbook by copying the standard template tabs: `AU raw data`, `AU timesheet working`, `Upload`, and `Unallocated`.
2. Paste the VIEW CATS report data to the raw data tab.
3. Copy data from raw data to the timesheet working tab.
4. Move rows with `CATSHOURS = 0` to `Unallocated`.
5. Delete rows where Column I/J and L/M both contain data.
6. Validate that Network is 10 digits.
7. Validate that Activity is 4 digits.
8. Correct reversed data where the business rule identifies reversed values.
9. When Column L has data, set Column M to `10`.
10. Filter blank Column I values and other incorrect items.
11. Move non-compliant data to `Unallocated`.
12. Sum `CATSHOURS` for reconciliation.
13. Copy finalized valid rows to the `Upload` tab.

##### SAP Upload

1. Login to SAP with the robot account.
2. Run T-code `/TKET/FOT_UPLOADCATS`.
3. Fill required upload parameters and select the upload file.
4. Execute Test Run.
5. Auto-handle expected popups, including `Retry` and `Don't Update`.
6. Filter the Message column.
7. Keep successfully processed messages such as `* Messages processed successfully` and `Record can be processed`.
8. For `CG5K322200`, auto-update `ACTTYPE` and retry.
9. For missing labor costing information, remove the row from upload and move it to `Unallocated`.
10. For other SAP upload errors, move the row to `Unallocated` where applicable and record the error type.
11. Loop test run until no resolvable errors remain.
12. Untick Test Run and execute Live Run.
13. Verify that all eligible rows were processed successfully.

##### SharePoint Save

1. Save uploaded data to SharePoint.
2. Save unallocated data to SharePoint.
3. Reflect hours posted, hours not posted, and error types in the execution report and remind mail.

##### SAP Approval

1. Run T-code `CATS_APPR_LITE`.
2. Fill Company Code: AU = `2223`, NZ = `2225`.
3. Execute report.
4. Compare uploaded hours with hours to approve.
5. If variance exceeds the acceptable threshold of 1-2 hours, log warning details in the execution summary report.
6. Record hours approved, hours uploaded, variance amount, and unapproved line details where applicable.
7. Select all eligible records and approve.
8. Send exception report if variance exists. The variance does not block the approval process.

##### SAP Transfer to Job

1. Run T-code `CATA`.
2. Calculate Posting date / Post. Date for cancel.
3. Use the last day of the processing week by default.
4. If the week crosses months, use the 1st day of the current month.
5. Execute Test Run.
6. Verify that `No errors were found` is shown.
7. Untick Test Run and execute Live Run.
8. Confirm `All data successfully transferred to CO`.

##### Final Reporting

1. Run the `COFC` report.
2. Export the report.
3. Generate `Job Timesheet Processing Execution Summary Report (yyyyMMdd).xlsx`.
4. Set execution status to `Success`, `Warning`, or `Failed`.

#### 3.3.3 Process Workflows

| Category | Workflow |
| --- | --- |
| Browser | `JobTimesheet_CloseBrowser.xaml` |
| Browser | `JobTimesheet_KillBrowser.xaml` |
| Browser | `JobTimesheet_DownloadTimesheetReport_VIEW.xaml` |
| Excel | `JobTimesheet_CloseExcel.xaml` |
| Excel | `JobTimesheet_KillExcel.xaml` |
| Excel | `JobTimesheet_CreateWorkingWorkbook.xaml` |
| Excel | `JobTimesheet_ProcessAndFilterTimesheetData.xaml` |
| Excel | `JobTimesheet_GenerateExecutionReport.xaml` |
| SAP | `JobTimesheet_LoginSAP.xaml` |
| SAP | `JobTimesheet_UploadTimesheetToSAP.xaml` |
| SAP | `JobTimesheet_ApproveWorkingTimes.xaml` |
| SAP | `JobTimesheet_PostToCostObject.xaml` |
| SAP | `JobTimesheet_RunCOFCReport.xaml` |
| SAP | `JobTimesheet_HandleSAPPopup.xaml` |
| SharePoint | `JobTimesheet_SaveDataToSharePoint.xaml` |
| Local Folder | `JobTimesheet_InitLocalFolder.xaml` |
| Email | `JobTimesheet_SendMail.xaml` |

#### 3.3.4 Process Activity

| Activity Area | UiPath Package / Component |
| --- | --- |
| Browser | `UiPath.UIAutomation.Activities` |
| SAP | `UiPath.UIAutomation.Activities` and SAP GUI scripting activities |
| Excel | `UiPath.Excel.Activities` |
| Folder | `UiPath.System.Activities` |
| SharePoint | Microsoft 365 / SharePoint file activities configured for the tenant |
| Mail | `UiPath.Mail.Activities` |
| DataTable | `UiPath.System.Activities` |

#### 3.3.5 Remind Mail

##### Success Remind Mail

The robot will send success remind mail to key users when the process executes completed without any exceptions.

| Field | Value |
| --- | --- |
| Subject | `[RPA-Success] Job Timesheet Processing - Execute Success` |
| To | Key users |
| Cc | RPA support team |
| Attachment | `Job Timesheet Processing Execution Summary Report (yyyyMMdd).xlsx` |

##### Business Exception Remind Mail

The robot will send business exception remind mail when the process executes with business exceptions, including no timesheet data, unresolved validation errors, unresolved SAP upload errors, or approval variance warnings.

| Field | Value |
| --- | --- |
| Subject | `[RPA-BusinessException] Job Timesheet Processing - Execute With Business Exception` |
| To | RPA support team |
| Cc | N/A |
| Attachment | `Job Timesheet Processing Execution Summary Report (yyyyMMdd).xlsx` when available |

##### System Exception Remind Mail

The robot will send system exception remind mail to RPA support team when the process executes with system exceptions such as element not found, VIEW/SAP login delay after retries, SAP GUI operation failure, SharePoint upload failure, or unhandled application popup.

| Field | Value |
| --- | --- |
| Subject | `[RPA-SystemException] Job Timesheet Processing - Execute With System Exception` |
| To | RPA support team |
| Cc | N/A |
| Attachment | `Job Timesheet Processing Execution Summary Report (yyyyMMdd).xlsx` when available |

## 4. Assets and Constants

### 4.1 Assets

| Asset | Type | Description |
| --- | --- | --- |
| `A_AP_ANZ_003_056_VIEW_AU_URL` | Text | VIEW production URL for AU: `https://au.ap.tkelevator.com` |
| `A_AP_ANZ_003_056_VIEW_NZ_URL` | Text | VIEW production URL for NZ: `https://nz.ap.tkelevator.com` |
| `A_AP_ANZ_003_056_VIEW_DEV_URL` | Text | VIEW DEV2 URL: `https://audev2.ap.tkelevator.com` |
| `A_AP_ANZ_003_056_VIEW_Credential` | Credential | VIEW robot credential for `S1002314@tkelevator.com` |
| `A_AP_ANZ_003_056_SAP_Credential` | Credential | SAP robot credential for `S1002314` |
| `A_AP_ANZ_003_056_SHAREPOINT_URL` | Text | SharePoint folder URL for uploaded, unallocated, and report files |
| `A_AP_ANZ_003_056_TEMPLATE_PATH` | Text | SharePoint or local path of the timesheet upload template |
| `A_AP_ANZ_003_056_KEY_USERS_MAIL_GROUP` | Text | `mae.ciriaco.external@tkelevator.com; kamal.dutt@tkelevator.com; renaud.ghysens@tkelevator.com` |
| `A_AP_ANZ_003_056_RPA_SUPPORT_MAIL_GROUP` | Text | RPA support team mail group |

### 4.2 Constants

| Constant | Value | Description |
| --- | --- | --- |
| `ProcessName` | `Job Timesheet Processing` | Process display name |
| `ProcessID` | `P_AP_ANZ_003_056_JobTimesheetProcessing` | Process ID |
| `Schedule` | `7:00 AM AEST each Thursday` | Planned trigger |
| `RetryCount` | `2` | Retry count for system exceptions |
| `RetryIntervalMinutes` | `1` | Retry interval for VIEW / SAP delays |
| `ProcessingPeriodStartRule` | `Last Monday` | Timesheet report start date |
| `ProcessingPeriodEndRule` | `Last Sunday` | Timesheet report end date |
| `AUCompanyCode` | `2223` | SAP company code for AU |
| `NZCompanyCode` | `2225` | SAP company code for NZ |
| `NetworkLength` | `10` | Network validation rule |
| `ActivityLength` | `4` | Activity validation rule |
| `ColumnMDefaultWhenColumnLHasValue` | `10` | Excel cleansing rule |
| `ApprovalVarianceThresholdHours` | `1-2` | Warning threshold for uploaded vs approval hours |
| `SAPUploadTCode` | `/TKET/FOT_UPLOADCATS` | SAP CATS upload T-code |
| `SAPApprovalTCode` | `CATS_APPR_LITE` | SAP working time approval T-code |
| `SAPTransferTCode` | `CATA` | SAP transfer to cost object T-code |
| `SAPCOFCReport` | `COFC` | Final SAP report |
| `ExecutionReportFileName` | `Job Timesheet Processing Execution Summary Report (yyyyMMdd).xlsx` | Final execution report file name |

## 5. QA

| No. | Question | Answer |
| --- | --- | --- |
| 1 | Does the design follow the existing City FM SDD structure? | Yes. It uses Background, Process Flowchart, Process Technical Design, Assets and Constants, QA, and Approvals. |
| 2 | Does the design map all PDD target systems? | Yes. VIEW, SharePoint, and SAP are included. |
| 3 | Does the design include PDD trigger information? | Yes. Weekly Thursday 7:00 AM AEST schedule is included. |
| 4 | Does the design include SAP T-codes from the PDD? | Yes. `/TKET/FOT_UPLOADCATS`, `CATS_APPR_LITE`, `CATA`, and `COFC` are included. |
| 5 | Does the design include validation and unallocated data handling rules? | Yes. Zero hours, Network, Activity, conflicting columns, reversed data, Column L/M, blank items, and SAP upload errors are included. |
| 6 | Does the design include business and system exception handling? | Yes. Business exception and system exception handling, retry count, and remind mails are included. |

## 6. Approvals

| Role | Name | Date |
| --- | --- | --- |
| RPA Process Owner | Kamal Dutt |  |
| RPA Service Owner | Santosh Adsul |  |
