# Shift Report Generator

Web app for filling out Lake Merwin Campers Hideaway shift reports and
downloading them as a PDF. The PDF is flattened: answers are printed on the
page, not left as editable form fields.

## Run

```sh
python3 -m http.server 8000
```

Open http://localhost:8000 (use `localhost`, not a LAN IP — unlocking needs
HTTPS or localhost). It must be served, not opened as a file. Needs
`assets/locked/bundle.bin`; see [Access code](#access-code).

## Access code

The app opens with an access-code screen. The form template is **encrypted**
with that code (AES-256-GCM, key derived with PBKDF2-SHA256, 600k iterations)
into `assets/locked/bundle.bin`, and only that bundle is committed — the
readable files under `assets/forms/` are git-ignored and stay on your machine.
The code itself is not stored anywhere; a wrong code just fails to decrypt.

**Set or change the code** (also run after changing the form):

```sh
python3 tools/lock_assets.py      # prompts for the code twice
git add assets/locked/bundle.bin && git commit -m "Update bundle" && git push
```

Changing the code makes every device ask for the new one the next time it
picks up the update. "Remember on this device" keeps the derived key (not the
code) in a cookie for 400 days; the **Lock** button forgets it.

Use a long code (a few random words). Anyone can download the bundle and try
codes offline, so a short PIN can be guessed; a passphrase can't.

## Install on iPad / iPhone / Android (offline use)

The app works offline once installed. It has to be hosted on HTTPS first
(any static host, e.g. GitHub Pages, Netlify, Railway); devices then
install it once while connected.

- **iPad / iPhone:** open the site in Safari → Share → **Add to Home Screen**.
- **Android:** open the site in Chrome → menu → **Install app** (or Add to
  Home screen).

Open it once from the home screen while still connected. When the bottom of
the page says "✓ Available offline", it will work with no signal. On every
device, **Download PDF** saves the report as a file.

**Updating:** deploy the new files. Each device picks up the change the next
time it opens the app with a connection (the change shows on the following
launch). To force every device to re-download everything, bump `VERSION` in
`sw.js`. If you add or rename files the app loads, add them to `SHELL` in
`sw.js`.

This app keeps its own storage, offline cache and remembered-key cookie, so it
can be hosted on the same site as the citation generator without the two
interfering.

## Layout

```
index.html                  app entry point
manifest.webmanifest        install metadata (name, icons)
sw.js                       service worker: offline cache
src/
  app.js                    form handling, signatures, PDF updates
  fill.js                   fills the PDF template (pdf-lib)
  vault.js                  access code: decrypts the bundle, remembers the key
  style.css
assets/
  locked/bundle.bin                 encrypted form template (committed)
  forms/shift_report_fillable.pdf   fillable form template  (local only)
  images/logo-circle.png            logo used in the app
  icons/                        home-screen icons (generated)
vendor/pdf-lib.min.js       pdf-lib 1.17.1
tools/
  make_icons.py             regenerates assets/icons/ from the logo
  lock_assets.py            encrypts the private files into assets/locked/
```

## Form template

The template's form fields can't be filled by name: all fourteen round and
bathhouse boxes are one field ("Time" / "Time 2"), both fuel gauges are one
dropdown, both totals are one field, and the Clean / Maintenance Issue radio
groups span two patrol areas. So `src/fill.js` maps each answer (the `name`
attributes in `index.html`) to one widget — field name plus widget index — and
prints it at that widget's position, then removes the form. If the template's
layout changes, update the `TEXT`, `CHECKS`, `YES_NO` and `WRAPPED` tables
there, and run `python3 tools/lock_assets.py` so the app gets the change.

## Notes

- The employee name is remembered per browser (localStorage) once a PDF has
  been downloaded. The report in progress is kept as a draft, so
  it survives closing the app mid-shift; drawn signatures are never saved on
  the device (not even in the draft).
- Every time box has a **Now** button that stamps the current time.
- The security journal wraps across the PDF's 18 ruled lines; longer entries
  print in smaller text to fit. Area comments wrap across their two lines.
- The sun/moon button switches light and dark mode. It follows the device
  setting until used, then remembers the choice on that device.
