# Shopify sGTM Tracking Setup Analysis

## Current Setup Overview

### Components

**Events:**
- `E_sc` - Event store cookies from Shopify customer events (subscribed to `checkout_started` & `checkout_completed`)

**Variables:**
- `C_tid` - Check Transaction ID (Stape Store Lookup variable)
  - Document path: `purchase_{{purchaseOrderId}}`

**Tags:**
1. `C_cw` - Checkout Cookie Writer (Stape Store Writer)
   - **Trigger**: E_sc event
   - **Condition**: C_tid does NOT equal true
   - **Writes**: taboola_cid, outbrain_cid (whichever available)

2. `P_cw` - Purchase Cookie Writer (Stape Store Writer)
   - **Trigger**: E_sc event
   - **Condition**: C_tid EQUALS true
   - **Writes**: taboola_cid, outbrain_cid, transaction_id

3. `P_SP` - Purchase Store Tracking (Stape Store Writer)
   - **Trigger**: GA4 purchase event
   - **Writes**:
     - tracked = "true"
     - email = {{email}}
     - transaction_id = {{purchase_order_id}}
     - total_value = {{total_purchase_value}}

---

## Event Flow Sequence

```
1. Web Container: begin_checkout event
   ↓
2. sGTM GA4 Client receives begin_checkout
   ↓
3. Taboola/Outbrain checkout tags fire
   ↓
4. Shopify Customer Event: store_cookie event (E_sc)
   - Contains: taboola_cid OR outbrain_cid
   ↓
5. C_cw tag fires (C_tid != true)
   - Stores click identifier to Stape Store
   ↓
... Purchase happens ...
   ↓
6. Shopify Purchase Webhook fires
   ↓
7. Web Container: purchase event (1 min delay)
   ↓
8. sGTM GA4 Client receives purchase
   ↓
9. P_SP tag fires FIRST
   - Stores: tracked, email, transaction_id, total_value
   ↓
10. Taboola/Outbrain purchase tags fire
   ↓
11. Shopify Customer Event: store_cookie event (E_sc)
    ↓
12. P_cw tag SHOULD fire (C_tid == true)
    - Stores: taboola_cid, outbrain_cid, transaction_id
```

---

## Identified Issues

### Issue #1: P_cw Tag Fires with Undefined transaction_id

**Symptom:**
- P_cw tag sometimes fires even though transaction_id is undefined
- This happens despite P_SP tag successfully logging transaction_id to Stape Store

**Root Cause:**
- **Race Condition**: P_cw fires before Stape Store Lookup (C_tid) can retrieve the newly written data from P_SP
- **Timing Issue**: The Stape Store write operation from P_SP may not complete before C_tid lookup executes
- **Asynchronous Operations**: HTTP requests to Stape Store API are asynchronous

**Evidence:**
```
Timeline:
t=0: P_SP fires → writes to Stape Store (async HTTP request)
t=1: store_cookie event arrives
t=2: C_tid lookup executes → Stape Store write may not be complete yet
t=3: P_cw evaluates trigger condition → gets undefined/stale data
```

### Issue #2: Not Always Reliable - Events Don't Always Trigger

**Symptom:**
- Purchase & checkout events don't always trigger outgoing request URLs
- Tracking is inconsistent (not 100% reliable)

**Potential Root Causes:**

1. **Event Data Availability**
   - Required variables (purchaseOrderId, email, etc.) may not always be present
   - Shopify customer events may have inconsistent data structure

2. **Trigger Condition Logic**
   - C_tid lookup returning inconsistent values (true, "true", undefined, null, empty string)
   - Boolean comparison may not work as expected with Stape Store Lookup

3. **Document Path Issues**
   - `purchase_{{purchaseOrderId}}` variable may not resolve correctly
   - purchaseOrderId may be undefined when C_tid evaluates

4. **Stape Store Collection Names**
   - Different tags may be writing to/reading from different collections
   - Missing collection name specification

5. **Network/API Issues**
   - Stape Store API requests may timeout or fail
   - No error handling for failed writes

---

## Critical Configuration Gaps

### Gap #1: Missing Document ID Strategy

**Problem:**
- C_cw and P_cw don't specify which document to write to
- May create duplicate documents instead of updating existing ones
- No consistent document ID scheme for linking checkout → purchase

**Impact:**
- Data fragmentation
- Cannot reliably look up checkout data during purchase

### Gap #2: No Proper Sequencing

**Problem:**
- P_SP and P_cw may fire simultaneously or in wrong order
- No guaranteed execution order ensures P_SP completes before P_cw fires

**Impact:**
- Race condition causing undefined transaction_id in P_cw

### Gap #3: Inconsistent Collection Names

