# Hotel EOP Form

> A paper HR form, reborn as a web app.

**Live demo:** [neo-penang-eop-form.pages.dev](https://neo-penang-eop-form.pages.dev/)

---

## The problem

I work in IT at a hotel. Every time staff needed to step off-property during work hours (lunch run, bank errand, quick Komtar trip), they had to:

1. Download a `.doc` file from the shared drive
2. Open it in Word
3. Fill in the same six fields every time
4. Print it on A4 paper
5. Sign it with a pen
6. Walk it over to their Department Head, then HR, then Security

The form itself is a standard two-up tear-off: one half goes to Security, the other to HR. In theory, elegant. In practice: slow, repetitive, and wasteful — if someone changed their mind about going out, an already-printed, half-signed form went straight to the bin.

I rebuilt it as a browser app.

## The solution

A single HTML file. No build step, no framework, no backend. Open in a browser, fill it in, save a PDF or print it. The output is **pixel-identical** to the original Word document, so Security and HR never had to adjust to a new-looking form.

### What it does

- **Fill in the fields once** — they mirror into both tear-off copies (optionally, per field)
- **Draw your signature** with a mouse, trackpad, or finger — it stamps both halves
- **Save your profile** (name, position, signature) to `localStorage` so you don't re-enter it every time
- **Pick a language** — English, Bahasa Malaysia, 中文, தமிழ் (the hotel's four staff languages)
- **Save as PDF** with a sensible filename like `EOP_KESH_20260417.pdf`
- **Print** to A4, guaranteed single-page output
- **Paper-saving default** — the HR (bottom) copy stays blank unless you explicitly link fields over. If you don't actually go out, you haven't wasted ink on a now-defunct bottom half

### What it doesn't do

- No backend, no database, no accounts — everything is client-side
- No tracking, no analytics, no telemetry
- No build step — the entire app is one HTML file you can open with a double-click

---

## Screenshots

> <img width="2560" height="1387" alt="image" src="https://github.com/user-attachments/assets/a51dec04-fe58-48e1-b38c-be1c76ec2223" />


<!-- 
![Control panel](docs/control-panel.png)
![Form output](docs/form-output.png)
![Side by side](docs/side-by-side.png)
![Mobile](docs/mobile.png) 
-->

---

## The engineering bits (the interesting stuff)

This was meant to be a quick weekend job. It turned into a lesson in how much bigger "just rebuild a form" actually is.

### Matching the Word doc exactly

The paper form uses **Arial Narrow** at 11pt, a **double thick-thin maroon border** (`#622423`) under the title, literal typed underscores (`_______________`) as signature lines, and a **23/77 column split** on the info table. None of those are CSS defaults. I extracted the `.doc` file, unpacked its XML, and read off the exact dimensions, colors, and font sizes to replicate them in CSS.

The signature line was the trickiest — my first attempt used `border-bottom: 1px solid black`, which spanned the full cell. The original doc uses a row of underscore characters in the middle of a vertically-padded cell, so the line is short and floating, not wall-to-wall. Switching to literal `<span class="sig-rule">_______________</span>` inside each signature cell was the fix.

### Print CSS is a minefield

Getting reliable single-page print output across Chrome, Firefox, Safari, and mobile browsers took more iterations than I'd like to admit. Key things I learned:

- **Don't trust `@page { margin: 0 }`.** Some browsers ignore it and apply their own default margins on top. Set explicit margins.
- **`min-height: 297mm` is a trap.** It forces the page element to be exactly A4, so any layout that's even 1mm taller spills to a second page. Use `height: auto` instead.
- **`page-break-inside: avoid` has side effects.** Applying it to a container pushes remaining content to the next page rather than just keeping the container together. I had to apply it narrowly to specific elements (`.form-copy`, signature tables) instead.
- **Firefox dark mode will ruin your day.** If the user's OS is in dark mode, Firefox inverts page backgrounds on print unless you explicitly declare `color-scheme: light only` and force `background: white !important` + `color: black !important` on everything inside `@media print`. Chrome doesn't do this. This took a while to figure out.

### i18n with zero dependencies

No `i18next`, no framework. Just an object:

```js
const I18N = {
  en: { name: "Name", ... },
  ms: { name: "Nama", ... },
  zh: { name: "姓名", ... },
  ta: { name: "பெயர்", ... }
};
```

And a single `applyI18n(lang)` function that walks the DOM for `[data-i18n]`, `[data-i18n-ph]` (placeholder), and `[data-i18n-title]` (tooltip) attributes and swaps their text. Adding a new language is an object literal and a `<option>` tag. Zero ceremony.

### PDF generation in the browser

I used [html2pdf.js](https://github.com/eKoopmans/html2pdf.js) (which wraps `html2canvas` + `jsPDF`). The gotcha: by default it renders at whatever viewport width the user has, so a desktop user and a mobile user get different-sized PDFs. Fix: temporarily apply a `.pdf-export` CSS class to `<body>` during generation that forces `.page` to a fixed A4 width in pixels (794px = 210mm @ 96dpi), then `windowWidth: 794` in the html2canvas options. Same crisp output regardless of viewport.

### Mobile responsive without media-query hell

Three breakpoints:
- **≤900px (tablet)** — control panel tightens, logo shrinks
- **≤600px (phone)** — inputs stack into one column, buttons go full-width, signature modal becomes full-screen with `aspect-ratio: 3/1`
- **≤380px (tiny phone)** — further size reductions

One sneaky detail: input `font-size: 16px` on phones prevents iOS Safari from auto-zooming into the input on focus. Tiny fix, huge UX win.

### Self-contained by design

Everything is in `index.html`. The logo is base64-embedded. The only external dependency is `html2pdf.js` loaded from a CDN (and the app degrades gracefully if it fails to load — the Print button still works as a fallback).

No `node_modules`, no `package.json`, no build output, no source maps, no config files. You can read the entire app in a single browser tab.

---

## Tech stack

| | |
|---|---|
| Markup | HTML5 |
| Styling | Plain CSS (no preprocessor) |
| Scripting | Vanilla JS, IIFE-wrapped, no framework |
| PDF | [html2pdf.js](https://github.com/eKoopmans/html2pdf.js) via CDN |
| Storage | Browser `localStorage` |
| Hosting | Cloudflare Pages |
| Build tool | Nope |
| Bundler | Nope |

---

## Running it locally

```bash
git clone https://github.com/<your-username>/hotel-eop-form.git
cd hotel-eop-form
open index.html   # or double-click it in Finder / Explorer
```

That's it. If you want a local server instead:

```bash
python3 -m http.server 8000
```

Then visit `http://localhost:8000`.

---

## Project structure

```
.
├── index.html      # the entire app — HTML + CSS + JS + embedded logo
├── _headers        # Cloudflare Pages security headers config
├── .gitignore
└── README.md
```

---

## Browser support

Tested on:
- Chrome / Edge (latest)
- Safari 15+ (macOS and iOS)
- Firefox (latest, including dark mode)
- Samsung Internet

Requires a browser with `localStorage`, Canvas API, and ES6+. So basically anything from ~2017 onward.

---

## Things I'd do differently

- **PWA + offline support** — register a service worker so the form works without internet. Useful when the hotel Wi-Fi is down (happens more than you'd think).
- **Server-side submissions** — right now the form generates a PDF, then someone has to email or print it. A Cloudflare Worker endpoint could accept submissions and auto-route them to HR's inbox.
- **Audit log** — who filled out what, when, and how often. Would let HR notice patterns (someone always takes a 2-hour lunch?).
- **Manager approvals** — digital sign-off rather than physical pen-and-paper, with email notifications.

None of these are necessary for the current use case, but they're where this would go if it graduated from "digital form" to "actual HR system."

---

## About me

Built by **Kesh** — IT guy at a hotel in Penang, Malaysia, accidentally doing frontend.

Find me: [GitHub](https://github.com/) · [LinkedIn](https://linkedin.com/)

---

## License

MIT — do whatever you want, just don't blame me if something breaks.

---

<sub>🥚 *Dev easter egg: open DevTools on the live site. Full menu prints automatically.*</sub>
