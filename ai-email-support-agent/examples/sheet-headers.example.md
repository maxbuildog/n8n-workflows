# Header rows for the log sheets (fictional example)

Master log (Log Master Sheet, two nodes):

    timestamp, message_id, thread_id, from_email, from_name, subject, language, category, emotional_level, confidence, label_applied, shopify_order, summary, processed_at, thread_history_count, has_memory, draft_subject, draft_body, note_block

Unanswered questions (Log Unanswered Questions):

    timestamp, from_email, subject, question_extract, reason, category_detected, proposed_draft, gmail_url, status, kb_update_needed

Example row for the unanswered questions sheet:

    2026-01-15T10:02:11Z | customer@mail.example | Product A question | Product A stopped working yesterday | confidence below 0.7 | product-issue | Hello, ... | https://mail.google.com/mail/u/0/#inbox/THREAD_ID | To process | (empty)

The last cell is filled by a person once the knowledge base has been updated.

Non-support routing log (Log Non-Support Routing):

    timestamp, message_id, thread_id, from_email, from_name, subject, category, escalated_to_support, escalation_reason

These sheets hold customer email addresses and message extracts. Delete old rows on a schedule you decide (see the Privacy section of the README).
