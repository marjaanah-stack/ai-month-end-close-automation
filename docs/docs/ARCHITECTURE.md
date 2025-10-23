# Technical Architecture

## System Overview

This document provides detailed technical information about the AI-Powered Month-End Close Orchestrator's architecture, data flows, and implementation details.

---

## Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                        Make.com Orchestrator                     │
│                    Scheduled Execution Engine                    │
│                     (Daily at 9:00 AM GMT)                       │
└─────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │   Module 2: Airtable      │
                    │   Check if Close Active   │
                    │   Formula: {Task ID}="T001"│
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │     Module 4: Router      │
                    │   Splits into 4 Paths     │
                    └─────────────┬─────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────┬──────────────────┐
        │                         │                     │                  │
   ┌────▼────┐               ┌────▼────┐           ┌───▼────┐       ┌─────▼─────┐
   │ Route 1 │               │ Route 2 │           │ Route 3│       │  Route 4  │
   │  Daily  │               │  Email  │           │  Bank  │       │   Final   │
   │ Report  │               │ Parser  │           │  Rec   │       │  Report   │
   └────┬────┘               └────┬────┘           └───┬────┘       └─────┬─────┘
        │                         │                    │                   │
        ▼                         ▼                    ▼                   ▼
   [Details Below]           [Details Below]      [Details Below]    [Details Below]
```

---

## Route 1: Daily Progress Report

### Data Flow
```
Airtable Search [5]
  → Get all Close Tasks (15 records)
    → Array Aggregator [14]
      → Collect all task data
        → Set Variable [15]
          → Calculate: total_tasks = 15
            → OpenAI [8]
              → Prompt: Analyze progress + generate summary
                → Slack [9]
                  → Post to #month-end-close
```

### Module Details

| Module # | Type | Configuration | Purpose |
|----------|------|---------------|---------|
| 5 | Airtable Search | Formula: (empty) | Retrieve all 15 tasks |
| 14 | Array Aggregator | Source: Module 5 | Combine into single bundle |
| 15 | Set Variable | total_tasks = 15 | Store count |
| 8 | OpenAI GPT-4o | Temperature: 0.6 | Generate AI summary |
| 9 | Slack Message | Channel: #month-end-close | Notify team |

### AI Prompt Structure
```
Analyze this month-end close progress and create an executive summary.

Metrics:
- Total Tasks: {{15.total_tasks}}
- Completed: [calculated]
- In Progress: [calculated]
- Not Started: [calculated]
- Blocked: [calculated]
- Overdue: [calculated]
- Completion: [percentage]%

Provide:
1. One-sentence status: Are we on track?
2. Key risks (overdue/blocked items)
3. Action items for today
4. Predicted completion date based on current velocity

Be concise, data-driven, and actionable. Use professional CFO language.
```

### Error Handling

- **No tasks found:** Filter blocks execution (expected if T001 doesn't exist)
- **OpenAI API failure:** Module retries 2x, then fails gracefully
- **Slack posting failure:** Error logged, doesn't block other routes

---

## Route 2: Email Response Parser

### Data Flow
```
Gmail Search [16]
  → Query: "is:unread newer_than:1h"
    → Filter: Emails Found
      → OpenAI [17]
        → Extract: Task ID + Status
          → Text Parser [21]
            → Pattern match: T\d{3}
              → Airtable Search [18]
                → Find task by ID
                  → Airtable Update [19]
                    → Set Status = Completed
                      → Slack [20]
                        → Notify team
```

### Module Details

| Module # | Type | Configuration | Purpose |
|----------|------|---------------|---------|
| 16 | Gmail Search | Query: newer_than:1h | Find recent emails |
| 17 | OpenAI GPT-4o | Temperature: 0.3 | Extract task data (low temp for precision) |
| 21 | Text Parser | Pattern: T\d{3} | Validate task ID format |
| 18 | Airtable Search | Formula: {Task ID}="T002" | Find task record |
| 19 | Airtable Update | Status: Completed | Update task status |
| 20 | Slack Message | Include AI analysis | Notify team |

### AI Prompt for Extraction
```
Read this email and extract ONLY the Task ID mentioned (format: T001, T002, etc.).

