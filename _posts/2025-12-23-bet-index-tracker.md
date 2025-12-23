---
title: Building a BET Index Tracker as a Fully Static App
date: 2025-12-23 10:00:00 +0200
categories: [Personal-Projects, Investing, Engineering]
---

# Building a BET Index Tracker as a Fully Static App

Over the past weeks I built a small web application that helps me track a personal BVB portfolio against the BET index and understand how to invest additional cash in a way that stays close to the index over time.

You can check out the project here:
https://github.com/DragosGeornoiu/bvb-index-distribution

And more can be read on the README.md of the github repository, as it's public: https://github.com/DragosGeornoiu/bvb-index-distribution

The live app is hosted as a GitHub Pages site and is entirely static.

This article is a mix of motivation, product thinking, and technical notes on how the app is structured and why certain decisions were made.

It's not a perfect app and it's not a perfect algorithm, but it is more than sufficient for my personal use case and, more importantly, it was an interesting system to design and reason about.

---

## Motivation

I wanted a tool that answers a few very specific questions:

- How close is my current portfolio to the BET index?
- Which positions are overweight or underweight?
- If I add new money, how can I invest it without selling anything and still track the index reasonably well?
- How does the BET index itself evolve day by day?

Existing tools tend to focus on performance tracking, trading, or very generic portfolio analytics. I wanted something simpler, deterministic, and aligned with how I actually invest: buy gradually, avoid unnecessary trades, and accept small tracking errors.

---

## A deliberate choice: fully static

One of the key constraints I imposed from the beginning was:

No backend. No database. No live APIs at runtime.

The entire application is:
- plain HTML, CSS, and vanilla JavaScript
- hosted on GitHub Pages
- driven by CSV and JSON files committed to the repository

This approach has several advantages:
- zero operational cost
- no authentication or secrets to manage
- deterministic behavior (results can be reproduced for any given day)
- extremely simple deployment (a git push is enough)

It also naturally limits complexity and forces the logic to remain explicit and inspectable.

---

## Pages overview

### Tracker

The main page (`index.html`) does most of the work.

It allows:
- importing a portfolio via CSV upload
- importing the same data via copy/paste from TradeVille (the platform I use for BVB investing; this part is intentionally tailored rather than fully generic)
- comparing portfolio weights against normalized BET weights (useful when not all BET constituents are held)
- generating buy-only investment suggestions
- validating the result via a simulated “portfolio after applying suggestions”

The emphasis is on transparency: all intermediate numbers are visible. This makes it easy to sanity-check results and adjust the algorithm when something feels off.

---

### Allocation History

The Allocation History page was added once it became clear that understanding the index itself is just as important as tracking a portfolio against it.

This page:
- loads daily BET snapshots
- displays symbol weights as a time-series chart
- allows selecting symbols and time ranges
- highlights day-to-day changes

Seeing abrupt changes (for example, a sudden increase in TLV weight) provides useful context for why a portfolio might drift even without any trading activity.

---

### Data Coverage

The Data Coverage page is intentionally boring.

It shows:
- which daily snapshots exist
- which days are missing
- direct links to each CSV file

Its main purpose is trust and debuggability. When results look unexpected, it is easy to verify exactly which data was used.

---

## Portfolio input formats

### CSV upload

The simplest input format is a CSV file:

symbol,value
SNP,126506.66
TLV,122179.18
SNG,77051.70
H2O,75628.80
BRD,45400.00
CASH_VALUE,12023.80

The idea is straightforward:
- values represent current position value, not percentages
- CASH_VALUE represents available cash and is excluded from portfolio weights

The app does not enforce currency; it only assumes consistency.

---

### TradeVille copy/paste

Because I personally use TradeVille, I added a tolerant copy/paste parser.

You can paste something like:

Instrumente BVB in RON
simbol    Nume                       sold     Pret piata   evaluare      Pondere
SNP       OMV Petrom                 128,368  0.9855       126,506.66    20.54%
TLV       Banca Transilvania         4,027    30.34        122,179.18    19.84%
RON       RON                        12,023.80

The parser:
- ignores headers and extra text
- extracts symbol and evaluated value
- detects the RON row and maps it to CASH_VALUE
- ignores totals and summary rows

Internally, this is converted to the same structure as the CSV upload.

---

## Index data and daily scraping

BET constituent weights are stored as daily CSV snapshots in the repository under:

input/bvb_distribution/

A GitHub Actions workflow runs once per day and performs the following steps:
1. Fetches the BET composition from the source page.
2. Parses the published table into a structured format.
3. Writes a date-stamped CSV snapshot.
4. Updates a separate “latest” snapshot for easy consumption by the app.

Because these snapshots are committed to the repository:
- the application never calls external services at runtime
- historical analysis is trivial
- index evolution can be visualized without additional infrastructure

This model trades freshness for simplicity and reproducibility, which is a trade-off I am comfortable with for a personal tool.

---

## Buy-only suggestion algorithm

The investment suggestion logic is intentionally constrained and pragmatic.

### Constraints
- buy-only (no selling)
- whole shares only
- total spend cannot exceed available cash
- minimum investment per symbol is enforced
- estimated broker fees are included (variable + fixed)
- prices are not rounded in the UI to avoid misleading calculations

### Approach

The algorithm is primarily greedy:
- index weights are normalized to the chosen universe
- current portfolio weights are computed
- underweight symbols are identified
- cash is allocated iteratively to reduce overall tracking error

At each step, the algorithm evaluates which feasible purchase reduces deviation from the index the most, while still respecting real-world constraints such as fees and minimum order sizes.

This is not an optimization algorithm in the strict mathematical sense. It does not guarantee a globally optimal solution, and it does not attempt to solve a full knapsack or quadratic programming problem.

Instead, it aims for:
- reasonable index tracking
- predictable behavior
- easy inspection and adjustment

In practice, the results are “good enough” and align well with how incremental investing actually works.

---

## Using AI as a force multiplier

A large part of this project was built with the help of AI.

I used it to:
- scaffold pages quickly
- iterate on UI structure
- explore alternative allocation strategies
- refactor logic while keeping behavior stable

However, it was not a hands-off process. Most iterations involved:
- reviewing generated code carefully
- adjusting constraints and edge cases
- validating output against real portfolio data
- refining UX and wording

AI significantly accelerated the process, but the intent, constraints, and final decisions remained manual.

---

## Why I enjoyed this project

This project sits at a useful intersection of:
- investing
- data engineering
- frontend logic
- pragmatic product design

It is small enough to evolve freely, yet complex enough to surface real trade-offs between correctness, usability, and simplicity.

Most importantly, it solves a real problem I actually have. That alone made it worth building and iterating on.

---

## Conclusion

Building this tool reinforced an idea I keep coming back to: small, focused, well-constrained systems can be surprisingly powerful.

A static site, some CSV files, and a bit of JavaScript were enough to build something genuinely useful, inspectable, and easy to maintain — without unnecessary complexity or infrastructure.
