# Guarda de Sião — Company Website (Portfolio Copy)

**[Live demo →](https://laylavianacruz.github.io/guardadesiao-portfolio/)**

This is a public portfolio copy of [guardadesiao.com](https://guardadesiao.com), the production website for Guarda de Sião, an electronic security company serving premium residential condominiums in Fortaleza, Brazil. Real business contact information (WhatsApp, email, CNPJ) is kept as-is since it's already public — but the AI chat assistant ("Sião") is fully mocked here: it responds with local, hard-coded demo replies instead of connecting to the company's real n8n automation webhook. See **Architecture & Security** below for details.

## The Problem

Condominium security is a high-trust, high-stakes sale: prospective clients (síndicos, property managers, residents' associations) need to quickly understand a company's technical credibility, see real installed work, and get a fast response — without friction like phone tag or slow email threads. The original site also needed to run in three languages (Portuguese, English, Spanish) for an increasingly international client base in Ceará's premium real estate market.

## The Solution

A single-page, trilingual marketing site that combines:

- A clear services breakdown (CCTV, access control, automatic gates, electric fencing, unmanned reception, monitoring centers)
- Social proof (partner brands, client condominiums, a portfolio gallery)
- A low-friction contact path: a form that pre-fills a WhatsApp message with the visitor's details, so the conversation starts on a channel people already use
- **Sião**, an AI-powered chat assistant embedded in the page that handles first-line inquiries 24/7 and hands off to the human team when needed

## Key Features

- **Full i18n system** — all copy lives in a single `translations` object (PT/EN/ES); switching language re-renders every `data-i18n`-tagged element and persists the choice in `localStorage`
- **WhatsApp-first contact flow** — the contact form builds a pre-filled `wa.me` deep link from the visitor's inputs instead of sending email, matching how the business actually operates
- **Scroll-reveal animations** — sections fade/slide in via `IntersectionObserver`, with a `try/catch` fallback so older browsers still render all content
- **Live clock widget** — a small CCTV-style live clock in the hero, reinforcing the "always-on monitoring" brand feel
- **Responsive, single-file architecture** — no build step; the whole site is one HTML file with inlined CSS and JS

## Architecture & Security

This repository is a **sanitized portfolio copy**, not the production codebase. Two things were deliberately changed from the live site:

1. **Real contact info kept.** The WhatsApp number, email, and CNPJ are genuine, publicly listed business details — removing them would make this an inaccurate portfolio piece, and they carry no security risk on their own.
2. **AI chat mocked.** In production, the "Sião" widget sends messages to a private n8n workflow that uses AI to generate real responses and can escalate to a human. That workflow is business infrastructure and isn't exposed here. In this copy, `SIAO_DEMO_REPLIES` is a small array of canned Portuguese responses played back with a simulated `setTimeout` delay — **no network request is made**. This was verified directly: sending a message in this demo triggers zero `fetch` calls.

Everything else — layout, copy, images, i18n — matches the production site.

## Backend Automations (n8n)

The production site is backed by a set of n8n workflows that run the business day-to-day. None of that automation is exposed in this portfolio copy (same boundary as the chat widget above), but here's what it does:

- **Sião — the WhatsApp assistant.** An AI Agent workflow that receives every inbound WhatsApp message, keeps conversation context, and answers using a knowledge base about services, pricing philosophy, and the installed client base — escalating to a human teammate when a conversation needs one. Getting it production-ready meant handling more than the happy path: WhatsApp delivery-status callbacks (read receipts, not real messages) were originally reaching the AI agent and causing errors, so a filtering step now checks for actual message content before anything reaches the model.
- **Photo-based support tickets.** Clients can open a technical support ticket — say, a malfunctioning camera — by sending photos straight into the WhatsApp chat, no separate app or portal required. The workflow collects incoming photos in temporary storage, waits for a small minimum before proceeding, and once the client confirms they're done, opens a ticket in the company's field-service platform with the photos attached automatically. Because photos from the same sender can arrive milliseconds apart, several concurrency edge cases had to be engineered around — duplicate folder creation, duplicate confirmation replies — each resolved by having every concurrent run converge on the same source of truth (the storage listing itself) instead of relying on in-memory state, which isn't safe across parallel executions. Photos that never turn into a ticket (an abandoned conversation) are automatically cleared out after 45 days, so storage doesn't grow indefinitely.
- **Automatic client & address lookup.** When a ticket is opened, the workflow tries to identify the requesting client and their registered address automatically, matching on phone number and condominium name against the field-service platform's records — skipping a manual step for the common case. Because a wrong match here has a real-world consequence (dispatching a technician to the wrong address, not just a data mistake), the matching logic is deliberately conservative: any ambiguity falls back to asking the client for the address directly rather than guessing the closest match.
- **Human handoff with full audit trail.** Sião answers first, but a teammate can step into any conversation at any moment from a shared team inbox (Chatwoot), replying on the exact same WhatsApp number and thread the client already sees — no separate app, no visible channel switch. That intervention is additive, not a lock: the assistant keeps answering the client's next message as normal regardless of whether — or how recently — a human also replied, so there's no toggle to remember to flip back. Every client-facing message — the AI conversation, photo uploads, finalized-ticket notifications, and financial documents sent over WhatsApp — is mirrored into the shared inbox as an internal note, so the team has one place to see the complete history of every client interaction, regardless of which automated workflow produced it.
- **Session-window recovery via approved templates.** WhatsApp only allows free-form business messages within 24 hours of the client's last message; outside that window, only a small set of Meta-approved message templates can re-open the conversation. The team keeps a couple of these templates on file — one to notify a client their ticket was resolved even after the window lapsed, and a generic one a human teammate can send to restart a stalled conversation — so a lapsed window never means waiting on the client to write first.
- **Financial document delivery over WhatsApp.** Clients can ask Sião for a boleto, nota fiscal, or service order without leaving the chat. Because these are financial documents, the workflow never releases one on a name match alone — it also confirms the client's CNPJ before returning anything, and returns the same generic "not found" message either way, so a close guess can't reveal whether a name was almost right.
- **Monthly document distribution.** A scheduled workflow runs at the start of each month, finds newly generated client documents in cloud storage, emails each client automatically, and files the sent documents away from the pending queue — replacing what used to be a manual, easy-to-forget step.
- **Lead notifications.** When a new lead comes in through the site, the team gets an email with a one-tap WhatsApp link to the lead's number, so follow-up starts in seconds instead of requiring someone to copy a phone number by hand.

- **A real database behind the assistant.** Sião's conversation memory moved from in-process storage to a managed Postgres database (Supabase, hosted in Brazil), so context survives server restarts. The same database now keeps an audit trail of everything the automations do — tickets opened, photos attached, documents sent, sales leads, satisfaction ratings, and automation failures — with row-level security on every table and no public API exposure. A nightly job backs every table up to cloud storage and then applies a retention policy (old conversations and error logs are deleted automatically, but only after that night's backup has succeeded), which keeps the data footprint aligned with Brazil's data-protection law (LGPD).
- **Feeding the model facts, not hoping it remembers.** Photos travel through a different pipeline than text, so the AI never "sees" them arrive — and early on it would keep asking for photos the client had already sent. Instructions in the prompt didn't fix it; what did was injecting a system-generated line with the real photo count (and today's date) into every message the model receives. After that, an interactive WhatsApp button ("Go ahead and open the ticket") and a two-minute silence timer let the assistant move on by itself once enough photos are in. Rapid-fire messages are also grouped for a few seconds and answered once, instead of producing three near-identical replies.
- **Knowing when a teammate already spoke.** Replies typed by a human in the shared inbox are written into the assistant's memory with a clear team marker, and the prompt treats them as decisions already made: no repeating a question the teammate asked, no re-requesting data or photos, no new ticket if the teammate said the existing one is already prioritized. The rules for this came from reading real conversations where the team stepped in — the questions the team asks (is a plant blocking the gate sensor? is there a power fluctuation?) became the assistant's own troubleshooting steps.
- **Ticket status on demand.** Clients can ask "how is my ticket?" and get the live status and a one-line summary of the technician's report. Because the field-service API takes tens of seconds per page, tickets and clients are mirrored into the database every 30 minutes and only the specific ticket is refreshed live. For privacy, the lookup only ever returns tickets linked to the phone number that is chatting — the assistant never offers to check another number.
- **Closing the loop with clients.** Two hours after a ticket is marked as resolved, the client receives a 1–5 satisfaction question; the rating is recorded and a low score alerts the team by email. Three days before each due date, the condominium receives an email reminder with its boleto attached. On the 3rd of every month, each property manager gets a report of the previous month: tickets opened and resolved, average time to resolution, average rating, and the list of visits. New client-facing features start in a review mode that only emails the internal team, and are switched on after a human checks the output.
- **A knowledge base the team can grow.** General questions about the company, services, and contract model are answered from a vector knowledge base (retrieval-augmented generation). The team adds or replaces topics through a simple form, and a weekly job reads the human replies in the shared inbox and suggests by email up to five new topics worth teaching the assistant.
- **Ticket intake that checks before it acts.** Before opening a ticket, the workflow checks for an open ticket for the same property in the last 30 days, asks the client to confirm when the address they give differs from the registered one, and matches property names by whole words (an early substring match confused "Residência" with "Residencial" and filed a ticket under the wrong client). When nobody on site can take photos, the ticket can be opened without them and flagged for the technician.
- **Operations you don't have to watch.** Every 15 minutes a monitor checks all workflows for failures and alerts the team; a daily report summarizes successes and failures. All workflow definitions are backed up nightly with secrets masked, monthly document folders are created ahead of time, and a password-protected dashboard shows conversations, tickets, documents sent, leads, and failures at a glance. API keys live in the automation platform's credential vault or in server environment variables — never inside workflow code.

These workflows are treated as business infrastructure, the same way the chat webhook is: purpose and design are documented here, but internals — credentials, endpoints, workflow definitions — are not published.

## Technologies

- Vanilla HTML/CSS/JS — no framework, no build tooling, no dependencies
- `IntersectionObserver` for scroll animations
- `localStorage` for language persistence
- Google Fonts (Oswald, Work Sans, JetBrains Mono)
- GitHub Pages for static hosting
- n8n for backend automation (WhatsApp AI assistant, photo-based support tickets, human handoff, document delivery, reminders, reports, monitoring)
- Postgres (Supabase) for conversation memory, audit logs, a field-service cache, and a vector knowledge base (pgvector)
- Claude (Anthropic) as the assistant's language model

## Project Structure

```
guardadesiao-portfolio/
├── index.html          # main site (all CSS/JS inlined)
├── privacidade.html     # privacy policy page (LGPD-compliant), trilingual
├── logo.png
└── photo-2.jpg … photo-8.jpg   # property/team photography
```

## What I Learned

Building a trilingual single-file site pushed me to think carefully about separating content from structure: keeping every string in one `translations` object (rather than scattering `if (lang === ...)` checks through the markup) made adding the third language dramatically faster than adding the second. It also made clear how much easier real automation integrations (like the WhatsApp deep-link and the n8n-backed chat and document workflows) are to reason about once the static presentation layer is fully decoupled from where the dynamic behavior actually lives — which made this sanitized copy possible without touching a single line of markup or business logic. On the automation side, the recurring lesson across the photo-ticket and handoff workflows was the same one in two disguises: anything that can run as more than one concurrent execution needs its coordination to live in a shared, already-consulted source of truth, not in memory local to a single run. A second lesson came later, from the photo flow: a prompt instruction cannot fix a fact the model never sees. When information arrives through another pipeline, it has to be injected deterministically into every turn — the same way the client's phone number already was.

## Project Evolution

The production site integrates the chat widget with a live n8n automation and AI backend, plus the photo-based ticketing, human handoff, document-delivery, and notification workflows described above. In October 2026 the backend gained a database layer (persistent memory, audit trail, backups with retention), ticket status lookup, satisfaction surveys, due-date reminders, monthly reports for property managers, a team-editable knowledge base, and automated failure monitoring. For this public portfolio copy, the chat integration was replaced with a self-contained mock so the code can be shared safely: same UI, same interaction pattern, zero external calls. The other automations aren't part of the static site at all, so they're described here rather than shipped as code.

## Credits

Photography and property details are used with permission from Guarda de Sião. Partner and client names shown are real, current business relationships as displayed on the production site.
