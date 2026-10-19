# #838 before/after: a valid group-placement repair

Same workflow content and geometry in both files; only the `command` group's
`fromCol`/`toCol` differ.

| file | group span | result |
| --- | --- | --- |
| `before.json` | `1..3` | `render` exits 1 with `workflow/route-preset-conflict` |
| `after.json` | `0..1` | `render` exits 0 |

The `after` span is not hand-picked: it is exactly what the diagnostic prints,
and applying it is what the regression tests do.

```sh
# rejected, with the repair in the message
node archify/bin/archify.mjs render workflow before.json out.html
#   [workflow/route-preset-conflict] … Fix: set group "command" fromCol to 0
#   and toCol to 1 so its label clears the "group label" blocking edge …

# accepted
node archify/bin/archify.mjs render workflow after.json out.html
```

`after.png` is a 1440x900 capture of `after.html` (dark, Classic, Still,
default zoom). The `Incident command` frame still contains `Page On-call`, so
the repair clears the route without dropping the node it was framing.

Why it matters: the same two documents also demonstrate the boundary. A span
that clears the label but changes which nodes the group contains — or which
group a node resolves to — is withheld rather than advertised.