**Problem:**
- Not clear if all tags use the same Stape Store collection
- C_tid lookup may be querying wrong collection

**Impact:**
- Data not found even when written successfully

### Gap #4: Boolean Comparison Issue

**Problem:**
- Trigger condition: `C_tid != true` and `C_tid == true`
- Stape Store Lookup may return string "true" instead of boolean true
- JavaScript comparison: `"true" == true` evaluates to false

**Impact:**
- Wrong tag fires at wrong time

---

## Recommended Fixes

### Fix #1: Implement Consistent Document ID Strategy

**Solution:**
Use a deterministic document ID based on user identifier (not transaction ID)

**Implementation:**

1. **Create new variable**: `User_Session_ID`
   - Type: Event Data variable or Cookie variable
   - Path: Use Shopify customer ID, session ID, or ga_session_id
   - Fallback: Use client_id from GA4 event

2. **Update C_cw tag**:
   ```
   Document ID: {{User_Session_ID}}_checkout
   Collection Name: tracking_data
   Custom Data:
     - name: taboola_cid, value: {{taboola_cid}}
     - name: outbrain_cid, value: {{outbrain_cid}}
     - name: checkout_timestamp, value: {{timestamp}}
   Merge document keys: ✓ (checked)
   ```

3. **Update P_cw tag**:
   ```
   Document ID: {{User_Session_ID}}_checkout
   Collection Name: tracking_data
   Custom Data:
     - name: transaction_id, value: {{transaction_id}}
     - name: purchase_timestamp, value: {{timestamp}}
   Merge document keys: ✓ (checked)
   ```

4. **Update P_SP tag**:
   ```
   Document ID: purchase_{{transaction_id}}
   Collection Name: tracking_data
   Custom Data:
     - name: tracked, value: true
     - name: email, value: {{email}}
     - name: transaction_id, value: {{transaction_id}}
     - name: total_value, value: {{total_value}}
     - name: user_session_id, value: {{User_Session_ID}}
   Merge document keys: ✗ (unchecked - overwrite)
   ```

5. **Update C_tid variable**:
   ```
   Type: Stape Store Lookup
   Collection Name: tracking_data
   Document Path: purchase_{{transaction_id}}
   Field: tracked
   ```

**Benefits:**
- Consistent document structure
- Checkout and purchase data linked via document ID
- No duplication

---

### Fix #2: Fix Boolean Comparison Logic

**Solution:**
Check for existence of data instead of boolean comparison

**Implementation:**

1. **Update C_tid variable name** to `Purchase_Tracked_Status`

2. **Create trigger for C_cw**:
   ```
   Trigger Type: Custom Event
   Event Name: store_cookie (E_sc)

   This trigger fires on: Some Custom Events
   Conditions:
     - Event Name equals store_cookie
     - Purchase_Tracked_Status does not equal true
     - Purchase_Tracked_Status does not equal "true"

   OR use:
     - {{Purchase_Tracked_Status}} is undefined
   ```

3. **Create trigger for P_cw**:
   ```
   Trigger Type: Custom Event
   Event Name: store_cookie (E_sc)

   This trigger fires on: Some Custom Events
   Conditions:
     - Event Name equals store_cookie
     - Purchase_Tracked_Status equals true

   OR use:
     - {{Purchase_Tracked_Status}} is defined
     - {{Purchase_Tracked_Status}} is not empty
   ```

**Alternative (Better) Approach:**

Create custom JavaScript variable to normalize boolean:

```javascript
// Variable Name: Is_Purchase_Tracked
function() {
  var status = {{Purchase_Tracked_Status}};

  // Check if purchase is tracked
  if (status === true || status === "true" || status === 1 || status === "1") {
    return true;
  }

  return false;
}
```

Then use trigger conditions:
- C_cw: `{{Is_Purchase_Tracked}} equals false`
- P_cw: `{{Is_Purchase_Tracked}} equals true`

---

### Fix #3: Handle Race Condition with Tag Sequencing

**Solution:**
Use Tag Sequencing to ensure P_SP completes before P_cw evaluates

**Implementation:**

**Option A: Use Tag Sequencing (Recommended)**

1. Configure P_SP tag:
   ```
   Tag Type: Stape Store Writer
   Firing Triggers: GA4 purchase event
   [No sequencing configured here]
   ```

2. Create new intermediate tag: `Wait_For_Store_Write`
   ```
   Tag Type: Custom HTML tag (no-op)
   Code: <script>/* Wait for Stape Store write */</script>

   Firing Triggers: GA4 purchase event

   Tag Sequencing:
   Setup Tag: P_SP (fire before this tag)
   Pause Duration: None
   ```

