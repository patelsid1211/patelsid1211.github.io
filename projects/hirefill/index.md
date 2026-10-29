---
layout: project
title: HireFill
kind: Chrome extension
summary: Fills job applications on Workday, Greenhouse, Lever and 25 more sites with one keystroke, without an account and without your data leaving the browser.
status: Chrome extension
stack: [JavaScript, Manifest V3, chrome.storage, WCAG 2.2]
links:
  - label: See the live demo
    url: /hirefill/
  - label: Privacy policy
    url: /hirefill/privacy.html
description: How I built HireFill, a privacy-first Chrome extension that fills job applications.
---

## The problem

Applying to jobs means typing the same résumé into a different form every time. Workday is the worst of it.
It makes you search for and pick every skill one at a time. Most autofill tools fix this by asking you to
create an account and upload your résumé to their servers.

I wanted the speed without giving my data to anyone.

## What I built

- **One shortcut to fill a form.** Press Alt + Shift + F on any application page. HireFill matches each field
  by its label, placeholder and autocomplete attributes, fills text fields, dropdowns and radio buttons, and
  highlights everything it filled so you can check it.
- **Workday skills, picked for you.** HireFill types, searches and selects each skill in Workday's picker
  from your saved list.
- **Résumé and cover letter in one click.** When a page asks for a file, a small card offers your saved résumé,
  or lets you choose a tailored one.
- **Saved answers.** Answer "Why do you want to work here?" or "Willing to travel?" once, and it fills them next time.
- **Application history.** Every form you fill is logged with the date and job title, with a weekly count and CSV export.
- **It never submits for you.** You always review and click Submit yourself.

## Engineering decisions

- **No servers at all.** Everything lives in `chrome.storage.local`. The extension makes no network requests of
  its own, so there's nothing to breach and nothing to sell.
- **Narrow permissions.** Instead of asking to read every website, HireFill only requests access to the
  applicant tracking sites it supports, plus the active tab when you press the shortcut.
- **Onboarding that takes two minutes.** Rather than a long form, you paste a prompt and your résumé into any
  AI assistant and paste its answer back, which creates your whole profile.
- **Accessible color in both themes.** I audited every color pair against WCAG 2.2 and APCA contrast, fixed the
  ones that failed, softened dark-mode highlights to cut glare, and made the "filled" indicator readable for
  people with color blindness.

## What I learned

Matching form fields reliably across dozens of job sites is mostly about handling messy, inconsistent HTML.
Keeping all data local made the privacy story simple to explain, and it turned into the product's main selling point.
