---
name: linkedin-open-profile-finder
description: Find people at a given company on LinkedIn who Shashank can message for free — either 1st-degree connections or people with LinkedIn's "Open Profile" enabled — and optionally draft + attach his resume to each message. Use this whenever Shashank names a company (and optionally a role) and wants a list of people to cold-message there, wants to find "who I can message for free at X", or wants to identify Open Profile / free-message LinkedIn users at a target company. Outputs a list of names + profile links, and can prep (draft + attach resume to) each message for Shashank's final review — but never hits Send without his explicit go-ahead per message. Requires Claude in Chrome connected to a logged-in LinkedIn session, and requires Shashank's resume PDF to be attached in the current chat session if attaching resumes is requested.
---

# LinkedIn Open Profile Finder

Finds people at a target company who can be messaged on LinkedIn for free (no InMail/Premium required), and hands back a clean list of names + profile URLs. Can optionally go a step further: open each confirmed profile's message box, draft an outreach note, and attach Shashank's resume — but always stops for his explicit confirmation before actually sending each message.

## Inputs needed

- **Company** (required)
- **Role / title** (optional, e.g. "recruiter", "engineering manager", "founding engineer") — narrows the search
- **Target count** (optional, default: 10) — how many valid people to find before stopping
- **Mode** (ask if not specified):
  - "Find only" — just return the list of names + links (no drafting/attaching)
  - "Find + prep messages" — also draft a message and attach the resume for each person, ready for Shashank to review and send himself

## Resume attachment — session requirement

Claude in Chrome's `file_upload` tool can only attach files that have been shared *within the current chat session* (attached by the user, or already present in this session's uploads/outputs folder). It cannot reach arbitrary paths on Shashank's Mac (e.g. `/Users/shashanktyagi/Documents/...`).

So: if Shashank wants messages prepped with the resume attached and no resume has been uploaded yet in this session, ask him to attach the resume PDF to the chat first. This has to be redone each new chat session — there's no persistent file store across sessions.

If the person only gives a company name, default to searching broadly at that company and ask only if the role is genuinely ambiguous — otherwise just proceed.

## Why this exact approach (learned the hard way)

Do **not** rely on the "Message" button appearing on a profile — it appears on almost everyone regardless of whether messaging is actually free. The only two reliable signals are:

1. The gold **"in" LinkedIn Premium badge** next to the person's name. Confirmed Open Profile / freely-messageable people consistently have this badge. If a profile does NOT have the gold badge, skip it immediately — do not waste a click testing it.
2. LinkedIn's own **"Open Profile" spotlight filter** in search (see below) — when available, this does the filtering up front and removes most of the guesswork.

Also do NOT treat "Follow" instead of "Connect" as a signal of free messaging — that just means Creator Mode, which is unrelated to Open Profile and still hits a paywall.

## Step-by-step workflow

1. **Navigate to LinkedIn People search** for the company (and role, if given):
   `https://www.linkedin.com/search/results/people/?keywords=<role>%20<company>`
   or use the search bar directly.

2. **Open "All Filters"** and look for a **"Spotlights"** section. If present, check the **"Open Profile"** option and apply. This restricts results to people who are messageable for free — this alone eliminates most of the manual checking.

3. If the Open Profile filter isn't available/visible for this search, fall back to manual screening:
   - Scan visible profiles in the results list for the **gold "in" Premium badge** next to their name.
   - Skip anyone without the badge — don't open their profile at all.
   - Only open profiles that have the badge.

4. **Verify each badge-holding candidate** by opening their profile and clicking **Message**. Confirm the compose box shows **"Free message | Why?"** (or equivalent free-messaging indicator). This is the only trustworthy confirmation.
   - If it shows this → confirmed, add to the list.
   - If it doesn't (e.g. it prompts to send an InMail / upgrade to Premium) → discard, even if they had the badge.

5. **If mode is "Find only":** close the compose box after verifying — do not draft or attach anything. Just record the person and move on.

6. **If mode is "Find + prep messages":**
   - Draft a short, specific outreach message in the compose box (mention the role/company context Shashank gave; keep it brief and non-generic — no filler).
   - Use `file_upload` to attach Shashank's resume PDF to the message's file input, if the resume has been uploaded to this session (see requirement above). If it hasn't been uploaded yet, pause and ask Shashank to attach it before continuing this step.
   - **Do NOT click Send.** Leave the drafted message + attached resume sitting in the compose box, or take a screenshot/summary of it, and explicitly ask Shashank to confirm before sending. Sending a message on his behalf always needs his real-time go-ahead for that specific message — a earlier "yes, prep messages" does not cover the send action itself.
   - Move to the next candidate only after Shashank has confirmed (sent it himself, or told you to move on) — don't queue up multiple unsent drafts unattended.

7. **Record** each confirmed person's:
   - Full name
   - Current title (if visible)
   - Profile URL
   - (if prepped) whether the message was drafted/attached and sent

8. Stop once you've confirmed the target count (default 10), or once you've reasonably exhausted the search results for that company/role.

## Output format

Present a simple numbered list, nothing fancier:

```
1. [Name] — [Title, if known]
   https://www.linkedin.com/in/[handle]
   [if prepped] Message drafted + resume attached — ready for you to review and send

2. [Name] — [Title, if known]
   https://www.linkedin.com/in/[handle]
```

End with a one-line reminder: for "Find only" mode, these are confirmed free-message profiles for Shashank to message himself; for "Find + prep messages" mode, the drafts are staged but nothing was sent — he still needs to hit Send on each one himself.

## Notes

- If Claude in Chrome isn't connected yet, ask Shashank to connect it before starting (see `list_connected_browsers` / `select_browser`).
- If LinkedIn requires login or hits a captcha/bot-check mid-session, stop and hand control back to Shashank rather than trying to bypass it.
- Never click Send on a LinkedIn message without Shashank's explicit confirmation for that specific message, even in "Find + prep messages" mode. Drafting and attaching the resume is fine to do proactively; sending is not.
- Never use `file_upload` with a path outside this session (e.g. a raw `/Users/...` path Shashank types out) — only use it with files actually present in this chat's shared files.
