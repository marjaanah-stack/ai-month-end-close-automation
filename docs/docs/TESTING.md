# Testing Guide

This guide walks you through testing all 4 routes of the Month-End Close Orchestrator to ensure everything works correctly.

---

## Pre-Test Checklist

Before testing, ensure:

- ✅ All Make.com connections are active (green checkmarks)
- ✅ Airtable has 15 sample tasks (T001-T015)
- ✅ Google Sheet has bank transaction data
- ✅ Slack channel `#month-end-close` exists
- ✅ OpenAI API key is valid and has credits

**Estimated testing time:** 30-45 minutes

---

## Test Strategy

We'll test each route independently, then run a complete end-to-end test.
```
Test Order:
1. Route 1 (Daily Report) - Quickest to verify
2. Route 3 (Bank Rec) - Tests Google Sheets integration
3. Route 2 (Email Parser) - Tests Gmail + AI extraction
4. Route 4 (Final Report) - Tests completion workflow
5. End-to-End - Full close cycle simulation
```

---

## Route 1: Daily Progress Report

### Objective
Verify that the scenario generates and posts AI-powered daily summaries.

### Setup (2 minutes)

1. **Verify T001 exists in Airtable:**
   - Open your Airtable base
   - Confirm Task T001 is present (any status is fine)

2. **Clear Slack channel:**
   - Go to `#month-end-close`
   - Note the last message timestamp (for comparison)

### Test Steps

**Step 1:** Run the scenario
```
Make.com → Click "Run once" button
```

**Step 2:** Monitor execution
- Watch modules turn green as they execute
- Route 1 modules should all complete successfully
- Routes 2-4 may stop at filters (this is correct!)

**Step 3:** Verify Slack output

Expected message format:
```
📊 **DAILY CLOSE PROGRESS REPORT**
Date: [Current date]

**Overview:**
✅ Completed: X/15 (XX%)
🔄 In Progress: X
⏸️ Not Started: X
🚫 Blocked: X
⏰ Overdue: X

**Executive Summary:**
[AI-generated 2-3 paragraph analysis with action items]

---

View detailed tracker: [Airtable URL]
```

### Pass/Fail Criteria

✅ **PASS if:**
- Slack message appears within 30 seconds
- Message shows current date
- Task counts are accurate (matches Airtable)
- AI summary is coherent and relevant
- No error messages in Make.com

❌ **FAIL if:**
- No Slack message after 60 seconds
- Message shows placeholder text (e.g., "{{variable}}")
- Task counts are all zeros
- Make.com shows red error indicators

### Troubleshooting

**Issue:** "No message in Slack"
- Check: Is Slack module (9) connected?
- Check: Is channel name exactly `#month-end-close`?
- Fix: Reconnect Slack in Make.com

**Issue:** "Task count shows 0 or 15"
- Check: Variable calculation in module 15
- Fix: See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#task-count-issues)

**Issue:** "AI summary is generic/unhelpful"
- Check: Is OpenAI receiving task data?
- Fix: Review OpenAI prompt in module 8

---

## Route 3: Bank Reconciliation

### Objective
Verify automated bank reconciliation with AI analysis.

### Setup (3 minutes)

1. **Set T003 to "In Progress":**
   - Open Airtable
   - Find Task T003 (Bank Reconciliation - Operating Account)
   - Change Status from "Not Started" to "In Progress"
   - Save

2. **Verify Google Sheet data:**
   - Open your `Bank Data - Operating Account` sheet
   - Confirm 9 transactions are present
   - Verify Column F (Cleared) shows TRUE for all

3. **Clear previous completion note:**
   - In Airtable T003, clear any existing "Completion Note"

### Test Steps

**Step 1:** Run the scenario
```
Make.com → Click "Run once"
```

**Step 2:** Monitor Route 3 execution
- Watch modules 22-27 execute
- Google Sheets module (23) should show "9 bundles"
- OpenAI module (25) should process for ~10 seconds

**Step 3:** Verify Airtable update

Check T003 in Airtable:
- Status should change to "Completed"
- Completion Note should contain reconciliation summary
- Actual Hours should show "0.5"

**Step 4:** Verify Slack notification

Expected message format:
```
🤖 **AUTOMATED: Bank Reconciliation Complete**

Task: T003 - Bank Reconciliation - Operating Account

**Bank Reconciliation Summary**

Beginning Balance: $125,000.00
Total Deposits: $35,325.00
Total Withdrawals: $39,485.75
Calculated Ending Balance: $120,839.25
Expected Ending Balance: $108,839.75
Variance: $12,000.00

Status: VARIANCE EXCEEDS THRESHOLD

Analysis: [AI explanation of the $12,000 variance]

This reconciliation was performed automatically using AI analysis.
```

