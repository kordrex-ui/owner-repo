# Kordrex AI Email Autoresponder & Spam Filter (n8n)

An n8n workflow that:

1. **Sorts possible spam automatically.** An AI classifier reads every new inbox email and every website contact-form submission, then labels it `spam`, `auto_reply`, `needs_human` or `no_action`.
2. **Replies to contact-form leads automatically.** When someone fills in the website contact form, they get a personal AI-written reply from your Gmail.
3. **Keeps the conversation going.** When that person replies, the email lands in your inbox, the workflow picks it up and answers in the same Gmail thread. Anything it shouldn't answer is passed to you.

Live workflow: https://rogerawilhelma.app.n8n.cloud/workflow/Lv8ReqQkA4Ijsee7 (published and active)

## How it flows

```
Website Contact Form (webhook) ─► Normalize Form ─┐
                                                  ├─► AI Spam & Intent Classifier ─► Route by Category
New Inbox Email (Gmail, 1 min) ─► Prepare Inbox ──┘
                                                       ├─ Spam (≥80% sure) ─► Gmail Spam folder (form spam is dropped)
                                                       ├─ Auto Reply ───────► AI Reply Writer ─► form: send new email
                                                       │                                         inbox: reply in thread + mark read
                                                       ├─ No Action ────────► mark read (newsletters, receipts, alerts)
                                                       └─ Needs Human ──────► email alert to owner + star the message
```

## Safety guards

- **Spam is never deleted.** It goes to Gmail's Spam folder, where you can recover it for 30 days. Anything under 80% spam confidence goes to you instead.
- **No reply loops.** Senders that look automated (no-reply addresses, `Auto-Submitted`, `Precedence: bulk`, `List-Unsubscribe` and auto-reply headers) never get an AI reply. A sender who has written more than 5 times in 24 hours is handed to you.
- **Conservative by default.** If the classifier is unsure between auto-reply and human, it chooses human. The reply prompt forbids inventing prices, timelines, guarantees or commitments.
- **Each email is handled once.** The Gmail trigger only watches `in:inbox is:unread -is:starred -from:me`. Every branch either marks the email read, moves it to Spam, or stars it.

## Setup status

- [x] Owner alerts go to `kordrex@kordrexsystems.com` (**Alert Owner: Needs Reply** node).
- [x] Reply prompt uses https://kordrexsystems.com and asks new leads for two or three call times and a phone number. If you get a booking link (Calendly, Cal.com, etc.), add it to the `Next step for new leads` line in **AI Reply Writer** → Options → System Message.
- [x] Both AI models run on the `Google Gemini(PaLM) Api account` credential (`models/gemini-3-flash-preview`), not the free OpenAI credits that run out.
- [x] Gmail nodes use the `Gmail account` credential.
- [x] Workflow published.
- [ ] **Connect the website contact form** (see below). The site at kordrexsystems.com was deployed to Netlify by drag-and-drop, so its source isn't in a repo; add the form to your site files and re-upload.
- [ ] **Test** by submitting the form once and emailing yourself from another address, then check the n8n execution log.

## Connecting your website contact form

Production webhook URL (works the workflow is published):

```
POST https://rogerawilhelma.app.n8n.cloud/webhook/kordrex-contact-form
```

It accepts JSON or form fields: `name`, `email`, `phone`, `company`, `message`, plus a hidden `website` honeypot field. Real visitors leave the honeypot empty; bots usually fill it in.

Drop-in example:

```html
<form id="contact-form">
  <input name="name" placeholder="Your name" required>
  <input name="email" type="email" placeholder="Email" required>
  <input name="phone" placeholder="Phone (optional)">
  <input name="company" placeholder="Company (optional)">
  <textarea name="message" placeholder="How can we help?" required></textarea>
  <!-- Honeypot: hidden from humans, bots fill it in -->
  <input name="website" tabindex="-1" autocomplete="off" style="position:absolute;left:-9999px" aria-hidden="true">
  <button type="submit">Send</button>
</form>
<script>
  document.getElementById('contact-form').addEventListener('submit', async (e) => {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(e.target));
    await fetch('https://rogerawilhelma.app.n8n.cloud/webhook/kordrex-contact-form', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    e.target.innerHTML = '<p>Thanks! We just emailed you, so check your inbox.</p>';
  });
</script>
```

If your site builder (Wix, Webflow, WordPress, Framer, etc.) has its own form, point its webhook or "send to URL" option at the same address and keep the field names above.

## Prompts

The two AI prompts are also kept as plain files so you can version and tweak them:

- [`prompts/classifier-system-prompt.md`](prompts/classifier-system-prompt.md): spam and intent triage, returns structured JSON.
- [`prompts/reply-writer-system-prompt.md`](prompts/reply-writer-system-prompt.md): writes the actual reply in the Kordrex voice.

`workflow.ts` is the n8n Workflow SDK source the live workflow was built from.

## Tuning

- **Too much reaches you?** Move topics from the `needs_human` list to the `auto_reply` list in the classifier prompt.
- **Want to approve AI replies first?** Change **Reply in Gmail Thread** and **Send First Reply to Lead** to the Gmail *Draft → Create* operation. Replies then wait in Drafts for you to send.
- **Spam threshold:** change `0.8` in the *Spam* rule of **Route by Category**.
