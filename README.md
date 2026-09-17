# lsp-nv

The Language Server Protocol lets an editor and a language analysis tool
talk to each other, so that one tool can supply completions, errors and
navigation to every editor. It is specified by Microsoft; this package
implements version
[3.17](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/).
It brings the protocol's types to novo-lang as values a server reads and
writes.

The messages travel as JSON-RPC 2.0, which is
[jsonrpc-nv](https://novo-lang.org/packages/jsonrpc-nv)'s work, and this
package is built on it.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What the protocol is

A **language server** is a separate process. An editor starts it,
talks to it over a pipe, and shows what it says. The conversation is
JSON-RPC: the editor is the **client** and sends requests such as
`textDocument/hover`; the server answers. Either side may also send a
**notification**, which is a message with no identifier and no answer.

The conversation has a fixed beginning and end. `initialize` is the
first request and nothing may precede it. The server answers with its
**capabilities**, a list of what it can do. The client sends the
`initialized` notification, and ordinary traffic begins. At the end,
`shutdown` is a request that must be answered, and `exit` is a
notification that ends the process.

Once the client sends `textDocument/didOpen`, the copy of the document
in the editor is the truth and the file on disk is not. The client keeps
the server in step with `textDocument/didChange` notifications, each
carrying a **version** that increases on every change.

A **position** in a document has a `line` and a `character`, both
zero-based. `character` is an offset measured in code units of the
**position encoding**, which the two sides agree on during
`initialize`. The protocol's historical default is UTF-16, and it is
the encoding every client must support. Version 3.17 lets a client offer
others.

| Encoding | A `character` counts |
| --- | --- |
| `utf-16` | UTF-16 code units. The default, and always supported |
| `utf-8` | Bytes |
| `utf-32` | Unicode code points |

A **range** is two positions, and its end is exclusive. A range whose
start and end are the same is empty, and names a cursor position.

A server does not write files. When a change touches forty-one of them,
the server answers a **workspace edit** and the client performs it,
which makes it one undo step and lets the user see it first.

Every function in this package performs no input or output. Reading
files, watching directories and talking over a pipe are the work of the
server built on it.

## Install

```
novo pkg add lsp-nv
```

## Example

```novo
use jrpcmsg
use jrpcid
use lspbase
use lspdiag
use lspinit
use lspmethod

fn main() [io]
    // The handshake settles the position encoding, and every position
    // afterwards is in those units.
    let client = lspinit.client_capabilities([LspUtf8, LspUtf16])
    let encoding = lspinit.negotiate_encoding(client.position_encodings,
                                              [LspUtf16, LspUtf8])
    println("positions are ${lspbase.encoding_name(encoding)}")

    // A message that arrived. The framing and the JSON-RPC decoding
    // are jsonrpc-nv's; this package types what is inside.
    let incoming = jrpcmsg.call(jrpcid.id_int(1), "textDocument/hover", None)

    // Refuse anything that arrives in the wrong phase, with the code
    // the protocol names for it.
    if not lspinit.phase_allows(LspRunning, incoming.method)
        println("refused: ${lspmethod.wrong_phase(LspRunning).code}")
    else
        match lspmethod.request_of(incoming.method)
            LspHoverReq =>
                match lspmethod.doc_position_params(incoming)
                    Err(e)  => println("bad params: ${e.code}")
                    Ok(dp)  => println("hover at ${dp.position.line} of ${dp.uri}")
            // A method this package does not type. The server either
            // reads the params itself or answers -32601.
            LspUnknownRequest(m) => println("no handler: ${m}")
            _ => println("some other request")

    // Diagnostics replace everything the client holds for the file, so
    // an empty set is how a file is cleared.
    let cleared = lspdiag.clear("file:///a.nv", 7)
    println("${cleared.version}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: lsp-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `lspbase` | Positions, ranges, locations, text edits, markup, and the conversions between a position encoding's offsets and a byte offset. |
| `lspdoc` | Document identity, versions, and the notifications that keep a server's copy in step. |
| `lspdiag` | Diagnostics, their severities and tags, and the publication that replaces a document's whole set. |
| `lspfeat` | Completion, hover, definition, references, rename, formatting and semantic tokens. |
| `lspedit` | Workspace edits: the changes a server asks the client to make, across any number of files. |
| `lspinit` | The handshake, the capabilities, the position-encoding negotiation and the lifecycle. |
| `lspmethod` | The bridge to jsonrpc-nv: a message's method and parameters as this package's values. |

## How to choose an entry point

**`lspmethod.request_of` is where a server's dispatcher starts.** It
turns a method name into one arm per request this package types, plus an
arm carrying the name of anything else.

**`lspmethod.doc_position_params` reads the parameters six requests
share.** Completion, hover, definition, references, prepareRename and
rename all take a document and a position, and this reads both or
answers `InvalidParams`.

**`lspinit.phase_allows` is the lifecycle in one call.** Ask it before
answering anything.

**`lspbase.byte_offset` and `lspbase.character_offset` are the pair a
server needs on every request.** A server holding files as bytes has to
convert every position it receives and every position it sends.

**`lspfeat.encode_tokens` writes the semantic token array.** Give it
absolute tokens; it produces the delta encoding the wire wants and sorts
them first.

## The rules a user needs

1. **A `character` is not a character.** It is an offset in the
   negotiated encoding's code units. A server that counted bytes is one
   column out on every line holding an accented letter; one that counted
   characters is out on every emoji, which is two UTF-16 units.
2. **The position encoding is settled at `initialize` and never
   changes.** `lspinit.negotiate_encoding` takes the client's first
   choice that the server supports, and falls back to UTF-16, which
   every client must handle.
3. **A range's end is exclusive.** Two ranges that touch do not overlap,
   and a diagnostic ending at the cursor does not cover it.
4. **An empty range is legal** and names a cursor. An insertion is an
   edit over one.
5. **Once a document is open, the client's copy is the truth.** A server
   that answered from the file on disk answers against text the user
   changed four keystrokes ago.
6. **A `didChange` notification's changes apply in order**, each range
   against the document as the previous changes left it. Applying them
   in any other order corrupts the document silently.
7. **Publishing diagnostics replaces every diagnostic the client holds
   for that document.** There is no add and no remove. An empty list is
   how a file is cleared, and it is the message servers forget to send.
8. **Publish diagnostics with the version they were computed against.**
   A client discards a stale set. A server that sends no version has
   chosen to show a user squiggles under the wrong words.
9. **A completion's `textEdit` is what is inserted, not its `label`.**
   The two differ constantly. `additionalTextEdits` is the import, and
   it may not overlap the main edit.
10. **An incomplete completion list must say so.** With
    `isIncomplete` false, the client filters what it has instead of
    asking again, and the list gets shorter and never gets right.
11. **Semantic tokens are five integers each, delta-encoded.** The
    character delta is from the previous token's start on the same line,
    and from the line start otherwise. The tokens must be sorted, and an
    unsorted array produces wrong colours rather than an error.
12. **No two text edits for one document may overlap**, and every one is
    against the document as it was. A server must not adjust later
    ranges for the effect of earlier ones.
13. **A workspace edit's resource operations need a client that supports
    them.** A client that does not refuses the whole edit, so an
    unsupported file rename loses the text changes too.
14. **`shutdown` is a request and must be answered; `exit` is a
    notification.** A server that exited on `shutdown` loses the
    client's last messages.
15. **A request before `initialize` is answered -32002**
    (`ServerNotInitialized`), and one after `shutdown` is answered
    -32600. `lspinit.phase_error_code` gives the code.
16. **A capability turned on is a promise.** Start from
    `lspinit.default_server_capabilities`, which claims nothing, and
    turn on what is implemented.

## What is not included

The protocol is large and this package carries the part a language
server and a formatter need. These are named rather than left to be
discovered:

- **Code actions and code lenses** — `textDocument/codeAction`,
  `codeAction/resolve`, `textDocument/codeLens`. The types are
  substantial and a server can answer plain workspace edits without
  them.
- **Symbols and structure** — `textDocument/documentSymbol`,
  `workspace/symbol`, `textDocument/documentHighlight`,
  `textDocument/foldingRange`, `textDocument/selectionRange`.
- **Signature help, inlay hints and inline values** —
  `textDocument/signatureHelp`, `textDocument/inlayHint`,
  `textDocument/inlineValue`.
- **Type and implementation navigation** —
  `textDocument/typeDefinition`, `textDocument/implementation`,
  `textDocument/declaration`, and the call and type hierarchies.
- **The pull-diagnostics model** — `textDocument/diagnostic` and
  `workspace/diagnostic`, added in 3.17. This package carries the push
  model, `textDocument/publishDiagnostics`.
- **Semantic token deltas and ranges** —
  `textDocument/semanticTokens/full/delta` and
  `semanticTokens/range`. The full request is here.
- **On-type formatting** — `textDocument/onTypeFormatting`.
- **Workspace management** — `workspace/didChangeConfiguration`,
  `workspace/didChangeWatchedFiles`, workspace folders, and
  `workspace/executeCommand`.
- **Progress, cancellation and the window messages** —
  `$/progress`, `$/cancelRequest`, `window/showMessage`,
  `window/logMessage`.
- **Notebook documents**, added in 3.17.
- **Snippet syntax.** A completion's insert text is plain text here. The
  snippet format with its tab stops is a small language of its own.

All of them arrive as `LspUnknownRequest` or `LspUnknownNotification`
carrying the method name, so a server that implements one reads the
parameters itself and this package stays out of the way.

Also not included: **a transport**, **a document store**, and **a
parser**. A server reads its own pipe, holds its own copies of the
open documents, and analyses its own language.

## Related packages

- [jsonrpc-nv](https://novo-lang.org/packages/jsonrpc-nv) carries the
  messages and the `Content-Length` framing. This package depends on it.
- [dap-nv](https://novo-lang.org/packages/dap-nv) is the Debug Adapter
  Protocol, which is the same idea for a debugger and travels over the
  same framing.
- [jsonpatch-nv](https://novo-lang.org/packages/jsonpatch-nv) changes a
  JSON document by RFC 6902, which is a different thing from a workspace
  edit: one names locations in a document's structure, the other spans
  of its text.

## Tests

```bash
novo test tests/lspbase_tests.nv     # positions, encodings, documents, diagnostics
novo test tests/lspmethod_tests.nv   # features, workspace edits, lifecycle, routing
```

The normative source is the LSP 3.17 specification, and each assertion
names the part it comes from. The reference implementations are the
TypeScript package `vscode-languageserver-types`, for the shape of the
types, and the Python package `pygls`, for the lifecycle.

The suite asserts that an emoji is two UTF-16 code units and four bytes,
that a range's end is exclusive, that changes in one notification apply
in order against the previous state, that clearing a file's diagnostics
means publishing an empty list, that a completion's inserted text is not
its label, that the semantic token encoding round-trips, that
overlapping text edits are refused, that the encoding negotiation takes
the client's preference, and that a request in the wrong phase is
answered -32002.

The tests compile today and fail at run, each on the
`not implemented: lsp-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `lspbase` — positions, ranges, encodings | the types are declared; every body is a `todo()` |
| `lspdoc` — documents and synchronisation | the types are declared; every body is a `todo()` |
| `lspdiag` — diagnostics | the types are declared; every body is a `todo()` |
| `lspfeat` — the seven requests | the types are declared; every body is a `todo()` |
| `lspedit` — workspace edits | the types are declared; every body is a `todo()` |
| `lspinit` — the handshake and the lifecycle | the types are declared; every body is a `todo()` |
| `lspmethod` — the bridge to jsonrpc-nv | the types are declared; every body is a `todo()` |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