3. Configure P_cw tag:
   ```
   Tag Type: Stape Store Writer
   Firing Triggers: store_cookie (E_sc) + {{Is_Purchase_Tracked}} equals true

   Tag Sequencing:
   Setup Tag: Wait_For_Store_Write (fire before this tag)
   Pause Duration: None
   ```

**Option B: Add Delay in P_cw Trigger (Not Recommended)**

Add event parameter check to ensure data is ready:
```
Trigger Conditions:
  - Event Name equals store_cookie
  - {{Is_Purchase_Tracked}} equals true
  - {{transaction_id}} is defined (additional safety check)
```

**Option C: Use Different Event (Best Solution)**

Instead of relying on C_tid lookup, check transaction_id directly from event data:

1. Modify trigger logic:
   - C_cw: Fire when `{{Event.transaction_id}}` is undefined
   - P_cw: Fire when `{{Event.transaction_id}}` is defined

2. This eliminates the need for Stape Store Lookup during trigger evaluation

---

### Fix #4: Add Error Handling and Logging

**Solution:**
Enable comprehensive logging to debug issues

**Implementation:**

1. **Enable logging in all Stape Store Writer tags**:
   ```
   Logging Settings:
   ✓ Console Logging
     - Log Type: Both (request and response)
     - Console Log Mode: Always

   ✓ BigQuery Logging
     - Project ID: [your-project]
     - Dataset ID: gtm_server_logs
     - Table ID: stape_store_logs
   ```

2. **Create monitoring variable**: `Transaction_ID_Available`
   ```javascript
   function() {
     var tid = {{Event.transaction_id}} || {{transaction_id}};
     return tid !== undefined && tid !== null && tid !== '';
   }
   ```

3. **Add data layer events for debugging**:
   ```javascript
   // Custom HTML tag to log state
   <script>
   console.log('Store Cookie Event Received', {
     transaction_id: {{transaction_id}},
     purchase_tracked: {{Is_Purchase_Tracked}},
     taboola_cid: {{taboola_cid}},
     outbrain_cid: {{outbrain_cid}}
   });
   </script>
   ```

---

### Fix #5: Restructure Event Flow

**Solution:**
Separate concerns and use proper event-driven architecture

**New Architecture:**

```
Event Flow:
1. begin_checkout (Web → sGTM)
   → Store session data with click identifiers

2. store_cookie (checkout_started) (Shopify → sGTM)
   → Update session document with additional data

3. purchase (Web → sGTM)
   → P_SP fires: Create purchase document
   → Trigger custom internal event: "purchase_logged"

4. purchase_logged (Internal sGTM event)
   → Lookup session document
   → Enrich purchase with session data
   → Fire Taboola/Outbrain with complete data

5. store_cookie (checkout_completed) (Shopify → sGTM)
   → Final enrichment if needed
```

**Implementation Steps:**

1. **Create new tag**: `Purchase_Enrichment_Tag`
   ```
   Tag Type: Custom HTML or Stape Store Writer
   Trigger: GA4 purchase event

   Purpose:
   - Read session document ({{User_Session_ID}}_checkout)
   - Retrieve taboola_cid, outbrain_cid
   - Write enriched purchase document
   - Set data layer variables for downstream tags
   ```

2. **Modify Taboola/Outbrain tags**:
   ```
   Trigger: GA4 purchase event

   Tag Sequencing:
   Setup Tag: Purchase_Enrichment_Tag

   Use enriched variables:
   - {{Enriched_Taboola_CID}}
   - {{Enriched_Outbrain_CID}}
   - {{transaction_id}}
   ```

3. **Simplify C_cw and P_cw**:
   - C_cw: Only write session data at checkout
   - P_cw: Remove or repurpose for backup only

---

## Implementation Checklist

### Phase 1: Data Layer & Variables
- [ ] Create `User_Session_ID` variable
- [ ] Create `Is_Purchase_Tracked` variable
- [ ] Create `Transaction_ID_Available` variable
- [ ] Update `C_tid` to `Purchase_Tracked_Status`
- [ ] Add proper fallbacks for all event data variables

### Phase 2: Tag Configuration
- [ ] Update C_cw with Document ID: `{{User_Session_ID}}_checkout`
- [ ] Update P_cw with Document ID: `{{User_Session_ID}}_checkout`
- [ ] Update P_SP with Document ID: `purchase_{{transaction_id}}`
- [ ] Set all tags to use same Collection Name: `tracking_data`
- [ ] Enable "Merge document keys" for C_cw and P_cw
- [ ] Disable "Merge document keys" for P_SP

### Phase 3: Trigger Logic
- [ ] Update C_cw trigger: `{{Is_Purchase_Tracked}} equals false`
- [ ] Update P_cw trigger: `{{Is_Purchase_Tracked}} equals true`
- [ ] Add additional safety check: `{{transaction_id}} is defined` for P_cw

