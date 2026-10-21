# OSINT

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / .claude/skills/osint/SKILL.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/.claude/skills/osint/SKILL.md)

---

---
name: osint
description: Playbook for OSINT challenges — a description asking you to find a person, organization, or piece of information using public sources; a username, handle, email, domain, or image to investigate. Use after ctf-methodology has classified the challenge as osint.
---

# OSINT

Reference commands: [na--knowledge--tools-reference--osint.md](https://raw.githubusercontent.com/BigBread57/zametki/main/15-osint/na--knowledge--tools-reference--osint.md).

## Hard rule — read this first

**Passive recon only.** Do not actively scan, port-probe, brute-force login
against, or otherwise touch infrastructure belonging to a real person or
organization that shows up during an OSINT challenge. The challenge is the
puzzle; a real target that happens to surface during it is not in scope for
anything beyond looking at what's already public.

## Approach

1. **Extract every concrete entity first**, before searching for anything:
   usernames, handles, emails, domains, IPs, phone-number formats, physical
   locations implied by an image. Treat each as a separate pivot rather than
   searching for "the challenge" as a whole.
2. **Pivot per entity type**:
   - Username/handle → `~/tools/sherlock/.venv/bin/sherlock <username>` for
     automated cross-platform search, or targeted `WebSearch`/`WebFetch` for
     a quick single-platform check.
   - Email → check domain ownership (`whois`), check if it surfaces in public
     leak-search results.
   - Domain → `whois`, DNS, Wayback Machine snapshots
     (`web.archive.org/web/2024*/<url>`), certificate transparency (`crt.sh`,
     web-searchable).
   - Image → `exiftool` for embedded GPS/metadata *first* — highest-value
     single step on any provided image — then visual clues (landmarks, visible
     text, reflections) via direct multimodal reading.
3. **Correlate across pivots** rather than treating each as independent —
   OSINT challenges usually require chaining two or three findings (a handle
   found via one platform leads to a domain, which whois-resolves to a name
   that matches a challenge clue), not one single lookup.
4. **Claude Code's built-in `WebSearch`/`WebFetch` cover a large share of this
   category directly** — reach for those for a quick single check; use
   `sherlock`/`theHarvester`/`holehe` ([na--knowledge--tools-reference--osint.md](https://raw.githubusercontent.com/BigBread57/zametki/main/15-osint/na--knowledge--tools-reference--osint.md))
   when the search needs to be systematic across many platforms/sources at
   once.

## Verification discipline

An OSINT answer is confirmed when it's corroborated by at least two
independent public sources, or matches the exact format the challenge asks
for — a single plausible-looking match isn't enough given how easy it is to
land on the wrong person/entity with a common name or handle.
