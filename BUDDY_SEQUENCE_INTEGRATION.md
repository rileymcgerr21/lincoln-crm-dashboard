# Outreach Sequence — Bot Integration Guide

The dashboard side of the sequence engine is shipped. It tracks every lead's
`next_action`, `next_action_due_at`, and `sequence_step` on the `leads` table
itself. Riley + I can advance steps manually from the side panel.

To make the cadence fully automatic, your bot needs to do **two things**:

---

## 1. After your bot sends an SMS

Whenever Twilio confirms a successful T1/T2/T3 SMS send, run this update.
Replace `$LEAD_ID`, `$STEP`, `$BODY`, `$TWILIO_SID`, `$FROM`, `$TO`.

The cadence:

```
Step 0 (queued, just imported)
  → bot sends T1 SMS
  → STEP becomes 1, next_action='call', due in 24h
Step 5 (after a 1st call + 2nd call + letter)
  → bot sends T2 SMS
  → STEP becomes 5, next_action='call', due in 72h
Step 7 (after a 3rd call)
  → bot sends T3 SMS
  → STEP becomes 7, next_action=NULL (sequence done)
```

### SQL (run in one transaction per send)

```sql
-- 1. Log the touch
INSERT INTO touches (
  lead_id, direction, channel, body, status, sent_at,
  provider, provider_message_id, from_number, to_number, sequence_step
) VALUES (
  $LEAD_ID, 'outbound', 'sms', $BODY, 'sent', NOW(),
  'twilio', $TWILIO_SID, $FROM, $TO,
  -- sequence_step = the step that just completed (0 for T1, 4 for T2, 6 for T3)
  $STEP_BEFORE_ADVANCE
);

-- 2. Advance the lead. Use the table below to fill in the fields.
UPDATE leads SET
  current_stage      = $NEW_STAGE,        -- 't1_sent' | 't2_sent' | 't3_sent'
  sequence_step      = $NEW_STEP,         -- step you just moved INTO
  next_action        = $NEXT_ACTION,      -- 'call' | NULL
  next_action_due_at = $NEXT_DUE,         -- NOW() + interval, or NULL
  last_action_at     = NOW()
WHERE id = $LEAD_ID;
```

### Step transition table

| Bot sends | step before | step after | new current_stage | next_action | next_action_due_at |
|---|---|---|---|---|---|
| T1 SMS | 0 | **1** | `t1_sent` | `call` | `NOW() + interval '24 hours'` |
| T2 SMS | 4 | **5** | `t2_sent` | `call` | `NOW() + interval '72 hours'` |
| T3 SMS | 6 | **7** | `t3_sent` | `NULL` | `NULL` |

**Don't send if `current_stage` is already `replied`, `opted_out`, `do_not_contact`, `closed_won`, `closed_lost`, or `bad_number`.** A trigger on `touches` already halts the sequence when an inbound message arrives — but defensive check is good.

To find leads ready for a T1 send right now:

```sql
SELECT id, primary_phone, owner_full_name, property_address
FROM leads
WHERE current_stage = 'queued'
  AND sequence_step = 0
  AND next_action = 'send_t1'
  AND next_action_due_at <= NOW()
  AND primary_phone IS NOT NULL
  AND is_opted_out = false
  AND is_dnc = false
  AND deleted_at IS NULL
ORDER BY next_action_due_at ASC
LIMIT 50;
```

For T2 (step 4 → 5):

```sql
WHERE next_action = 'send_t2' AND next_action_due_at <= NOW() AND ...
```

For T3 (step 6 → 7): `WHERE next_action = 'send_t3' AND ...`

---

## 2. When Twilio's webhook receives an inbound message

Just insert the touch row with `direction='inbound'`. The trigger we installed
(`tr_touches_halt_on_reply`) will:

- Set `current_stage = 'replied'`
- Clear `next_action` and `next_action_due_at` (sequence halts)
- Add tag `needs-reply`

So all you need:

```sql
INSERT INTO touches (
  lead_id, direction, channel, body, status, created_at,
  provider, provider_message_id, from_number, to_number
) VALUES (
  $LEAD_ID, 'inbound', 'sms', $BODY, 'received', NOW(),
  'twilio', $TWILIO_SID, $FROM, $TO
);
```

That's it. Riley sees the lead in the **Inbox** sidebar item the next time he refreshes.

---

## 3. (Optional) Twilio status callback

When Twilio fires the status callback (delivered / failed / undelivered):

```sql
UPDATE touches
SET status = $STATUS,
    delivered_at = CASE WHEN $STATUS = 'delivered' THEN NOW() ELSE delivered_at END,
    failed_at    = CASE WHEN $STATUS IN ('failed','undelivered') THEN NOW() ELSE failed_at END,
    failure_reason = $FAILURE_REASON
WHERE provider_message_id = $TWILIO_SID;
```

If `failed`, you might also want to set the lead's `current_stage = 'bad_number'`
so it stops trying. The dashboard will show that lead as Closed.

---

## What the dashboard does (so you don't double-handle it)

- **Calls and letters**: Riley/buddy press buttons in the side panel. Those advance the sequence and log a touch row with `channel='call'` or `channel='letter'`. Don't try to handle these in the bot — they're human actions.
- **Outbound SMS that buddy's bot sends**: only the bot updates these. Don't let the dashboard send SMS.
- **Replies**: trigger handles them. Just insert the inbound touch row.

---

## Quick test plan

1. Pick one lead. From the dashboard, click "Start sequence" in the side panel. The lead's `next_action` becomes `send_t1`, due now.
2. Your bot picks it up via the SQL above. Sends T1. Updates the lead per the table.
3. Dashboard shows: `t1_sent`, badge "📞 Call · in 24h", appears in **Today's Tasks** in 24h.
4. 24h later, Riley sees it in Today's Tasks. Clicks "Mark called". Sequence advances to step 2.
5. … and so on.

If the lead replies before any of this completes, the trigger halts everything and Riley sees them in **Inbox**.

---

## Schema reference (already deployed)

```
leads.next_action         TEXT          -- 'send_t1' | 'call' | 'send_letter' | 'send_t2' | 'send_t3' | NULL
leads.next_action_due_at  TIMESTAMPTZ
leads.sequence_name       TEXT          -- 'standard_4_step' (only one for now)
leads.sequence_step       INT           -- 0..7
leads.sequence_started_at TIMESTAMPTZ
leads.last_action_at      TIMESTAMPTZ

trigger: tr_touches_halt_on_reply  AFTER INSERT ON touches
  -- if direction='inbound', clears sequence + sets stage='replied' + adds 'needs-reply' tag
```

That's it. Ping Riley with questions.