Email Subject: {{16.Subject}}
Email Body: {{16.Text Plain}}

Return ONLY the task ID with no other text. Examples:
- If email mentions "task T005", return: T005
- If email mentions "completed T012", return: T012
- If no task mentioned, return: NONE
```

### Limitations

**Current Implementation:**
- Hardcoded task ID (T002) in Airtable Search for testing
- **Future enhancement:** Make fully dynamic using extracted task ID from Text Parser

**Why hardcoded for now:**
- Make.com filter syntax proved complex during build
- Hardcoded version works reliably for demonstration
- Easy to update once per close cycle

---

## Route 3: Automated Bank Reconciliation

### Data Flow
```
Airtable Search [22]
  → Find T003 with Status="In Progress"
    → Filter: Task Active
      → Google Sheets Search [23]
        → Column F = "TRUE" (cleared transactions)
          → Table Aggregator [24]
            → Collect all transactions
              → OpenAI [25]
                → Calculate: deposits, withdrawals, variance
                  → Airtable Update [26]
                    → Status = Completed + Note
                      → Slack [27]
                        → Post reconciliation summary
```

### Module Details

| Module # | Type | Configuration | Purpose |
|----------|------|---------------|---------|
| 22 | Airtable Search | Formula: AND({Task ID}="T003", {Status}="In Progress") | Check if ready |
| 23 | Google Sheets Search | Column: F, Value: TRUE | Get cleared transactions |
| 24 | Table Aggregator | Source: Module 23 | Combine transaction data |
| 25 | OpenAI GPT-4o | Temperature: 0.3 | Perform reconciliation |
| 26 | Airtable Update | Fields: Status, Completion Note | Save results |
| 27 | Slack Message | Include full reconciliation | Notify team |

### Bank Data Structure

**Google Sheet Columns:**

| Column | Field Name | Data Type | Example |
|--------|------------|-----------|---------|
| A | Date | Date | 1/2/2025 |
| B | Description | Text | Customer Payment - ABC Corp |
| C | Amount | Currency | 15000.00 |
| D | Type | Text | Deposit / Payment / Fee |
| E | GL Account | Text | 1100 - AR |
| F | Cleared | Boolean | TRUE |

**Sample Data:**
- 9 transactions totaling $35,325 in deposits
- Withdrawals/fees totaling $39,485.75
- Expected ending balance: $108,839.75
- Actual calculated balance: $120,839.25
- Variance: $12,000 (intentional test variance)

### AI Reconciliation Prompt
```
You are a financial analyst performing a bank reconciliation.

Bank Transaction Data:
{{24.Text}}

Expected Account Balance Information:
- Beginning Balance: $125,000.00
- Expected Ending Balance: $108,839.75

Tasks:
1. Calculate total deposits
2. Calculate total withdrawals/payments
3. Calculate the actual ending balance: Beginning + Deposits - Withdrawals
4. Compare actual vs expected ending balance
5. Calculate variance
6. Determine if reconciled (variance < $100)

Provide your analysis in this format:

**Bank Reconciliation Summary**

Beginning Balance: $125,000.00
Total Deposits: $[amount]
Total Withdrawals: $[amount]
Calculated Ending Balance: $[amount]
Expected Ending Balance: $108,839.75
Variance: $[amount]

Status: [RECONCILED or VARIANCE EXCEEDS THRESHOLD]

