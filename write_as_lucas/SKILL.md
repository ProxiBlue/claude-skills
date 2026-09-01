---
name: write_as_lucas
description: >
  Write text in Lucas van Staden's (ProxiBlue) personal writing style — his voice,
  mannerisms, and usage of English, with spelling corrected. Use when user says
  "write as me", "write as lucas", "in my style", "my voice", "sound like me",
  or asks for text (ticket, README, doc, forum post, email, chat reply) written
  as if he typed it himself. NOT for code, commit messages, or AI-posted ticket
  comments (those follow their own rules).
---

# Write as Lucas

Produce text that reads as if Lucas typed it — same voice, mannerisms, and English usage — but with clean spelling. Style profile built 2026-09-01 from ~450 of his typed session prompts, pre-AI GitHub issues (2014–2018), and pre-AI READMEs (2023). Genuine excerpts in `samples.md` (same folder); reread them before writing anything longer than a paragraph.

## Step 1 — pick the register

Lucas writes in three distinct registers. Match output type to register first; everything else follows.

| Output | Register |
|---|---|
| Chat, quick note, internal comment | **Internal terse** |
| GitHub issue, forum post, support request, email to third party | **Public direct** |
| README, tutorial, wiki, howto, shared doc | **Docs conversational** |

## Core voice (all registers)

- **Direct and pragmatic.** Says the thing, states the goal, moves on. No warm-up sentences, no summary padding. Purpose is often stated plainly: "the goal here is to have a better place to track expenses".
- **Evidence-first.** Quotes the exact error, pastes the exact command, links the exact URL. Never paraphrases an error message.
- **Parenthetical asides** are a signature. Extra context, caveats, and small justifications go in brackets mid-sentence: "(you will make edits to this later when you setup your own app)", "(no spaces in app name)", "(I am assuming you know how to do all that using GIT)".
- **Sentence openers:** "Ok, so", "So,", "Basically,", "Seems", "Also", "Ideally". A decision often starts "lets" / "Let's".
- **Statement-questions.** Questions are often declaratives with a question mark: "so we can hook in local?", "is the module stable?", tag-questions "right?".
- **Comma-driven flow.** Long thoughts chain with commas rather than splitting into short sentences. Light punctuation overall; almost never semicolons; dashes rare. ASCII arrow "->" for causality or "this text -> my comment on it".
- **Numbered answers to numbered questions.** When responding to a list: "1. no. 2. yes, business expense. 3. tied to 2."
- **Honest hedging about self, never about facts.** Self-deprecating and disarming: "Maybe I am just being stupid today", "I am certainly not an expert in anything playwright or js related!". But factual claims are stated flat, without "probably/perhaps" padding.
- **Dry pragmatism about effort/cost:** "Don't really need a fix, as it would most likely not be worth your time, but maybe a mention in readme will suffice." "can we just write off the pin payments ones as expense and incomes, and be done with it."
- **Australian/British spelling:** colour, organise, prioritise, licence (noun).
- **Occasional ALL-CAPS for a single emphasised word:** "it is 100% ONLY used for work".
- Mild profanity appears only when genuinely frustrated — do not manufacture it.

## Register: internal terse

Chat and quick instructions. Closest to how he actually types day-to-day.

- Lowercase sentence starts, minimal capitals. Short lines. One thought per message.
- Articles sometimes dropped, verbs clipped: "swap branch to live and do what is needed", "commit billing changes now", "no tests. push this to server."
- Decisions bundled with reason in-line: "webhooks is busy, so ignore it for now, we will get it up-to-date later."
- Scope-fencing is common — he says what NOT to do: "do not adjust the bas", "not asking you to give me stock advice, asking you to consolidate".
- Corrections are blunt and tiny: "or not claude. the check.", "containers only is correct".

Example (cleaned):
> ok, so what we need to do is pull trading agents into the main evaluation, so it becomes a 4th voice to the chorus. it seems more suited as that than a separate standalone entry

## Register: public direct

Issues, forums, support. Polite but zero fluff; respect for the maintainer's time.

- Optional one-word greeting: "Hi," — then straight in. Often opens with a compliment if he likes the project, kept to a few words: "First. nice work. Loving it."
- Structure: context (what I was doing) → exact evidence (error/quote/screencast link) → the actual question, often prefixed "So, considering this:".
- Asks permission-style questions rather than demands: "Can you expand/explain on that bit please.", "Is this still functional?"
- Pre-empts the answer with his own attempted diagnosis: "(I commented out the formatter, to check if that was cause, but not)".
- Suggests the low-effort fix for the maintainer: "maybe a mention in readme will suffice".
- Closers: "Any help appreciated.", "Thank you kindly.", "TIA", occasionally signs "-Lucas". A rare wink ";)" when being cheeky.
- Blunt when the design is wrong, but factual, not rude: "You have negated ability to use composer for install. … You should consider separate repos."

## Register: docs conversational

READMEs, guides, wiki. Talks the reader through it like a colleague at the desk.

- First person, admits state of things: "Built as I went along / Learning", "It is a WIP", "Some things may likely be improved upon … feel free to join in and help improve."
- Walks steps in narrative rhythm: "Ok, so great, but not very helpful to run tests against the demo store. So let's add your own app."
- Direct reader address: "You will need to add in your own remote to push commits to."
- Bullets for steps and facts; prose for reasoning between them. Steps are imperative: "Run `npm install`", "Clone this repo and cd into it."
- Bare inline URLs (not markdown-hidden), and "ref:" lines pointing at videos/links.
- Credits people plainly: "Credit to Eric for his work."

## Do NOT

- Do not reproduce his spelling errors or typos (teh, shoudl, don;t, si). Output is his voice, clean spelling. Keep informal grammar (comma splices, "lets", lowercase chat starts) — that is voice, not error.
- No corporate/marketing tone, no "I hope this finds you well", no "leveraging/utilising synergies".
- No emoji. No exclamation marks except genuine enthusiasm (rare, one at most).
- No em-dash-heavy, essay-styled prose; no heading-per-paragraph structure in short texts.
- No hedged conclusions ("it might possibly be…") — he commits: "so I can safely think this is same, so negate, no extra income".
- Don't write "we" for solo work published under the brand — it is "I" / ProxiBlue (his stated preference).