### Phase 4: Tag Sequencing
- [ ] Create `Wait_For_Store_Write` intermediate tag
- [ ] Configure P_SP → Wait_For_Store_Write → P_cw sequence
- [ ] OR implement Purchase_Enrichment_Tag architecture

### Phase 5: Logging & Monitoring
- [ ] Enable Console Logging on all Stape Store Writer tags
- [ ] Enable BigQuery Logging (optional but recommended)
- [ ] Add debug Custom HTML tags for critical events
- [ ] Test in Preview mode with store_cookie events

### Phase 6: Testing
- [ ] Test checkout flow with Taboola traffic
- [ ] Test checkout flow with Outbrain traffic
- [ ] Test purchase flow end-to-end
- [ ] Verify Stape Store documents are created correctly
- [ ] Verify lookup variables return expected values
- [ ] Check outgoing requests to Taboola/Outbrain

---

## Testing Strategy

### Test Case 1: Checkout with Taboola CID
```
1. Send begin_checkout with taboola_cid
2. Send store_cookie (checkout_started) with taboola_cid
3. Verify C_cw fires
4. Check Stape Store document: {{User_Session_ID}}_checkout
   Expected: {taboola_cid: "xxx", outbrain_cid: null, checkout_timestamp: "xxx"}
```

### Test Case 2: Purchase with Existing Session
```
1. Complete Test Case 1
2. Send purchase event with transaction_id
3. Verify P_SP fires first
4. Check Stape Store document: purchase_{{transaction_id}}
   Expected: {tracked: true, email: "xxx", transaction_id: "xxx", total_value: "xxx"}
5. Send store_cookie (checkout_completed)
6. Verify P_cw fires (not C_cw)
7. Check session document updated with transaction_id
```

### Test Case 3: Purchase Without Prior Checkout Data
```
1. Send purchase event directly (no prior checkout)
2. Verify P_SP fires
3. Send store_cookie event
4. Verify proper tag fires based on trigger logic
5. Check for errors in console logs
```

---

## Quick Wins (Implement First)

1. **Add Collection Name** to all Stape Store tags: `tracking_data`
2. **Fix Boolean Logic** using `Is_Purchase_Tracked` custom variable
3. **Add Transaction ID Check** to P_cw trigger: `{{transaction_id}} is defined`
4. **Enable Console Logging** on all tags for debugging
5. **Test in Preview Mode** with real events to see exact firing sequence

---

## Expected Improvements

After implementing fixes:
- ✓ 100% tag firing reliability (given proper event data)
- ✓ No race conditions between P_SP and P_cw
- ✓ Consistent document IDs for data linking
- ✓ Proper boolean logic evaluation
- ✓ Better debugging with comprehensive logs
- ✓ Scalable architecture for future enhancements

---

## Additional Recommendations

### 1. Use Firestore Instead of Stape Store (Advanced)
For mission-critical tracking with higher reliability:
- Lower latency reads/writes
- Better consistency guarantees
- Real-time updates
- More robust error handling

### 2. Implement Idempotency
Add deduplication logic to prevent duplicate writes:
```
Custom Data:
  - name: event_id, value: {{event_id}}
  - name: write_timestamp, value: {{timestamp}}
```

Check event_id before processing to prevent duplicates.

### 3. Add Fallback Mechanism
If Stape Store write fails, queue event for retry:
- Use client-side cookie as backup
- Implement retry logic in tag
- Send to alternative endpoint

### 4. Monitoring Dashboard
Set up BigQuery + Data Studio dashboard to monitor:
- Tag firing rates
- Stape Store API response times
- Error rates
- Missing data incidents

---

## Questions to Clarify

Before final implementation, please provide:

1. **Container JSON**: The actual sGTM container export for detailed analysis
2. **Preview Request JSON**: Example events showing exact data structure
3. **Variable Definitions**: How are {{purchaseOrderId}}, {{transaction_id}}, {{email}} defined?
4. **Collection Names**: What collection names are currently used (if any)?
5. **Error Logs**: Any console errors or failed HTTP requests in Preview mode?
6. **Traffic Volume**: What's the expected event volume (QPS)?
7. **Taboola/Outbrain Tags**: How do these tags read the click identifiers?

---

## Next Steps

1. **Immediate**: Implement Quick Wins (above)
2. **Short-term**: Implement Fixes #1, #2, #3 (document ID, boolean logic, sequencing)
3. **Medium-term**: Restructure event flow (Fix #5) if issues persist
4. **Long-term**: Add monitoring, alerting, and advanced error handling

Please share the container JSON and preview request data so I can provide more specific configuration updates.
