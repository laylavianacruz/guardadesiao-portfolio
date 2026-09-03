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
- **Financial document delivery over WhatsApp.** Clients can ask Sião for a boleto, nota fiscal, or service order without leaving the chat. Because these are financial documents, the workflow never releases one on a name match alone — it also confirms the client's CNPJ before returning anything, and returns the same generic "not found" message either way, so a close guess can't reveal whether a name was almost right.
- **Monthly document distribution.** A scheduled workflow runs at the start of each month, finds newly generated client documents in cloud storage, emails each client automatically, and files the sent documents away from the pending queue — replacing what used to be a manual, easy-to-forget step.
- **Lead notifications.** When a new lead comes in through the site, the team gets an email with a one-tap WhatsApp link to the lead's number, so follow-up starts in seconds instead of requiring someone to copy a phone number by hand.

These workflows are treated as business infrastructure, the same way the chat webhook is: purpose and design are documented here, but internals — credentials, endpoints, workflow definitions — are not published.

## Technologies

- Vanilla HTML/CSS/JS — no framework, no build tooling, no dependencies
- `IntersectionObserver` for scroll animations
- `localStorage` for language persistence
- Google Fonts (Oswald, Work Sans, JetBrains Mono)
- GitHub Pages for static hosting
- n8n for backend automation (WhatsApp AI assistant, document delivery, lead notifications)

## Project Structure

```
guardadesiao-portfolio/
├── index.html          # main site (all CSS/JS inlined)
├── privacidade.html     # privacy policy page (LGPD-compliant), trilingual
├── logo.png
└── photo-2.jpg … photo-8.jpg   # property/team photography
```

## What I Learned

Building a trilingual single-file site pushed me to think carefully about separating content from structure: keeping every string in one `translations` object (rather than scattering `if (lang === ...)` checks through the markup) made adding the third language dramatically faster than adding the second. It also made clear how much easier real automation integrations (like the WhatsApp deep-link and the n8n-backed chat and document workflows) are to reason about once the static presentation layer is fully decoupled from where the dynamic behavior actually lives — which made this sanitized copy possible without touching a single line of markup or business logic.

## Project Evolution

The production site integrates the chat widget with a live n8n automation and AI backend, plus the document-delivery and notification workflows described above. For this public portfolio copy, the chat integration was replaced with a self-contained mock so the code can be shared safely: same UI, same interaction pattern, zero external calls. The other automations aren't part of the static site at all, so they're described here rather than shipped as code.

## Credits

Photography and property details are used with permission from Guarda de Sião. Partner and client names shown are real, current business relationships as displayed on the production site.
