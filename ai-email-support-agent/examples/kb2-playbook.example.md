# KB2: support playbook (fictional example, internal, email use only)

Each numbered section is one or more chunks embedded in `kb_playbook_embeddings`. The agent is told to consult this base before writing a draft and before choosing the label. Reply templates should be replies a human reviewer has already approved: copy real approved answers here, with the customer-specific parts replaced by [PLACEHOLDERS].

## 1. Tone rules
- Polite and plain, "we" for the company, never "I".
- 60 to 120 words, one idea per sentence, one request per email.
- No em dashes, no emoji on red items, at most one "!" and only for good news.
- Closing line before the signature: "Please let us know if we can help with anything else."
- The signature and the AI disclosure line are added by the workflow: do not write them in templates.

## 2. Goodwill scale (the reviewer decides, the agent only suggests)
| Situation | Gesture | Label |
| --- | --- | --- |
| Short delay, calm customer | none | green or yellow |
| Long delay, unhappy customer | code EXAMPLE10 (10% off a next order) | yellow |
| Confirmed fault within the guarantee period | replacement, plus code EXAMPLE20 if the customer is unhappy | yellow |
| Anything above 20% | never | n/a |

Codes listed here must also be listed in `ALLOWED_PROMO` in Parse agent output.

## 3. Reviewer note format
- CONTEXT: one factual paragraph.
- ACTIONS TO DO (reviewer): bullet list in plain language ("ship a replacement to the address on file").
- LINKS: order number, tracking link.

## 4. Label rules
- Green: thanks, simple information, no commitment.
- Yellow: any refund, reshipment, replacement, cancellation, address change or code to create.
- Blue: order not found, language other than the default, confidence below the threshold, question not covered by KB1.
- Red: legal threat, safety or health concern, personal data request, payment dispute, chat tool alert, very unhappy customer.
- Partnerships: business proposals, no draft.

## 5. Escalation handling
- Legal threat: sober draft, ask for order number and circumstances, no admission, no promise. A human replies.
- Safety or health concern: express concern, ask for order number and details, say a team member will follow up personally. No advice, no discount, no reshipment before review.
- Personal data request: acknowledge only. No personal data in the reply. The reviewer verifies identity and follows the data request procedure.

## Reply template: parcel in transit (approved)
Situation: the customer asks where the parcel is, tracking shows it in transit.
Label: green

    Hello [FIRST NAME],

    Thank you for your message.

    Your order [#ORDER NUMBER] has been in transit since [DATE] and is within the expected delivery time. You can follow it here: [TRACKING LINK].

    Please let us know if we can help with anything else.

## Reply template: product problem under guarantee (approved)
Situation: the customer reports that a product does not work as expected, the order is within the guarantee period.
Label: yellow (the reviewer decides on the replacement)

    Hello [FIRST NAME],

    Thank you for your message. We are sorry for the inconvenience.

    Your order dates from [DATE]. Could you send us a photo or a short video showing [THE PROBLEM]? You can attach it to your reply.

    Once we receive it, we will come back to you with a solution.

Reviewer note: "Replacement to validate once the photo or video is received. Confirm the shipping address."

## Reply template: personal data request (approved)
Situation: the customer asks for a copy or the deletion of their personal data.
Label: red

    Hello [FIRST NAME],

    Thank you for your message. We have received your request about your personal data.

    A member of our team will handle it and come back to you within [N] days. We may ask you to confirm your identity first.

Reviewer note: "Verify identity, then follow the data request procedure. Do not send any data by email before that."