Analysis: [Brief 2-3 sentence explanation]
```

### Variance Detection

**Threshold:** $100
- Variance < $100 → Status: RECONCILED
- Variance ≥ $100 → Status: VARIANCE EXCEEDS THRESHOLD

**In test data:**
- AI correctly identifies $12,000 variance
- AI traces variance to DEF Ltd deposit ($12,000)
- Suggests further investigation

---

## Route 4: Close Completion Report

### Data Flow
```
Airtable Search [28]
  → Find T015 with Status="Completed"
    → Filter: Final Task Complete
      → Airtable Search [29]
        → Get all 15 tasks (validate all done)
          → OpenAI [30]
            → Generate: Executive report
              → Slack [31]
                → Post: 🎉 Celebration + Full Report
```

### Module Details

| Module # | Type | Configuration | Purpose |
|----------|------|---------------|---------|
| 28 | Airtable Search | Formula: AND({Task ID}="T015", {Status}="Completed") | Check final task |
| 29 | Airtable Search | Formula: (empty) | Get all tasks for validation |
| 30 | OpenAI GPT-4o | Temperature: 0.6, Max tokens: 1000 | Generate comprehensive report |
| 31 | Slack Message | Include 🎉 celebration | Announce completion |

### AI Report Generation Prompt
```
Create a comprehensive month-end close completion report.

All tasks have been completed. Here is the summary:

Total Tasks: 15
All tasks status: Completed

Create an executive report with these sections:

1. **Executive Summary** (3-4 sentences)
   - Confirm close completion
   - Overall timeline assessment
   - Key achievements

2. **Process Highlights**
   - Automation utilized (email parsing, bank reconciliation)
   - Efficiency gains
   - Notable successes

3. **Metrics**
   - Tasks completed: 15/15
   - Automation time saved: ~3 hours
   - Close duration: 5 days

4. **Recommendations for Next Close**
   - 3-4 actionable improvements
   - Process optimization opportunities

Format professionally with clear sections. Be data-driven and forward-looking. 400-600 words.
```

---

## Data Models

### Airtable: Close Tasks Table

**Schema:**
```javascript
{
  "Task ID": "T001",              // Primary key
  "Task Name": "Lock AP System",
  "Owner": "AP Team",             // Single select
  "Owner Email": "team@company.com",
  "Due Date": "2025-10-15T17:00:00Z",
  "Status": "Not Started",        // Single select
  "Priority": "Critical",         // Single select
  "Depends On": ["recXYZ123"],    // Link to records
  "Estimated Hours": 0.5,
  "Actual Hours": null,
  "Last Reminder Sent": null,
  "Reminder Count": 0,
  "Completion Note": "",
  "Automation Type": "Manual",    // Single select
  "Data Source": "",
  "Variance Threshold": null,
  "Created Time": "2025-10-15T18:30:00Z",
  "Completed Time": null
}
```

**Relationships:**
- **Depends On** → Links to other Close Tasks records
- Creates task dependency chains
- Example: T002 depends on T001 (can't run aged AP report until AP system is locked)

**Task Dependency Tree:**
```
T001 (Lock AP) ─┬─→ T002 (Aged AP Report) ──→ T006 (Accrue Invoices)
                │
                ├─→ T004 (Journal Entries) ─┬─→ T005 (Prelim P&L)
                │                            │
                │                            └─→ T007 (Revenue Rec) ──┬─→ T009 (Intercompany)
                │                                                      │
                └─→ T010 (Payroll Accrual)                           │
                                                                       │
T003 (Bank Rec) ────────────────────────────────────────────────────┘
                                                                       │
                                                                       ↓
                                                        T011 (Final P&L Review) ─┬─→ T012 (Variance Analysis)
                                                                       │          │
                                                                       └──────────┴─→ T013 (Cash Flow)
                                                                                       │
                                                                                       ↓
                                                                            T014 (Board Package)
                                                                                       │
                                                                                       ↓
                                                                            T015 (Close Books)
