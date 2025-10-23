# Troubleshooting Guide

This guide helps you diagnose and fix common issues with the Month-End Close Orchestrator.

---

## Quick Diagnostic Checklist

Start here when something goes wrong:
```
□ Are all Make.com connections showing green checkmarks?
□ Is the scenario turned ON (not just saved)?
□ Did you click "Run once" to test?
□ Are there error messages in Make.com execution history?
□ Is your OpenAI account active with available credits?
□ Do you have operations remaining (check XXX/1000)?
```

---

## Connection Issues

### Error: "Connection expired" or "Reconnect required"

**Symptoms:**
- Red X on module
- Error: "The operation failed with an error. [401] Unauthorized"

**Cause:** OAuth token expired (common after 30-90 days)

**Fix:**

**Step 1:** Click on the failing module

**Step 2:** Click on the connection dropdown

**Step 3:** Click "Reconnect" or "Add" new connection

**Step 4:** Re-authorize the app (login and grant permissions)

**Step 5:** Test by running scenario

**Prevention:** Make.com usually auto-refreshes tokens, but manual reconnection needed occasionally

---

### Error: "Invalid API key" (OpenAI)

**Symptoms:**
- OpenAI module shows error
- Message: "Incorrect API key provided"

**Cause:** API key revoked, expired, or typed incorrectly

**Fix:**

