# ctf_workflow

[Оглавление](https://raw.githubusercontent.com/BigBread57/zametki/main/INDEX.md) · [Происхождение и ограничения](https://raw.githubusercontent.com/BigBread57/zametki/main/SOURCES.md)

> Справочный документ. Инструкции и примеры ниже принадлежат источнику; они не подтверждают состояние вашей системы и не запускаются автоматически. Чтение этого файла не устанавливает скил, инструменты или MCP.
> Архивная методика старой реализации; упомянутые функции и обёртки в библиотеку не включены.

Источники этого текста:

- [nonoleekr/CTF-Agent / archive/legacy-python-agent/prompts/ctf_workflow.md](https://github.com/nonoleekr/CTF-Agent/blob/0cbb136429a499ee586da468425d0a45132970e2/archive/legacy-python-agent/prompts/ctf_workflow.md)

---

General CTF agent workflow (category-agnostic reference).

This mirrors the actual solve loop in `workflows/main.py::run_investigation`:

1. Read the challenge description (and target file, if provided).
2. Classify the challenge into a category (keyword ladder, or Gemini if
   `CTF_AGENT_USE_LLM` is enabled).
3. Select the next unused tool for that category, in order, with
   evidence-driven refinements on top of the static order.
4. Run the tool, record its structured `Evidence` in memory, and generate
   one follow-up hypothesis from its output.
5. Check the output for a flag: regex first, then optionally a
   Gemini-assisted second pass — never the reverse, and the model's answer
   is only accepted if it's a verbatim substring of the evidence text.
6. If a flag was found, no tools remain, or the step limit is hit: save
   memory, build and export a writeup, and stop.
7. Otherwise, repeat from step 3.

Rules that apply across every category:

- Never assert a flag that isn't a verbatim match against actual evidence.
- Prefer the deterministic/heuristic path; treat any LLM assistance as an
   optional enhancement that must degrade safely if disabled or unusable.
- Every step should leave a record (evidence, hypothesis, or writeup
   content) — no silent reasoning that isn't captured anywhere.


---

## Зависимости и ссылки вне библиотеки

Следующие файлы или ресурсы упомянуты в исходнике, но не входят в эту библиотеку документов. Их наличие и работоспособность здесь не проверялись.

- `archive/legacy-python-agent/app/main.py`
- `archive/legacy-python-agent/workflows/main.py`
