# PCAP / Network Forensics

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- [Salman0x01/ctf-ai-agent-skills / instructions/forensics-dfir/pcap-network-forensics.md](https://github.com/Salman0x01/ctf-ai-agent-skills/blob/0e91c90a7125c6591a56f51276275905f2dfb017/instructions/forensics-dfir/pcap-network-forensics.md)

---

---
name: pcap-network-forensics
category: forensics-dfir
triggers: ["pcap", "pcapng", "wireshark", "tshark", "network capture", "follow stream", "extract files pcap", "DNS exfiltration", "C2 beaconing", "TLS keylog", "SSLKEYLOGFILE", "HTTP credentials", "FTP", "zeek", "network forensics"]
tools: ["wireshark", "tshark", "capinfos", "editcap", "mergecap", "zeek", "NetworkMiner", "tcpflow", "foremost", "scapy", "dnscat2-decode", "aircrack-ng"]
priority: high
description: Dissect a packet capture - filter to the relevant flows, follow and reassemble streams, carve transferred files and credentials, decrypt TLS, and detect exfiltration and C2 beaconing.
---

# PCAP / Network Forensics

## Trigger context

Artifact is a `.pcap`/`.pcapng`/`.cap` (confirmed by magic `D4C3B2A1`/`A1B2C3D4`
for pcap, `0A0D0D0A` for pcapng). Asks: "what was exfiltrated", "recover the
transferred file", "find the password", "what is the C2 server", "decode the DNS
tunnel", "crack the Wi-Fi handshake", "read the HTTP/FTP session". Wireshark GUI
for exploration; `tshark`/`zeek` for scripted extraction and large captures.

## Objective

Reconstruct the conversation on the wire: which hosts talked, over what
protocols, what data (files, creds, commands, flags) moved, and whether the
traffic shows exfiltration or command-and-control - then extract the specific
payload the challenge wants.

## Methodology (decision tree)

```
1. OVERVIEW     capinfos (size/time/count); Wireshark Statistics >
                Protocol Hierarchy, Conversations, Endpoints.
2. SCOPE        pick the loud/odd flow (rare port, biggest talker, beacon cadence).
3. PROTOCOL BRANCH
     HTTP  -> objects/creds/user-agents      DNS  -> tunneling/exfil
     FTP   -> USER/PASS + data channel        SMB  -> file transfer, auth
     TLS   -> need keylog to decrypt          ICMP -> covert channel
     SMTP/IMAP -> emails+attachments          Kerberos/NTLM -> hash capture
     802.11 -> WPA handshake crack
4. FOLLOW       Follow TCP/UDP/HTTP Stream on the chosen flow; read both sides.
5. EXTRACT      Export Objects / --export-objects / tcpflow / foremost carve.
6. CREDS        grep streams for USER/PASS, Authorization, POST bodies; NTLM/Kerb.
7. DECRYPT      load SSLKEYLOGFILE for TLS; WPA PSK for 802.11.
8. DETECT       beaconing (regular intervals, fixed size), DNS/ICMP exfil volume.
9. ANSWER       decode/carve the payload; defang and report IOCs.
```

## Commands & payloads

```bash
PCAP=capture.pcapng

# --- 1. Overview -------------------------------------------------------------
capinfos $PCAP                                   # counts, duration, throughput
tshark -r $PCAP -q -z io,phs                     # protocol hierarchy
tshark -r $PCAP -q -z conv,tcp                   # top TCP conversations
tshark -r $PCAP -q -z endpoints,ip

# --- 2/3. Display-filter recon ----------------------------------------------
tshark -r $PCAP -Y http.request -T fields -e ip.dst -e http.host -e http.request.uri
tshark -r $PCAP -Y 'http.request.method=="POST"' -T fields -e http.host -e http.file_data
tshark -r $PCAP -Y dns -T fields -e dns.qry.name | sort | uniq -c | sort -rn | head
tshark -r $PCAP -Y ftp -T fields -e ftp.request.command -e ftp.request.arg
tshark -r $PCAP -Y 'ftp.request.command=="USER" or ftp.request.command=="PASS"' \
       -T fields -e ftp.request.arg
tshark -r $PCAP -Y 'tcp.flags.syn==1 && tcp.flags.ack==0' -T fields -e tcp.dstport | sort -u

# --- 4. Follow a stream ------------------------------------------------------
tshark -r $PCAP -q -z follow,tcp,ascii,0          # stream index 0 both directions
tshark -r $PCAP -Y 'tcp.stream==5' -T fields -e data -e text
# GUI: right-click packet > Follow > TCP/HTTP Stream; switch ASCII/Hex/Raw.

# --- 5. Extract transferred files -------------------------------------------
tshark -r $PCAP --export-objects http,./http_out    # also: smb,tftp,imf,dicom
tshark -r $PCAP --export-objects smb,./smb_out
tcpflow -r $PCAP -o ./flows                          # every stream to a file
# raw carve when protocol reassembly fails:
tshark -r $PCAP -Y 'tcp.stream==5' -T fields -e data | tr -d '\n' | xxd -r -p > blob.bin
foremost -i blob.bin -o carved

# --- 6. Credentials ----------------------------------------------------------
tshark -r $PCAP -Y http.authorization -T fields -e http.authorization   # Basic base64
tshark -r $PCAP -Y 'http.request.method=="POST"' -T fields -e http.file_data \
  | grep -iE 'user|pass|login|token'
# NTLMv2 / Kerberos: extract with NetworkMiner or the Wireshark auth fields,
# then crack: hashcat -m 5600 ntlmv2.txt rockyou.txt ; -m 13100 (kerberoast)

# --- 7. Decrypt TLS (needs the session keys) --------------------------------
# If you have the browser/app SSLKEYLOGFILE (premaster/secret log):
tshark -r $PCAP -o tls.keylog_file:keys.log -Y http2 -T fields -e http2.data.data
# GUI: Preferences > Protocols > TLS > (Pre)-Master-Secret log filename = keys.log
# RSA private key (only for non-PFS ciphers): TLS > RSA keys list.

# --- 8. Detect exfil / C2 ----------------------------------------------------
# Beaconing: constant inter-arrival + constant payload size to one dst.
tshark -r $PCAP -Y 'ip.dst==203.0.113.9' -T fields -e frame.time_delta_displayed -e frame.len
# DNS tunneling: many long/high-entropy subdomains to one authoritative server.
tshark -r $PCAP -Y dns -T fields -e dns.qry.name | awk '{print length,$0}' | sort -rn | head
# ICMP covert channel: unusual/large/consistent ICMP data payloads.
tshark -r $PCAP -Y 'icmp' -T fields -e data.text

# --- Zeek (structured logs for big captures) --------------------------------
zeek -r $PCAP
cat conn.log | zeek-cut id.orig_h id.resp_h id.resp_p proto service duration orig_bytes
cat dns.log http.log files.log        # files.log lists carved transfers + hashes

# --- 802.11 Wi-Fi handshake --------------------------------------------------
tshark -r wifi.pcap -Y eapol            # confirm 4-way handshake present
aircrack-ng -w rockyou.txt wifi.pcap    # crack the WPA/WPA2 PSK
# then decrypt: Wireshark > IEEE 802.11 > Decryption keys (wpa-pwd: pass:SSID)
```

## Branching logic

- **Protocol Hierarchy dominated by TLS** → you cannot read payloads without keys.
  Hunt for an SSLKEYLOGFILE in the challenge files/memory image; else pivot to
  metadata (SNI, JA3, cert CN, timing) for the answer.
- **Loud HTTP** → Export Objects first; then follow the POSTs for creds/flags.
- **FTP** → creds are cleartext on the control channel; the file is on the
  ephemeral data channel - follow that stream / carve it.
- **DNS with long random subdomains** → tunneling; concatenate and base32/base64
  or hex-decode the labels; try `dnscat2`/`iodine` decoders.
- **One host, regular tiny bursts** → C2 beacon; identify the dst as the C2 IOC,
  decode the beacon body (often base64/XOR/gzip).
- **SMB/SMB2 with writes** → Export Objects smb; reassemble the transferred file.
- **EAPOL present** → Wi-Fi crack branch (aircrack-ng), then decrypt and re-scope.

## Common pitfalls

- Filtering on ports instead of protocols - services run on odd ports; use the
  dissector (`Y http`), or "Decode As" in the GUI.
- Confusing capture filter (BPF, at capture time) with display filter (Wireshark
  syntax, post-hoc) - `tshark -f` vs `-Y`.
- Reading only one side of a stream; follow shows both, colour-coded.
- Assuming TLS is crackable without keys - PFS ciphers defeat an RSA key alone.
- Missing UTF-16/gzip/chunked encoding - decode HTTP `Content-Encoding` before
  grepping payloads.
- Ignoring `files.log` from Zeek, which already hashes every carved transfer.
- Defang IOCs in the report (`hxxp`, `1.2.3[.]4`) to avoid accidental clicks.

## Proof / verification

- For an extracted file: hash it, confirm magic, and re-run [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md).
- For creds: show the exact packet/stream and decode (base64 → plaintext) inline.
- For C2/exfil: cite the dst IP/domain, the cadence or byte volume, and the
  decoded beacon content.
- Note the stream index and display filter used so any claim is reproducible.

## Cross-references

- [sa--instructions--forensics-dfir--forensics-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--forensics-triage.md) - carve/identify files pulled out of the capture.
- [sa--instructions--forensics-dfir--soc-alert-triage.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--soc-alert-triage.md) - reputation-check extracted IPs/domains/hashes.
- [sa--instructions--forensics-dfir--threat-hunting-detection.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--threat-hunting-detection.md) - turn a beacon pattern into a detection.
- [sa--instructions--forensics-dfir--log-analysis-sherlock.md](https://raw.githubusercontent.com/BigBread57/zametki/main/09-forensics-soc/sa--instructions--forensics-dfir--log-analysis-sherlock.md) - correlate network flows with host logs.

## Credible sources

- Wireshark User's Guide: https://www.wireshark.org/docs/wsug_html_chunked/
- Wireshark display filter reference: https://www.wireshark.org/docs/dfref/
- tshark manual: https://www.wireshark.org/docs/man-pages/tshark.html
- Unit 42 "Exporting Objects from a Pcap" tutorial: https://unit42.paloaltonetworks.com/using-wireshark-exporting-objects-from-a-pcap/
- Zeek documentation: https://docs.zeek.org/en/master/
- Wireshark TLS decryption wiki: https://wiki.wireshark.org/TLS
- HackTricks pcap inspection: https://book.hacktricks.wiki/en/generic-methodologies-and-resources/pentesting-network/pcap-inspection/index.html
- NetworkMiner: https://www.netresec.com/?page=NetworkMiner
