# method — how the work is done, and how it goes wrong

Three documents. All are short, and the first two are worth more than any
single finding in this repository.

| Document | What |
|---|---|
| [discipline.md](discipline.md) | The rules that keep a measurement honest. Every one was learned by getting it wrong first. **Read this before measuring anything.** |
| [naming-oracle.md](naming-oracle.md) | The technique that turned function naming from a heuristic into a mechanism — the single biggest change to this project's economics. |
| [instruments.md](instruments.md) | The tools: capturing the game's own API calls, disassembling, locating a field's reader, and drawing a field when statistics stall. |

`discipline.md` opens with the rule the rest of the project is built on: a
format is known when two independently written decoders agree, and not
before. The refutations kept throughout [../formats/](../formats/) are what
that rule looks like in practice.
