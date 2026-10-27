# Kordrex AI Email Autoresponder & Spam Filter (n8n)

An n8n workflow that:

1. **Sorts possible spam automatically.** An AI classifier reads every new inbox email and every website contact-form submission, then labels it `spam`, `auto_reply`, `needs_human` or `no_action`.
2. **Replies to contact-form leads automatically.** When someone fills in the website contact form, they get a personal AI-written reply from your Gmail.
3. **Keeps the conversation going.** When that person replies, the email lands in your inbox, the workflow picks it up and answers in the same Gmail thread. Anything it shouldn't answer is passed to you.

Live workflow: https://rogerawilhelma.app.n8n.cloud/workflow/Lv8ReqQkA4Ijsee7 (published and active)

Works alongside the existing **Speed to Lead - Kordrex Website** workflow, which answers new website form submissions.

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
- [x] Reply prompt uses the real services, process, phone numbers and response time from kordrexsystems.com, and asks new leads for two or three call times and a phone number. If you get a booking link (Calendly, Cal.com, etc.), add it to the `Next step for new leads` line in **AI Reply Writer** → Options → System Message.
- [x] Both AI models run on the `Google Gemini(PaLM) Api account` credential (`models/gemini-3-flash-preview`), not the free OpenAI credits that run out.
- [x] Gmail nodes use the `Gmail account` credential.
- [x] Workflow published.
- [x] Website contact form is connected (see below). No website changes were needed.
- [ ] **Test** by submitting the website form and replying to the email you get back, then check the n8n execution log for both workflows.

## How the website contact form fits in

The form on kordrexsystems.com already posts to your existing **Speed to Lead - Kordrex Website** workflow (`/webhook/kordrex-lead`). That workflow qualifies the lead, sends the first reply from the same Gmail account, alerts you about urgent or sales leads, and logs everything to Data Tables.

This autoresponder takes over from there:

1. Visitor submits the form → **Speed to Lead** emails them within seconds.
2. Visitor replies to that email → the reply lands in the inbox.
3. **This workflow** picks it up within a minute, classifies it, and either replies in the same thread or stars it and alerts you.

The reply prompt knows the lead already got that first email, so it doesn't re-introduce the company.

This workflow also has its own **Website Contact Form** webhook (`/webhook/kordrex-contact-form`). It is not used by the live site; it's there for another site or form you want to connect directly. Don't point the main site at it, or leads will lose Speed to Lead's logging and urgent alerts.

## Prompts

The two AI prompts are also kept as plain files so you can version and tweak them:

- [`prompts/classifier-system-prompt.md`](prompts/classifier-system-prompt.md): spam and intent triage, returns structured JSON.
- [`prompts/reply-writer-system-prompt.md`](prompts/reply-writer-system-prompt.md): writes the actual reply in the Kordrex voice.

`workflow.ts` is the n8n Workflow SDK source the live workflow was built from.

## Tuning

- **Too much reaches you?** Move topics from the `needs_human` list to the `auto_reply` list in the classifier prompt.
- **Want to approve AI replies first?** Change **Reply in Gmail Thread** and **Send First Reply to Lead** to the Gmail *Draft → Create* operation. Replies then wait in Drafts for you to send.
- **Spam threshold:** change `0.8` in the *Spam* rule of **Route by Category**.