### Pass/Fail Criteria

✅ **PASS if:**
- T003 status updates to "Completed"
- Completion Note contains full reconciliation details
- Slack shows reconciliation summary
- AI correctly identifies the $12,000 variance
- AI mentions DEF Ltd deposit as likely cause

❌ **FAIL if:**
- T003 remains "In Progress"
- No Slack message appears
- Reconciliation shows $0 deposits/withdrawals
- Route 3 doesn't execute at all

### Troubleshooting

**Issue:** "Route 3 didn't execute"
- Check: Is T003 status "In Progress" (not "Not Started")?
- Check: Airtable module 22 formula correct?
- Fix: Verify formula: `AND({Task ID}="T003", {Status}="In Progress")`

**Issue:** "Google Sheets returned no data"
- Check: Sheet ID correct in T003 Data Source field?
- Check: Column F has TRUE values?
- Fix: Verify Google Sheets connection in Make.com

**Issue:** "Variance calculation is wrong"
- Check: OpenAI prompt includes transaction data?
- Check: Transaction amounts are numbers, not text?
- Fix: Review module 24 (Aggregator) output

---

## Route 2: Email Response Parser

### Objective
Verify AI extracts task information from emails and updates Airtable.

### Setup (5 minutes)

1. **Reset test task:**
   - In Airtable, set T001 back to "Not Started"
   - Clear any Completion Note
   - Set Actual Hours to blank

2. **Mark all emails as read:**
   - Go to Gmail
   - Select all unread emails related to testing
   - Mark as read (so we start fresh)

