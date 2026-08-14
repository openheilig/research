# engine — how the retail engine behaves

Not file formats but behaviour: what the shipped binary is built from, and
what its rules actually compute.

| Document | What |
|---|---|
| [tech-stack.md](tech-stack.md) | What the Linux port is made of, measured from the binary rather than from documentation — 615 dynamic imports, what LGP swapped in for the Windows middleware, and the class information that survived stripping. |
| [combat-formulas.md](combat-formulas.md) | One formula recovered end to end and confirmed in two binaries. Offered as the worked example of what the method yields, not as a combat model — the rest of the system is not decoded. |

`combat-formulas.md` is also where the project's most interesting negative
result lives: a clamp that exists in the code and is absent from retail
behaviour.

## Related

This describes the **retail** engine. The reimplementation that consumes these
findings is the [engine](../../engine) repo — same word, different thing.
