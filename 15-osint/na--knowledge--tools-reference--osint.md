# OSINT — tools reference

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [nonoleekr/CTF-Agent / knowledge/tools-reference/osint.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/knowledge/tools-reference/osint.md)

---

# OSINT — tools reference

## Available tools
```
~/tools/theHarvester/.venv/bin/theHarvester -d <domain> -b all      # emails, subdomains, hosts from public sources
~/tools/sherlock/.venv/bin/sherlock <username>                        # cross-platform username search
.venv/bin/holehe <email>                                                # check which sites an email is registered on
whois <domain>
curl -s "http://archive.org/wayback/available?url=<url>"                    # wayback machine API, no key needed
curl -s "https://web.archive.org/web/2024*/<url>"
```
theHarvester and sherlock run from their own isolated venvs — see
`knowledge/environment.md`.

## Threat-intel API scripts — `tools/`

Need a key in `.env` (see `.env.example`); each prints exactly what's missing
and where to get it if unset, no silent failures.

```
.venv/bin/python tools/shodan_lookup.py host <ip>              # what's exposed on a host, per Shodan's existing scan data
.venv/bin/python tools/shodan_lookup.py search "<query>"          # e.g. a banner string or service fingerprint from the challenge
.venv/bin/python tools/hibp_lookup.py <email>                        # which breaches an email appeared in — passive, no probing
```
`WebSearch`/`WebFetch` (Claude Code's built-in tools) cover a large share of
OSINT work directly — reverse-image-style lookups, checking if a username/handle
shows up on a given platform, pulling a cached page. Reach for those before
assuming a dedicated tool is required.

## Entity extraction discipline
Before anything else, pull out every concrete entity from the provided
text/image: usernames, handles, email addresses, domains, IPs, phone number
formats, physical locations implied by an image. Treat each as a separate
pivot rather than searching for the challenge "as a whole."

## Pivot checklist
- **Username/handle** → `sherlock <username>` for automated cross-platform
  search, or targeted `WebSearch` per platform for a quick single check.
- **Email** → check for it appearing in leaked-data search results, and whether
  the domain has an obvious ownership trail (`whois`).
- **Domain** → `whois`, DNS records, Wayback Machine snapshots, certificate
  transparency logs (`crt.sh` is web-searchable, no special tool needed).
- **Image** → `exiftool` for GPS/embedded metadata first (this is a forensics
  tool but the highest-value OSINT step on any provided image), then visual
  landmark/text clues via Claude's own multimodal reading.

## Hard rule
Passive recon only. Do not actively scan, port-probe, or otherwise touch
infrastructure belonging to a real person or organization that shows up during
an OSINT challenge — the challenge target is the puzzle, not an invitation to
test someone's actual systems.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `tools/hibp_lookup.py`
- `tools/shodan_lookup.py`
