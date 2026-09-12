# WhatsApp Restaurant Agent (n8n)

A WhatsApp chatbot that answers customer questions and takes food orders. It reads live stock from Google Sheets before confirming anything, and writes each order back to an Orders sheet.

Built in n8n. Import one JSON file, connect the credentials, run it.

## How it works

```
WhatsApp message
      |
   AI Agent  <-- Google Gemini
      |      <-- Simple Memory (keyed by phone number)
      |
      |      <-- get_inventory  (Sheets: read stock)
      |      <-- get_faqs       (Sheets: read answers)
      |      <-- post_orders    (Sheets: append order)
      |
  Send WhatsApp reply
```

Three things worth pointing out:

- **Memory is keyed by phone number.** Each customer gets their own conversation thread, so two people ordering at the same time do not mix. This is the `sessionKey` on the memory node.
- **The agent checks stock before confirming.** It is told to call `get_inventory` first and refuse the order if the item is out of stock or the quantity is too high.
- **FAQs come from a sheet, not the model.** The prompt forbids inventing answers. Anything not in the sheet gets "a team member will follow up".

## Stack

| Part | Used |
|---|---|
| Workflow engine | n8n |
| Model | Google Gemini |
| Messaging | WhatsApp Business Cloud API |
| Data store | Google Sheets |
| Memory | n8n Simple Memory |

## Google Sheet structure

One spreadsheet, three tabs.

**Inventory**

| Food Item | Quantity | Status |
|---|---|---|
| Chicken Biryani | 20 | In stock |
| Beef Burger | 0 | Out of stock |

**FAQs**

| Question | Answer |
|---|---|
| What are your opening hours? | 11am to 11pm, every day |
| Do you deliver? | Yes, within 5km |

**Orders**

| Customer Name | Food Item | Quantity Ordered | Order Date | Status |
|---|---|---|---|---|

Column names must match the workflow exactly. Some of them currently have a trailing space, which is easy to break. See issue 3 below.

## Setup

You need a WhatsApp Business Cloud API app, a Google Gemini API key, and a Google account for Sheets.

1. Create the spreadsheet with the three tabs above.
2. Add three credentials in n8n: WhatsApp Trigger (OAuth), WhatsApp, Google Gemini, Google Sheets (OAuth2).
3. Import `workflows/whatsapp_agent.json`.
4. Open every node marked `REPLACE_WITH_YOUR_...` and set your own credential, spreadsheet, and WhatsApp phone number ID.
5. Point the Meta webhook at your n8n trigger URL.
6. Activate the workflow and send a test message.

## Test cases

| Message | Expected |
|---|---|
| `What are your opening hours?` | Answer from the FAQs sheet |
| `Do you have parking?` (not in sheet) | Says not sure, team will follow up |
| `2 Chicken Biryani` | Checks stock, confirms, writes to Orders |
| `50 Chicken Biryani` (only 20 left) | Refuses, says unavailable |
| `Beef Burger` (out of stock) | Refuses politely |

## Known issues

**1. Stock is never reduced.** The agent reads inventory and appends an order, but nothing decrements the Quantity column. Ten customers can all order the last item. This is the most important fix: add a Sheets update step after the order is written.

**2. The AI decides the order Status.** The `post_orders` node fills Status from `$fromAI`, so the model can write anything there. The prompt says to use "Pending", but nothing enforces it. Hardcode the value to `Pending` in the node instead.

**3. Column names have trailing spaces.** `Customer Name `, `Food Item `, `Order Date `, `Status `. Anyone tidying the sheet will silently break the mapping. Rename them without spaces in both the sheet and the node.

**4. Non-text messages crash the workflow.** The agent reads `messages[0].text.body`. If a customer sends an image, voice note, sticker, or location, that field does not exist and the run fails. Add a filter before the agent that only passes text messages, and reply with a short "text only" message otherwise.

**5. No error handling.** If the Sheets API fails or Gemini times out, the customer gets no reply at all and never finds out why. Add an error branch that sends a fallback message.

**6. Order Date is a full timestamp.** `{{ $now }}` writes an ISO datetime with timezone, not a date. Format it if the sheet is meant to hold dates.

**7. The tools have no descriptions.** `get_inventory` and `get_faqs` give the model only their node names to work from. Adding a one-line description to each makes tool selection more reliable.

## Note on the JSON

The export has been scrubbed. The WhatsApp phone number ID, Google Sheet ID, credential IDs, webhook IDs, and the n8n instance ID were replaced with placeholders, and `active` was set to `false`.

n8n exports do not contain API keys, but they do contain IDs tied to one live account. A WhatsApp phone number ID in a public repo is worth removing.

## License

MIT
