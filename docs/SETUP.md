# Setup Guide

## Prerequisites

Before you begin, ensure you have:

- ✅ Make.com account (free tier) - [Sign up here](https://make.com)
- ✅ Airtable account (free tier) - [Sign up here](https://airtable.com)
- ✅ OpenAI API key - [Get API key](https://platform.openai.com/api-keys)
- ✅ Google account with Gmail and Google Sheets access
- ✅ Slack workspace (free tier) - [Create workspace](https://slack.com)

**Estimated setup time:** 45-60 minutes

---

## Step 1: Set Up Airtable (15 minutes)

### 1.1 Create Base

1. Log into Airtable
2. Click **"Create a base"** → **"Start from scratch"**
3. Name it: `Month-End Close Tracker`

### 1.2 Create Table Structure

Rename the default table to **"Close Tasks"** and add these fields:

| Field Name | Field Type | Configuration |
|------------|------------|---------------|
| Task ID | Single line text | Primary field |
| Task Name | Single line text | |
| Owner | Single select | Options: AP Team, Controller, FP&A, Treasury, CFO |
| Owner Email | Email | |
| Due Date | Date | Include time |
| Status | Single select | Options: Not Started, In Progress, Completed, Blocked, Overdue |
| Priority | Single select | Options: Critical, High, Medium |
| Depends On | Link to another record | Link to Close Tasks table |
| Estimated Hours | Number | 1 decimal place |
| Actual Hours | Number | 1 decimal place |
| Last Reminder Sent | Date | Include time |
| Reminder Count | Number | Integer |
| Completion Note | Long text | |
| Automation Type | Single select | Options: Manual, Auto-Reconciliation, Auto-Report |
| Data Source | Single line text | |
| Variance Threshold | Number | 2 decimal places |
| Created Time | Created time | |
| Completed Time | Last modified time | |

### 1.3 Load Sample Data

See [data-samples/airtable-tasks.csv](../data-samples/airtable-tasks.csv) for 15 sample tasks to import.

**Quick Import:**
1. Download the CSV file
2. In Airtable, click the dropdown next to your table name
3. Select "Import data" → "CSV file"
4. Upload the file

---

## Step 2: Set Up Google Sheets (10 minutes)

### 2.1 Create Bank Transaction Sheet

1. Go to [sheets.google.com](https://sheets.google.com)
2. Create new spreadsheet: `Bank Data - Operating Account`
3. Add these column headers in row 1:

   | A | B | C | D | E | F |
   |---|---|---|---|---|---|
   | Date | Description | Amount | Type | GL Account | Cleared |

4. Load sample data from [data-samples/bank-transactions.csv](../data-samples/bank-transactions.csv)

5. **Important:** Copy the Sheet ID from the URL:
   - Your URL looks like: `https://docs.google.com/spreadsheets/d/[SHEET_ID]/edit`
   - Copy the long string between `/d/` and `/edit`
   - Save this ID - you'll need it for Make.com

---

## Step 3: Set Up Slack (5 minutes)

### 3.1 Create Channel

1. In your Slack workspace, create a new channel: `#month-end-close`
2. This is where all automation notifications will be posted

### 3.2 Note Your Workspace

- Make note of your Slack workspace URL (e.g., `yourcompany.slack.com`)
- You'll connect this to Make.com in the next step

---

## Step 4: Set Up Make.com Scenario (25 minutes)

### 4.1 Create Make.com Account

1. Go to [make.com](https://make.com)
2. Sign up with Google (easiest)
3. Verify your email

### 4.2 Create New Scenario

1. Click **"Create a new scenario"**
2. Name it: `Month-End Close Master Controller`
3. Click Save

### 4.3 Connect Your Apps

**Connect Airtable:**
1. When adding an Airtable module, click "Add" connection
2. Choose OAuth 2.0
3. Authorize access to your Airtable account

**Connect Gmail:**
1. Add Gmail module
2. Sign in with your Google account
3. Grant permissions

**Connect Google Sheets:**
1. Same process as Gmail
2. Use the same Google account

**Connect Slack:**
1. Add Slack module
2. Authorize your workspace
3. Grant permissions

**Connect OpenAI:**
1. Add OpenAI module
2. Click "Add" connection
3. Paste your OpenAI API key
4. Name it "My OpenAI"

### 4.4 Build the Scenario

Follow the architecture shown in [ARCHITECTURE.md](ARCHITECTURE.md) to build all 4 routes.

**High-Level Steps:**
1. Add Airtable Search as first module
2. Add Router
3. Build Route 1 (Daily Report)
4. Build Route 2 (Email Parser)
5. Build Route 3 (Bank Reconciliation)
6. Build Route 4 (Final Report)

**Detailed build instructions:** See the main README or reach out with questions.

---

## Step 5: Test the Automation (10 minutes)

### 5.1 Test Route 1 (Daily Report)

1. Ensure T001 exists in your Airtable
2. Click "Run once" in Make.com
3. Check Slack for daily progress report

### 5.2 Test Route 2 (Email Parser)

1. Send test email to yourself:
   - Subject: `Task T001 completed`
   - Body: `I finished task T001`
2. Keep it unread
3. Click "Run once"
4. Verify T001 status updated in Airtable
5. Check Slack for notification

### 5.3 Test Route 3 (Bank Reconciliation)

1. Set T003 status to "In Progress" in Airtable
2. Click "Run once"
3. Check that T003 updates to "Completed"
4. Review reconciliation summary in Slack

### 5.4 Test Route 4 (Final Report)

1. Set all tasks (T001-T015) to "Completed"
2. Click "Run once"
3. Verify final celebration message in Slack

---

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues and solutions.

---

## Next Steps

Once your automation is working:
1. Schedule the scenario to run daily at 9:00 AM
2. Customize task list for your actual close process
3. Update email addresses to your team members
4. Adjust bank reconciliation expected balances

---

## Need Help?

- Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- Review [ARCHITECTURE.md](ARCHITECTURE.md) for technical details
- Open a GitHub issue with questions