```

---

## API Integration Details

### Make.com Operations Budget

**Free Tier Limit:** 1,000 operations/month

**Operation Count per Execution:**

| Route | Modules | Operations per Run |
|-------|---------|-------------------|
| Route 1 | 6 modules | ~20 ops (due to aggregator) |
| Route 2 | 7 modules | ~5-10 ops (when emails exist) |
| Route 3 | 7 modules | ~15 ops (when T003 active) |
| Route 4 | 4 modules | ~10 ops (when T015 complete) |

**Monthly Usage (Running Daily):**
- Route 1: 30 days × 20 ops = 600 operations
- Route 2: ~10 emails/month × 7 ops = 70 operations
- Route 3: 1 time/month × 15 ops = 15 operations
- Route 4: 1 time/month × 10 ops = 10 operations

**Total:** ~695 operations/month (under 1,000 limit) ✅

### OpenAI API Usage

**Cost Calculation (GPT-4o pricing as of Oct 2024):**
- Input: $2.50 per 1M tokens
- Output: $10.00 per 1M tokens

**Per Close Cycle:**

| Route | Prompts/Month | Avg Tokens | Cost |
|-------|--------------|------------|------|
| Route 1 | 30 | 500 in + 300 out | $0.04 + $0.003 = $0.043 |
| Route 2 | 10 | 200 in + 50 out | $0.005 + $0.0005 = $0.0055 |
| Route 3 | 1 | 800 in + 400 out | $0.002 + $0.004 = $0.006 |
| Route 4 | 1 | 500 in + 600 out | $0.00125 + $0.006 = $0.00725 |

**Total OpenAI Cost:** ~$0.06/month (negligible)

---

## Security Considerations

### API Keys & Credentials

**Stored in Make.com:**
- ✅ OpenAI API key (encrypted at rest)
- ✅ OAuth tokens for Airtable, Gmail, Sheets, Slack (encrypted)
- ✅ No credentials stored in code or GitHub

**Access Control:**
- Make.com account: 2FA enabled (recommended)
- Airtable: OAuth permissions scoped to specific base
- Gmail: Read-only for search, send for updates
- Slack: Post messages permission only

### Data Privacy

**PII Handling:**
- Email addresses stored in Airtable (team members only)
- No customer PII processed
- Financial data remains in Google Sheets (controlled access)
- Slack notifications visible to channel members only

**Compliance:**
- No data retention beyond operational needs
- Email parser marks emails as read (audit trail in Gmail)
- All updates timestamped in Airtable

---

## Performance & Scalability

### Current Capacity

**Task Limits:**
- Designed for 15-50 tasks per close
- Airtable free tier: 1,200 records total (sufficient for years of history)
- Google Sheets: Up to 1,000 bank transactions per reconciliation

**Execution Time:**
- Route 1: ~15 seconds
- Route 2: ~10 seconds per email
- Route 3: ~20 seconds (depends on transaction count)
- Route 4: ~15 seconds

### Scaling Considerations

**To handle 100+ tasks:**
- Implement pagination in Airtable searches
- Increase aggregator timeout settings
- Consider breaking into multiple scenarios

**To handle multiple entities:**
- Duplicate scenario per entity
- Use Make.com variables for entity-specific configs
- Centralize reporting in master Slack channel

---

## Known Issues & Limitations

### 1. Multiple Slack Notifications

**Issue:** Each task generates separate notification

**Root Cause:** Aggregation not applied before final Slack module

**Impact:** Spam in Slack channel (10-20 messages instead of 1)

**Workaround:** Filter Slack notifications or mute channel

**Future Fix:** Add Array Aggregator before Slack module, combine into single summary message

**Priority:** Low (functional, cosmetic issue only)

---

### 2. Hardcoded Task IDs in Route 2

**Issue:** Airtable Search uses static formula `{Task ID}="T002"`

**Root Cause:** Make.com filter syntax complexity; dynamic extraction required Text Parser + variable mapping that proved unreliable during build

**Impact:** Only one specific task can be updated per run cycle

**Workaround:** Update formula manually for different tasks, or AI extracts correctly and only hardcoded task is searched (works for demo)

**Future Fix:** Implement proper variable extraction from AI result, use in dynamic Airtable formula

**Priority:** Medium (limits full automation, but acceptable for portfolio demo)

---

### 3. Expected Balance Hardcoded in Reconciliation

**Issue:** Expected ending balance ($108,839.75) hardcoded in OpenAI prompt

**Root Cause:** Design decision for simplicity

**Impact:** Must manually update prompt each close cycle with new expected balance

**Workaround:** Update prompt before each run, or store in Airtable configuration

**Future Fix:** Create Airtable "Configuration" table with expected balances; lookup in Route 3

**Priority:** Low (easy manual update, one-time per month)

---

### 4. No Rollback Mechanism

**Issue:** If automation incorrectly updates a task, no automatic rollback

**Root Cause:** One-way data flow by design

**Impact:** Manual correction required in Airtable if AI misinterprets email

**Mitigation:** 
- AI extraction is highly accurate (low temperature = 0.3)
- Slack notifications provide audit trail
- Airtable history tracks all changes

**Future Fix:** Add confirmation step before updates, or "undo" button in Slack

**Priority:** Low (rare occurrence, easy manual fix)

---

## Technology Decisions & Rationale

### Why Make.com over Zapier?

| Factor | Make.com | Zapier |
|--------|----------|--------|
| Free tier multi-step | ✅ Yes | ❌ No ($20/mo) |
| Operations/month | 1,000 | 100 |
| Visual builder | Excellent | Basic |
| Error handling | Superior | Good |
| Cost for this project | $0 | $240/year |

**Decision:** Make.com for cost savings and better functionality

### Why Airtable over Google Sheets?

| Factor | Airtable | Google Sheets |
|--------|----------|---------------|
| Relational data | ✅ Native | ❌ Manual |
| API access | Excellent | Good |
| UI for stakeholders | Professional | Basic |
| Task dependencies | Built-in | Manual |
| Free tier limits | 1,200 records | Unlimited |

**Decision:** Airtable for superior structure and UX

### Why GPT-4o over GPT-3.5-turbo?

| Factor | GPT-4o | GPT-3.5-turbo |
|--------|--------|---------------|
| Accuracy | Excellent | Good |
| Financial reasoning | Superior | Adequate |
| Cost | $0.06/month | $0.02/month |
| Context window | 128k tokens | 16k tokens |

**Decision:** GPT-4o for $0.04/month premium, significantly better output quality

---

## Monitoring & Maintenance

### Daily Checks (Automated)

- ✅ Scenario executes (visible in Make.com history)
- ✅ Slack message posted (visible in channel)
- ✅ No error notifications

### Weekly Checks (Manual - 5 minutes)

- Review Make.com execution history
- Check operation count (should be ~150-175/week)
- Verify all 4 routes executing as expected

### Monthly Maintenance (15 minutes)

- Update expected balances in Route 3 prompt
- Archive previous month's tasks in Airtable
- Review and clear test data
- Check OpenAI API usage and costs

### Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for detailed error resolution.

---

## Future Architecture Improvements

### Phase 2: Enhanced Reliability

- [ ] Add retry logic with exponential backoff
- [ ] Implement dead letter queue for failed messages
- [ ] Add health check endpoint
- [ ] Create monitoring dashboard

### Phase 3: Advanced Features

- [ ] Real-time dependency chain execution
- [ ] Predictive close completion ML model
- [ ] Multi-entity consolidation support
- [ ] Mobile app for status updates

---

## Contributing

This is a portfolio project, but suggestions are welcome! Please open an issue to discuss proposed changes.

---

## References

- [Make.com API Documentation](https://www.make.com/en/api-documentation)
- [Airtable API Reference](https://airtable.com/developers/web/api/introduction)
- [OpenAI API Guide](https://platform.openai.com/docs)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [Slack API](https://api.slack.com/)

---

**Last Updated:** October 23, 2025
