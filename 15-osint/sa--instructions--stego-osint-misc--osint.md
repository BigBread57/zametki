# OSINT Investigation Methodology

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/stego-osint-misc/osint.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/stego-osint-misc/osint.md)

---

---
name: osint
category: stego-osint-misc
triggers: ["osint", "find the person", "find the place", "find the location", "geolocation", "who is", "username", "email", "social media", "reverse image search", "exif gps", "google dorking", "wayback machine", "identify the building", "where was this photo taken"]
tools: ["sherlock", "holehe", "maigret", "exiftool", "google-images", "yandex", "tineye", "google-maps", "google-earth", "wayback-machine", "whois", "suncalc"]
priority: high
description: Decision-tree methodology for OSINT CTF challenges - finding people, usernames, emails, and geolocating photos from open sources.
---
# OSINT Investigation Methodology

## Trigger context
Use when a challenge asks you to identify a person, account, organization, or location using only publicly available information. Signals: "find the person behind this handle", "where was this photo taken", "what is the flag hidden in their social media", a bare username/email, or a single photograph with no obvious embedded data.

## Objective
Attribute an identity or pinpoint a location from open sources, then locate the flag (often planted in a bio, pinned post, image caption, EXIF field, commit message, or Wayback snapshot). Always work only with public data; do not attempt to log in to or breach private accounts.

## Methodology (decision tree)
1. Classify the seed you were given: username, email, real name, image, or URL/domain. Each opens a different branch.
2. Username seed -> enumerate accounts across platforms:
   - Automate with `sherlock <username>` (300+ sites) and `maigret <username>` (richer, pulls profile data).
   - Manually verify the strong hits (GitHub, X/Twitter, Reddit, Instagram, LinkedIn). Read bios, pinned posts, followers, and linked sites. Check GitHub commit emails and repo contents.
3. Email seed -> `holehe <email>` shows which services the address is registered on (no password reset sent). Pivot: search the email in Google, paste sites, and gravatar (md5 of email -> avatar).
4. Real-name seed -> Google with quotes and modifiers, cross-reference LinkedIn/company pages, and pivot back to usernames.
5. Image seed (find the place / find the person) -> two parallel tracks:
   - Metadata: `exiftool image.jpg` for GPS coordinates, camera, timestamp, software. GPS often ends the challenge instantly. Also `strings image.jpg | grep -i flag`.
   - Visual: reverse image search across MULTIPLE engines (Google Lens, Yandex - best for faces/places, TinEye, Bing). Then geolocate from visual clues: signage/language, license plates, road markings, architecture, vegetation/climate, sun angle & shadow direction (SunCalc), distinctive landmarks in Google Street View / Google Earth.
6. URL/domain seed -> `whois`, DNS, certificate transparency (crt.sh), and the Wayback Machine for deleted/older content. Check robots.txt, sitemaps, and page source comments.
7. Pivot relentlessly: every artifact (avatar, reused handle, email, phone, PGP key) links accounts together. The flag is usually one or two pivots deep.

## Seed-to-branch quick map
- Username / handle -> `sherlock`, `maigret`, manual profile review, gravatar.
- Email address -> `holehe`, gravatar (md5), Google + paste-site search, breach context.
- Real name -> quoted Google dorks, LinkedIn/company pages, pivot to handles.
- Photograph -> `exiftool` (GPS) FIRST, then reverse image (Yandex/Lens/TinEye) + Street View.
- URL / domain -> `whois`, crt.sh, robots.txt/sitemap, Wayback Machine snapshots.
- Phone number -> country/carrier lookup, link to accounts registered with it.