3. **Prepare test email (don't send yet):**
```
   To: [your-email]@gmail.com
   Subject: Task T001 completed
   Body: Hi, I've finished task T001. The AP system is now locked and no new entries will be accepted.
```

### Test Steps

**Step 1:** Send test email
- Send the prepared email to yourself
- **Important:** Do NOT open/read it

**Step 2:** Wait 30 seconds
- Gmail needs time to index the email
- Make.com checks for unread emails

**Step 3:** Run the scenario
```
Make.com → Click "Run once"
```

**Step 4:** Monitor Route 2 execution
- Gmail module (16) should find 1 email
- OpenAI module (17) should extract "T001"
- Airtable modules (18-19) should update T001

**Step 5:** Verify Airtable update

Check T001:
- Status should change to "Completed"
- Completion Note should mention email update
- Record should show timestamp update

**Step 6:** Verify Slack notification

Expected message format:
```
✅ **Task Updated via Email**

Task: T001 - Lock AP System - Prevent New Entries
Previous Status: Not Started
New Status: Completed

Updated based on email from: [Your Name]

AI Analysis: T001
```

**Step 7:** Verify email marked as read
- Check Gmail - test email should now be marked as read

### Pass/Fail Criteria

✅ **PASS if:**
- Gmail module finds the email
- AI correctly extracts "T001"
- T001 status updates to "Completed"
- Slack notification appears
- Email marked as read

❌ **FAIL if:**
- Gmail module finds 0 emails (filter blocked it)
- AI extracts wrong task ID or "NONE"
- T001 doesn't update
- Slack shows wrong task information

### Troubleshooting

**Issue:** "Gmail found no emails"
- Check: Is email still unread?
- Check: Gmail query in module 16 correct?
- Fix: Change query to just `newer_than:1h` (broader search)

**Issue:** "AI extracted NONE or wrong ID"
- Check: Does email body mention "T001"?
- Check: OpenAI prompt in module 17
- Fix: Improve email wording to be more explicit

**Issue:** "Wrong task updated (not T001)"
- Known issue: Airtable Search (18) is hardcoded to T002
- Fix: Change formula to `{Task ID}="T001"` for this test

**Issue:** "Multiple Slack messages (spam)"
- Known limitation: Aggregation not implemented
- Impact: Cosmetic only, all functional
- Fix: See [Architecture](ARCHITECTURE.md#known-issues)

---

## Route 4: Close Completion Report

### Objective
Verify final report generation when all tasks complete.

### Setup (3 minutes)

1. **Complete all tasks:**
   - In Airtable, set ALL tasks (T001-T015) to "Completed"
   - This simulates end of close cycle
   - Shortcut: Use Airtable's bulk edit feature

2. **Clear Slack:**
   - Note last message in channel for reference

### Test Steps

**Step 1:** Run the scenario
```
Make.com → Click "Run once"
```

**Step 2:** Monitor Route 4 execution
- Airtable module 28 should find T015 with Status="Completed"
- Filter should PASS (not block)
- OpenAI module 30 should generate long report (~600 words)

**Step 3:** Verify Slack message

Expected message format:
```
🎉 **MONTH-END CLOSE COMPLETE!** 🎉

All 15 tasks have been successfully completed.

---

**Month-End Close Completion Report**

**Executive Summary**
[3-4 sentences confirming completion, timeline, achievements]

**Process Highlights**
- Automation Utilized: [Email parsing, bank rec details]
- Efficiency Gains: ~3 hours saved
- Notable Successes: [Key wins]

**Metrics**
| Metric | Result |
|--------|--------|
| Total Tasks Completed | 15/15 |
| Close Duration | 5 Days |
| Time Saved via Automation | ~3 Hours |
| On-Time Completion Rate | 100% |

**Recommendations for Next Close**
1. [Actionable improvement 1]
2. [Actionable improvement 2]
3. [Actionable improvement 3]
4. [Actionable improvement 4]

---

**Automation Summary:**
✅ Daily AI progress reports
✅ Email-based task updates with AI parsing
✅ Automated bank reconciliation
✅ Comprehensive close analysis

Next close begins: [Date ~30 days from now]

Great work team! 🚀
```

### Pass/Fail Criteria

✅ **PASS if:**
- Route 4 executes (doesn't stop at filter)
- Slack shows comprehensive report
- Report includes all required sections
- AI provides specific, actionable recommendations
- 🎉 celebration emoji appears
- Report is professional and well-formatted

❌ **FAIL if:**
- Route 4 stops at filter (T015 not completed?)
- Slack shows placeholder text like `{{variable}}`
- Report is generic/template-like
- Sections are missing or incomplete

### Troubleshooting

**Issue:** "Route 4 didn't execute"
- Check: Is T015 Status = "Completed"?
- Check: Airtable module 28 formula
- Fix: Ensure formula: `AND({Task ID}="T015", {Status}="Completed")`

**Issue:** "Report is too generic"
- Check: Is OpenAI receiving all task data?
- Check: Module 29 returning 15 records?
- Fix: Verify Airtable Search (29) has empty formula

**Issue:** "Slack formatting is broken"
- Check: Markdown in Slack module (31)
- Fix: Ensure proper newlines and formatting

---

## End-to-End Integration Test

### Objective
Simulate a complete 5-day close cycle with all features.

### Setup (10 minutes)

1. **Reset Airtable completely:**
   - Set ALL tasks (T001-T015) to "Not Started"
   - Clear all Completion Notes
   - Clear all Actual Hours
   - Set T003 Automation Type to "Auto-Reconciliation"

2. **Clear Gmail:**
   - Archive or mark as read all test emails

3. **Clear Slack:**
   - Note the starting point in `#month-end-close`

### Test Script

**Day 1 Simulation:**
```
Action: Run scenario
Expected: 
- Daily report shows 0/15 completed
- No other routes execute
- Slack shows progress report only
```

**Day 2 Simulation:**
```
Action 1: Send email "Task T001 completed"
Action 2: Wait 30 seconds
Action 3: Run scenario

Expected:
- Daily report shows 1/15 completed
- Email parser updates T001
- Slack shows 2 messages (daily + T001 update)
```

**Day 3 Simulation:**
```
Action 1: Set T003 to "In Progress"
Action 2: Run scenario

Expected:
- Daily report shows 1/15 completed
- Bank rec executes automatically
- T003 updates to "Completed"
- Slack shows reconciliation summary
```

**Day 4 Simulation:**
```
Action 1: Send email "Task T002 done"
Action 2: Manually set T004-T014 to "Completed" (simulate team work)
Action 3: Run scenario

Expected:
- Daily report shows 14/15 completed
- Email parser updates T002
- T015 is last remaining task
```

**Day 5 Simulation (Final Close):**
```
Action 1: Set T015 to "Completed"
Action 2: Run scenario

Expected:
- Daily report shows 15/15 completed
- Final completion report generates
- 🎉 Celebration message in Slack
- Comprehensive executive report
```

### Pass/Fail Criteria

✅ **PASS if:**
- All 4 routes execute at appropriate times
- Task statuses update correctly
- Slack shows progression over "5 days"
- Final report confirms 15/15 completion
- No errors or failed modules
- All automation features demonstrated

❌ **FAIL if:**
- Any route fails to execute when triggered
- Task updates are missed
- Slack messages are missing or incorrect
- Final report doesn't generate

### End-to-End Success Metrics

At completion, you should have:
- ✅ 15 tasks marked "Completed" in Airtable
- ✅ ~10-15 Slack messages showing progress
- ✅ Bank reconciliation completed automatically
- ✅ 2+ email-triggered updates
- ✅ Final celebration report

---

## Performance Benchmarks

### Expected Execution Times

| Route | Avg Time | Max Time |
|-------|----------|----------|
| Route 1 | 15 sec | 30 sec |
| Route 2 | 10 sec | 20 sec |
| Route 3 | 20 sec | 40 sec |
| Route 4 | 15 sec | 30 sec |

**Total scenario run:** 20-60 seconds depending on which routes execute

### Operation Counts

**Per execution:**
- Minimum: 15-20 operations (Route 1 only)
- Maximum: 50-60 operations (all routes)
- Average: 20-25 operations

**Monthly budget check:**
- Daily runs: 30 × 20 = 600 operations
- Email triggers: ~10 × 7 = 70 operations
- Special runs: ~30 operations
- **Total: ~700/1000 operations** ✅ Under limit

---

## Test Data Reset

After testing, reset for production use:

**Airtable:**
```
1. Delete all test tasks OR
2. Set all to "Not Started" and clear notes OR
3. Duplicate base for production, keep test base separate
```

**Gmail:**
```
1. Archive all test emails
2. Consider creating a filter to auto-label close emails
```

**Slack:**
```
1. Pin important messages
2. Consider muting channel except for @mentions
```

**Make.com:**
```
1. Review execution history
2. Clear any failed runs
3. Adjust schedule if needed (currently manual "Run once")
```

---

## Continuous Testing

### Weekly Health Check (5 minutes)

Run this quick test every Monday:
```
1. Run scenario once
2. Verify daily report in Slack
3. Check Make.com operations count
4. Review any error notifications
```

### Monthly Validation (15 minutes)

Before each month-end:
```
1. Run full end-to-end test (above)
2. Update expected balances in Route 3
3. Verify all connections still active
4. Test with 1 real email
5. Confirm Slack notifications working
```

---

## Test Checklist Summary

Use this checklist for each testing session:

**Pre-Test:**
- [ ] All connections active in Make.com
- [ ] Airtable has sample data
- [ ] Google Sheet has transaction data
- [ ] Slack channel exists and accessible

**Route 1:**
- [ ] Daily report generates
- [ ] Slack message formatted correctly
- [ ] Task counts accurate

**Route 2:**
- [ ] Email sent and unread
- [ ] AI extracts correct task ID
- [ ] Airtable updates
- [ ] Slack notification appears

**Route 3:**
- [ ] T003 set to "In Progress"
- [ ] Bank rec calculates correctly
- [ ] Variance identified
- [ ] T003 marked complete

**Route 4:**
- [ ] All tasks marked complete
- [ ] Final report generates
- [ ] Report includes all sections
- [ ] Celebration message appears

**Post-Test:**
- [ ] No errors in Make.com
- [ ] Operations under budget
- [ ] Ready for production use

---

## Getting Help

**If tests fail:**
1. Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) first
2. Review [ARCHITECTURE.md](ARCHITECTURE.md) for technical details
3. Check Make.com execution history for error details
4. Open a GitHub issue with:
   - Which route failed
   - Error message/screenshot
   - Steps to reproduce

---

## Test Results Template

Document your test results:
```markdown
## Test Session: [Date]

**Environment:**
- Make.com operations available: XXX/1000
- OpenAI credits: $XX.XX
- Test duration: XX minutes

**Results:**
- Route 1: ✅ PASS / ❌ FAIL - [notes]
- Route 2: ✅ PASS / ❌ FAIL - [notes]
- Route 3: ✅ PASS / ❌ FAIL - [notes]
- Route 4: ✅ PASS / ❌ FAIL - [notes]
- End-to-End: ✅ PASS / ❌ FAIL - [notes]

**Issues Found:**
1. [Issue description]
2. [Issue description]

**Resolution:**
1. [How you fixed it]
2. [How you fixed it]

**Ready for Production:** YES / NO
```

---

**Happy Testing! 🧪**