**Step 1:** Go to [OpenAI API Keys](https://platform.openai.com/api-keys)

**Step 2:** Check if your key is still active

**Step 3:** If expired, create new API key

**Step 4:** In Make.com:
- Click OpenAI module
- Click connection name
- Click "Show advanced settings"
- Update API key field
- Save

**Step 5:** Test scenario

**Prevention:** Use OpenAI API keys with no expiration, and store securely

---

### Error: "Spreadsheet not found" (Google Sheets)

**Symptoms:**
- Google Sheets module fails
- Error: "Requested entity was not found"

**Cause:** Sheet ID incorrect or permissions revoked

**Fix:**

**Step 1:** Get correct Sheet ID:
```
URL: https://docs.google.com/spreadsheets/d/[SHEET_ID_HERE]/edit
Copy the SHEET_ID portion
```

**Step 2:** Update in Make.com:
- Click Google Sheets module (23)
- Paste correct Spreadsheet ID
- Save

**Step 3:** Verify permissions:
- Sheet must be owned by or shared with the Google account connected to Make.com
- Sharing permissions: "Editor" or "Owner"

**Step 4:** Test scenario

---

## Route-Specific Issues

### Route 1: Daily Report Not Posting

**Symptom:** No Slack message appears

**Diagnostic Steps:**

**Check 1:** Did Route 1 execute at all?
- Look at Make.com execution history
- Module 2 should show green checkmark
- If red X on module 2, T001 doesn't exist in Airtable

**Check 2:** Did the filter pass?
- Check filter between Router and Route 1
- Filter should check if "2.ID exists"
- If blocked, module 2 returned no results

**Check 3:** Did OpenAI generate text?
- Check module 8 output
- Click on module 8 in execution history
- Look at "Result" field - should contain text
- If empty, OpenAI prompt may be malformed

**Check 4:** Did Slack module execute?
- Check module 9 in history
- If red X, check connection
- If green but no message, check channel name

**Common Fixes:**

**Fix 1:** T001 doesn't exist
```
Solution: Add Task T001 to Airtable Close Tasks table
Minimum fields needed: Task ID = "T001", Task Name = "Any name"
```

**Fix 2:** Slack channel name wrong
```
Solution: 
- Click Slack module (9)
- Change channel to: #month-end-close
- Must include # symbol
- Check spelling exactly
```

**Fix 3:** OpenAI prompt too long
```
Solution:
- Check if you have 100+ tasks (exceeds context window)
- Reduce Max Tokens in OpenAI module to 500
- Or split into multiple prompts
```

---

### Route 2: Email Not Being Processed

**Symptom:** Email sent but task not updated

**Diagnostic Steps:**

**Check 1:** Did Gmail find the email?
- Look at module 16 in execution history
- Should show "1 bundle" or more
- If "0 bundles", email wasn't found

**Check 2:** Is email still unread?
- Go to Gmail
- Check if test email is unread
- If read, Gmail module won't find it (query filters for unread)

**Check 3:** Did AI extract task ID?
- Check module 17 (OpenAI) output
- Should show just "T001" or similar
- If shows "NONE", AI didn't find task ID in email

**Check 4:** Did Airtable Search find the task?
- Check module 18 output
- If "0 bundles", task doesn't exist OR formula is wrong
- Currently hardcoded to T002 (known limitation)

**Common Fixes:**

**Fix 1:** Email marked as read
```
Solution:
- Send a NEW test email
- Do not open it
- Run scenario immediately (within 1 hour)
```

**Fix 2:** Gmail query too restrictive
```
Solution:
- Click Gmail module (16)
- Change Query from: subject:(task OR T0) is:unread newer_than:1h
- To simpler: is:unread newer_than:1h
- This catches all recent unread emails
```

**Fix 3:** Task ID not in email
```
Solution:
- Ensure email body mentions task ID explicitly
- Good: "I completed task T001"
- Good: "T005 is done"
- Bad: "The AP task is finished" (no ID mentioned)
```

**Fix 4:** Wrong task being searched (hardcoded issue)
```
Known Limitation: Airtable Search (18) formula is hardcoded
Current formula: {Task ID}="T002"

Workaround:
- For testing T001: Change formula to {Task ID}="T001"
- For testing T003: Change formula to {Task ID}="T003"
- Must manually update for each test

Future Fix: Implement dynamic task ID extraction
```

---

### Route 3: Bank Reconciliation Not Running

**Symptom:** T003 stays "In Progress", no Slack message

**Diagnostic Steps:**

**Check 1:** Is T003 status "In Progress"?
- Check Airtable - exact status must be "In Progress"
- Not "in progress" (lowercase)
- Not "Not Started"

**Check 2:** Did Route 3 execute at all?
- Check if module 22 has green checkmark
- If red X or gray (not executed), check filter

**Check 3:** Did Google Sheets return data?
- Check module 23 output
- Should show "9 bundles" (or number of transactions)
- If "0 bundles", no cleared transactions found

**Check 4:** Did AI calculate correctly?
- Check module 25 (OpenAI) output
- Should show dollar amounts and variance
- If generic text, prompt may be wrong

**Common Fixes:**

**Fix 1:** T003 status is wrong
```
Solution:
- Go to Airtable
- Find Task T003
- Set Status dropdown to exactly: "In Progress"
- Make sure it's not "Not Started" or "Completed"
```

**Fix 2:** Google Sheets has no cleared transactions
```
Solution:
- Open your Google Sheet
- Check Column F (Cleared)
- Ensure at least some rows have TRUE
- If all FALSE or blank, module 23 returns no data
- Update some rows to TRUE
```

**Fix 3:** Google Sheets column name mismatch
```
Solution:
- Verify column headers exactly match:
  A: Date
  B: Description
  C: Amount
  D: Type
  E: GL Account
  F: Cleared
- Column F must be named "Cleared" (case-sensitive)
```

**Fix 4:** Expected balance is wrong
```
Known Issue: Expected balance is hardcoded in prompt

Current value: $108,839.75

To update:
- Click OpenAI module (25)
- Find line: "Expected Ending Balance: $108,839.75"
- Change to your actual expected balance
- Save and re-run
```

---

### Route 4: Final Report Not Generating

**Symptom:** All tasks complete but no celebration message

**Diagnostic Steps:**

**Check 1:** Is T015 status "Completed"?
- Must be exactly "Completed" (case-sensitive)
- Check Airtable Task T015

**Check 2:** Did module 28 find T015?
- Check execution history
- Module 28 should return "1 bundle"
- If "0 bundles", formula is wrong or T015 doesn't exist

**Check 3:** Did filter pass?
- Check filter after module 28
- Should pass if T015 found
- If blocked, T015 wasn't marked complete

**Check 4:** Did OpenAI generate report?
- Check module 30 output
- Should show 400-600 word report
- If empty or error, API issue

**Common Fixes:**

**Fix 1:** T015 not completed
```
Solution:
- Go to Airtable
- Find Task T015 (Close Books & Archive)
- Set Status to: "Completed"
- Save and run scenario
```

**Fix 2:** T015 doesn't exist
```
Solution:
- Check if you have all 15 tasks (T001-T015)
- If missing T015, add it:
  - Task ID: T015
  - Task Name: Close Books & Archive
  - Status: Completed (for testing)
```

**Fix 3:** Slack message shows placeholder text
```
Symptom: Message shows {{30.Result}} instead of report

Solution:
- Click Slack module (31)
- Find the Text field
- Look for {{[OpenAI module number].Result}}
- Delete that line
- Click in that spot
- From left panel, click on module 30 → Result
- This properly maps the variable
- Save and test
```

---

## Data Issues

### Issue: Task Counts Are Wrong

**Symptom:** Daily report shows 0/15 or 15/15 when tasks are mixed

**Cause:** Variable calculation logic error

**Fix:**

**Step 1:** Check module 15 (Set Variable)

**Step 2:** Verify these variables are calculated:
```
completed_count = [formula counting completed tasks]
in_progress_count = [formula counting in progress]
not_started_count = [formula counting not started]
```

**Step 3:** If formulas are missing or wrong:
```
completed_count should use: length(filter(tasks, status="Completed"))
```

**Step 4:** Alternatively, simplify:
- Remove complex calculations
- Let AI calculate from raw task data
- AI is better at counting than Make.com formulas

---

### Issue: Bank Reconciliation Shows $0 Everywhere

**Symptom:** All amounts in reconciliation summary are $0.00

**Cause:** Google Sheets data not reaching OpenAI

**Fix:**

**Step 1:** Check module 24 (Table Aggregator) output
- Click on module 24 in execution history
- Expand "Text" field
- Should show transaction data in structured format
- If empty, aggregator didn't receive data

**Step 2:** If empty, check module 23 (Google Sheets)
- Should show multiple bundles (one per transaction)
- If 0 bundles, see "Route 3: Bank Reconciliation" fixes above

**Step 3:** Verify OpenAI prompt
- Click module 25
- Ensure prompt includes: {{24.Text}}
- This inserts transaction data
- If missing, add it

**Step 4:** Check transaction amounts
- In Google Sheet, verify Column C (Amount) contains numbers
- Not text like "$1,000" - should be just: 1000
- Format column as Number, not Currency (removes $)

---

### Issue: Multiple Slack Messages (Spam)

**Symptom:** Get 10-20 duplicate Slack messages on each run

**Cause:** Known limitation - bundles not aggregated before Slack

**Impact:** Cosmetic only, all messages have correct data

**Current Workaround:**
- Accept the spam during testing
- Mute Slack channel notifications
- Or filter Slack by specific keywords

**Permanent Fix (Future):**
```
Add Array Aggregator before final Slack module:
- Aggregates all task updates into single array
- Slack posts one combined message
- Requires restructuring Route 2 end
```

**Priority:** Low - doesn't affect functionality

---

## Make.com Platform Issues

### Error: "Quota exceeded"

**Symptom:** Scenario won't run, message about operations limit

**Cause:** Used 1,000+ operations this month (free tier limit)

**Check:**
- Go to Make.com dashboard
- Look at "Operations used: XXX / 1,000"

**Solutions:**

**Option 1:** Wait until next month (quota resets)

**Option 2:** Upgrade to paid plan
- Core plan: $9/month for 10,000 operations
- Rarely needed for this project

**Option 3:** Optimize operations
- Reduce daily runs (every other day instead of daily)
- Remove unnecessary modules
- Use filters to prevent unnecessary executions

**Prevention:**
- Monitor operations weekly
- Current project uses ~700/month, safe margin

---

### Error: "Scenario timed out"

**Symptom:** Execution stops mid-way, shows timeout error

**Cause:** OpenAI took too long (>40 seconds) or Google Sheets has huge dataset

**Fix:**

**For OpenAI timeout:**
- Reduce Max Tokens from 1000 to 500
- Simplify prompt
- Use faster model (gpt-3.5-turbo instead of gpt-4o)

**For Google Sheets timeout:**
- Limit rows to last 30 days only
- Add date filter in module 23
- Only search recent transactions

**For Airtable timeout:**
- Reduce Max Records from 100 to 50
- Add more specific filters

---

### Error: "Incomplete execution"

**Symptom:** Scenario runs but some modules are gray (not executed)

**Cause:** Filter blocked execution OR scenario hit an error earlier

**Diagnosis:**

**Step 1:** Look for filter icons in flow
- Filters have a small funnel icon
- Red X means filter blocked execution
- This is often CORRECT behavior

**Step 2:** Check if filter should have passed
- Click on filter
- View condition (e.g., "ID exists")
- Check previous module output - did it return data?

**Step 3:** If filter should have passed:
- Check filter logic
- May need to adjust condition

**Common Filter Issues:**
```
Filter: "Emails Found"
Condition: 16.Message ID exists
Blocked: Gray/Red X

This is CORRECT if no emails found!
Solution: Send test email and re-run
```
```
Filter: "Bank Rec Task Active"
Condition: 22.ID exists
Blocked: Gray/Red X

This is CORRECT if T003 not "In Progress"!
Solution: Set T003 to "In Progress" in Airtable
```

---

## OpenAI / AI Issues

### Issue: AI Extracts Wrong Information

**Symptom:** AI returns "NONE" when task ID is clearly in email

**Cause:** Prompt ambiguity or email format unexpected

**Fix:**

**Step 1:** Review the email content
- Make sure task ID is explicit: "T001" or "Task T001"
- Not implicit: "the AP task" or "first task"

**Step 2:** Improve prompt clarity
```
Current prompt: "Extract the task ID"

Better prompt: "Extract ONLY the task ID in format TXXX (like T001, T002). 
Return just the ID, nothing else. If no task ID found, return NONE."
```

**Step 3:** Lower temperature
- Currently 0.3 (good)
- Could go to 0.1 for even more deterministic output
- Trade-off: Less flexible parsing

**Step 4:** Test with different email formats
```
Format 1: "Task T001 is complete" ✅ Works
Format 2: "T001 done" ✅ Works
Format 3: "Finished the AP locking task" ❌ No ID, returns NONE (correct)
```

---

### Issue: AI Report is Too Generic

**Symptom:** Final report sounds template-like, not specific to your data

**Cause:** OpenAI not receiving task details OR temperature too low

**Fix:**

**Step 1:** Verify data input
- Check module 29 output (should have all 15 tasks)
- Ensure OpenAI prompt includes: {{29.Task Name}} or similar

**Step 2:** Increase temperature
- Currently 0.6
- Try 0.7 or 0.8 for more creative output
- Trade-off: Less consistent formatting

**Step 3:** Improve prompt specificity
```
Add to prompt:
"Use specific task names and real completion dates. 
Mention specific achievements like:
- Which reconciliation was completed
- Which reports were generated
- Actual bottlenecks encountered
Be data-driven and specific, not generic."
```

**Step 4:** Increase Max Tokens
- Currently 1000
- Try 1500 for longer, more detailed reports

---

### Issue: OpenAI Error "Rate limit exceeded"

**Symptom:** Error message from OpenAI module

**Cause:** Too many API calls in short time (rare with this project)

**Fix:**

**Immediate:** Wait 60 seconds and try again

**Prevention:**
- Don't run scenario more than once per minute
- Free tier: 3 requests/min for GPT-4
- Tier 1+: Higher limits

**Check your tier:**
- Go to [OpenAI Usage](https://platform.openai.com/usage)
- View current tier and limits

---

## Performance Issues

### Issue: Scenario Takes Too Long (>2 minutes)

**Symptom:** Execution hangs, doesn't complete

**Causes & Fixes:**

**Cause 1: OpenAI thinking too long**
```
Fix: Reduce Max Tokens to 500 in all OpenAI modules
```

**Cause 2: Too many Airtable records**
```
Fix: Add "Max records: 50" to Airtable Search modules
```

**Cause 3: Large Google Sheet**
```
Fix: Add date filter - only last 30 days
Module 23 → Add condition: Date > today - 30
```

**Cause 4: Network latency**
```
No fix - wait for better connectivity
Make.com servers may be slow (rare)
```

---

## Error Messages Reference

### Common Error Codes

**[401] Unauthorized**
- **Meaning:** Invalid or expired credentials
- **Fix:** Reconnect the app (see Connection Issues above)

**[403] Forbidden**
- **Meaning:** No permission to access resource
- **Fix:** Check sharing settings (Google Sheets) or workspace permissions (Slack)

**[404] Not Found**
- **Meaning:** Resource doesn't exist (wrong ID, deleted item)
- **Fix:** Verify IDs (Sheet ID, Record ID, etc.)

**[429] Too Many Requests**
- **Meaning:** Rate limit hit
- **Fix:** Wait 60 seconds, then retry

**[500] Internal Server Error**
- **Meaning:** Service is down (Airtable, OpenAI, etc.)
- **Fix:** Wait and retry, check service status pages

---

## Debugging Workflow

When something breaks, follow this systematic approach:

### Step 1: Identify Which Route Failed

Check Make.com execution history:
```
✅ Route 1 (modules 2,5,14,15,8,9) - All green?
✅ Route 2 (modules 16,17,21,18,19,20) - All green?
✅ Route 3 (modules 22,23,24,25,26,27) - All green?
✅ Route 4 (modules 28,29,30,31) - All green?
```

### Step 2: Find First Failed Module

- Look for first red X in the route
- This is where error occurred
- Everything after will also fail

### Step 3: Check Module Output

- Click on failed module
- Look at "Output" tab
- Read error message
- Note any error codes

### Step 4: Verify Module Input

- Check "Input" tab
- Verify all required fields have data
- Look for empty fields that should have values

### Step 5: Check Previous Module

- Click module immediately before failed one
- Verify it returned expected data
- If empty/wrong data, fix that module first

### Step 6: Test Connections

- Click connection name
- Verify it's active (green checkmark)
- Reconnect if needed

### Step 7: Simplify & Test

- Temporarily remove complex logic
- Test with hardcoded values
- Once working, add back complexity

### Step 8: Document & Fix

- Note what you changed
- Test again
- Document fix in case it recurs

---

## Getting Additional Help

### Before Opening an Issue

1. ✅ Check this troubleshooting guide
2. ✅ Review [ARCHITECTURE.md](ARCHITECTURE.md) for design details
3. ✅ Check [TESTING.md](TESTING.md) for proper test procedures
4. ✅ Review Make.com execution history
5. ✅ Try reconnecting all connections

### When Opening a GitHub Issue

Include:
```markdown
**Route:** [Which route failed: 1, 2, 3, or 4]

**Module:** [Which module number showed error]

**Error Message:** 
[Paste exact error text from Make.com]

**Screenshot:**
[Upload screenshot of execution history]

**What I tried:**
1. [First thing you tried]
2. [Second thing]

**Environment:**
- Make.com operations used: XXX/1000
- OpenAI tier: [Free/Tier 1/etc]
- Last successful run: [Date]
```

### Useful Resources

- [Make.com Help Center](https://www.make.com/en/help)
- [Make.com Community](https://community.make.com/)
- [OpenAI Help](https://help.openai.com/)
- [Airtable Support](https://support.airtable.com/)

---

## Prevention Best Practices

### Weekly Maintenance (5 minutes)
```
□ Check operations used (should be ~150-200/week)
□ Verify all connections still active
□ Review last 7 days execution history
□ Test with one manual run
```

### Monthly Maintenance (15 minutes)
```
□ Update expected balances in Route 3
□ Review and archive old tasks in Airtable
□ Clear test data from previous month
□ Verify OpenAI credits sufficient
□ Check Make.com billing (should still be $0)
```

### Before Each Close Cycle
```
□ Run full test (see TESTING.md)
□ Verify T001-T015 exist and are "Not Started"
□ Update Google Sheet with current month transactions
□ Clear previous Slack messages (optional)
□ Confirm team knows automation is running
```

---

## Still Stuck?

If you've tried everything in this guide:

1. **Take a break** - Fresh eyes often spot issues
2. **Start from scratch** - Sometimes rebuilding one route is faster
3. **Ask for help** - Open a GitHub issue with details
4. **Check service status:**
   - [Make.com Status](https://status.make.com/)
   - [OpenAI Status](https://status.openai.com/)
   - [Airtable Status](https://status.airtable.com/)

---

**Remember:** Most issues are simple connection or configuration problems. Systematic debugging will find them!

**Last Updated:** October 23, 2025