## Commands & payloads
```bash
# --- Username enumeration ---
sherlock targetuser                            # scan 300+ sites for the handle
maigret targetuser --html                      # deeper: extracts bio/name/links, HTML report
# manual pivots: github.com/<user>, twitter.com/<user>, reddit.com/user/<user>

# --- Email enumeration ---
holehe target@example.com                      # which sites the email is registered on
# gravatar: avatar at https://gravatar.com/avatar/<md5(lowercased email)>
python3 -c "import hashlib;print(hashlib.md5(b'target@example.com').hexdigest())"

# --- Image: metadata first ---
exiftool photo.jpg                             # GPS, timestamp, camera, software, comments
exiftool -gpslatitude -gpslongitude photo.jpg
strings photo.jpg | grep -iE 'flag|http|geo'

# --- Reverse image search (use several engines) ---
# Google Lens:  https://lens.google.com/
# Yandex (best for people/places):  https://yandex.com/images/
# TinEye (finds oldest copy):  https://tineye.com/
# Bing Visual Search:  https://www.bing.com/visualsearch

# --- Geolocation confirmation ---
# Google Maps / Street View:  https://www.google.com/maps
# Google Earth (historical imagery):  https://earth.google.com/
# SunCalc (verify time-of-day from shadow direction):  https://www.suncalc.org/

# --- Google dorking ---
# "exact phrase"  site:example.com  intitle:  inurl:  filetype:pdf  "@example.com"
# reused username across sites, cached bios, leaked docs

# --- Historical / deleted content ---
# Wayback Machine:  https://web.archive.org/web/*/https://target-site.com/*
whois example.com                              # registrant, dates, nameservers
# crt.sh certificate transparency:  https://crt.sh/?q=example.com
```

## Geolocation clue checklist (photo challenges)
- Language / script on signs, menus, ads -> narrows to a region/country.
- Road markings & line colors, side of the road, license-plate format -> country.
- Utility poles, guardrails, bollards, road signs -> country-specific styles (see GeoGuessr metas).
- Architecture, roofing, building materials -> region and climate.
- Vegetation, terrain, sky/sun position -> hemisphere and latitude band.
- Shadow direction + EXIF timestamp -> verify candidate site with SunCalc.
- Any readable business name / phone -> Google it -> exact address -> Street View confirm.

## Branching logic
- Seed is a handle -> sherlock/maigret -> read profiles -> pivot to email/name.
- Seed is an email -> holehe + gravatar + Google paste-site search.
- Seed is a photo with EXIF GPS -> done fast; convert coordinates to a place, grab flag from context.
- Photo with stripped EXIF (common - platforms strip it) -> pure visual geolocation via reverse image + Street View.
- Can identify the country but not the exact spot -> narrow by language, road-line color, utility poles, then Street View sweep.
- Account found but no flag visible -> check pinned posts, older tweets (Wayback), image alt-text, repo commit history, README, and linked personal site.
- Dead/deleted profile -> Wayback Machine snapshot of the profile URL.

## Common pitfalls
- Uploading the photo to a reverse-image engine before extracting EXIF locally - you may lose the easiest win; run `exiftool` FIRST.
- Relying only on Google reverse image; Yandex is far stronger for faces and non-US places. Always try 2-3 engines.
- Assuming platforms keep EXIF; most social sites strip it, so absence of GPS is expected, not a dead end.
- Confusing similar usernames across sites (false positives from sherlock); verify each hit by profile content, not just existence.
- Crossing the line into private data / credential attacks - OSINT means OPEN sources only.
- Ignoring timestamps: EXIF DateTime + shadow direction (SunCalc) can confirm or refute a candidate location.
- Overlooking the obvious - the flag is sometimes literally in the bio or the first comment.

## Proof / verification
- State the exact pivot chain (seed -> tool/query -> artifact -> flag location) so the find is reproducible.
- For geolocation, cite the coordinates and a Street View / Maps confirmation matching the visual clues (same building, signage, road geometry).
- For identity, corroborate with at least two independent public artifacts (e.g. same avatar + same handle) before asserting attribution.

## Cross-references
- Photo also has hidden embedded data (not just EXIF) -> see steganography.md.
- Coordinates lead to a map puzzle or QR -> misc-jail-escapes.md recon section / crypto methodology.
- Domain/host enumeration deepens into web recon -> web/recon methodology.

## Credible sources
- CTF Support - OSINT: https://ctf.support/osint/
- Sherlock (username enumeration): https://github.com/sherlock-project/sherlock
- holehe (email enumeration): https://github.com/megadose/holehe
- Maigret: https://github.com/soxoj/maigret
- OSINT Framework (tool directory): https://osintframework.com/
- Bellingcat geolocation guide: https://www.bellingcat.com/resources/how-tos/2020/12/03/we-are-bellingcat-and-heres-how-you-can-verify-a-video/
- SunCalc (shadow/time verification): https://www.suncalc.org/
- Wayback Machine: https://web.archive.org/
