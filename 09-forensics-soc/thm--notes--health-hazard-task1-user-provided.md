# Health Hazard — Task 1: текст, предоставленный пользователем

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.

Источники этого текста:

- Личные конспекты TryHackMe / notes/health-hazard-task1-user-provided.md
- Личные конспекты TryHackMe / skills/thm-threat-hunting-splunk/references/task1-user-provided.md

---

# Health Hazard — Task 1: текст, предоставленный пользователем

Источник фрагмента: сообщение пользователя в этом чате, 20.10.2026.
Это именно предоставленный фрагмент, а не полный текст всех Task комнаты.

In the SOC Simulator, the work comes to you. An alert fires, it lands in your queue, and you decide what it is.

Threat hunting flips that around. Nothing has fired. Nobody is telling you something is wrong. Instead, you start with a hunch, something like "an attacker may have gotten in through a poisoned software package," and you go looking through the logs to find out if it's true. That hunch is called a **hypothesis**, and it's where every hunt begins.

That's what the **Threat Hunting Simulator** is. It gives you a simulated company, a hypothesis, a threat intelligence report, and a Security Information and Event Management (SIEM) tool full of logs. Your job is to dig through those logs, piece together what the attacker did one step at a time, and decide whether the hypothesis holds up.

In this room, we'll play the **Health Hazard** scenario together, from start to finish. It's a short hunt with three stages, all on one machine, so it's an easy one to learn the platform on. We'll go through each stage one at a time: where to start looking, what we find, and how to record it.

## Learning Objectives

- Learn how a Threat Hunting Simulator scenario works, from the hypothesis to the final report.
- Find your way around the threat intel, the SIEM, the documentation, and the timeline.
- Search Splunk to follow an attack from one event to the next.
- Map what you find to MITRE ATT&CK tactics and techniques.
- Build an attack chain and decide whether the hypothesis is proven.
- Understand the results screen and the feedback on your hunt.
