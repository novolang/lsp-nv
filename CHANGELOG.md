# Changelog

All notable changes to lsp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `lspbase` — the load-bearing interface, and the decision is that the
  POSITION ENCODING IS A VALUE rather than an assumption. A `character`
  offset is in code units of the encoding the handshake settled, and a
  server that counted bytes is one column out on every line with an
  accent in it while a server that counted characters is out on every
  emoji. `byte_offset` and `character_offset` are the pair a server
  calls on every request, and the encoding is one of their arguments so
  it cannot be forgotten.
- `lspinit` — the handshake as the two things it settles.
  `negotiate_encoding` takes the client's first choice the server
  supports, in the CLIENT's order, because the offer is a preference
  list; with no overlap it answers UTF-16, the one encoding every client
  must handle. `phase_allows` is the lifecycle as one function, so an
  out-of-order request is refused with -32002 instead of answered
  without a workspace root.
- `lspmethod` — the bridge, and the reason this package does not
  reimplement JSON-RPC: a request IS a `jrpcmsg.JrpcRequest`, and what
  is added is the type of its `params`. The id rules matter here because
  LSP lets a server call the client, so both ends hold a table of
  outstanding ids and the matching rule has to be the same rule.
  `shape_agrees` is the check between a well-behaved client and a server
  that hangs: the protocol says whether a method is a notification, and
  `jrpcmsg` says whether an id arrived.
- `lspdoc` — the client's copy is the truth from `didOpen`, and a
  `didChange`'s changes apply IN ORDER, each range against what the
  previous ones left. `apply_changes` is that loop written once, because
  computing every offset against the original corrupts the document
  silently.
- `lspdiag` — `publish` replaces every diagnostic the client holds for
  the document, so `clear` is named rather than left to each server to
  spell as an empty list: the message a server forgets to send is
  exactly that one.
- `lspfeat` — a completion's `text_edit` is what is inserted and its
  `label` is what is shown, and they differ constantly.
  `encode_tokens` takes absolute tokens and produces the five-integer
  delta array, sorting first, because every server that hand-rolled the
  encoding got the second delta's base wrong and an unsorted array
  produces wrong colours rather than an error.
- `lspedit` — a server changes nothing itself; it answers a workspace
  edit and the client performs it. `is_disjoint` is the
  no-overlapping-edits rule as a function worth asserting in a server's
  own tests, because clients differ on what they do with an overlap and
  a server only finds out from a user.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  lsp-nv.<module>.<fn>`.
- **jsonrpc-nv is a PATH dependency in this staged release.** Nothing is
  on the registry yet, so a sibling directory is the only thing that
  resolves. The publish round rewrites it to `^0.0.1` before the tarball
  is built: a path dependency in a tarball sends the consumer to a
  directory that does not exist.
- **The protocol's surface is scoped, and what is left out is named**
  in the README's "What is not included" — code actions, symbols,
  signature help, inlay hints, the pull-diagnostics model, semantic
  token deltas, workspace management, progress and cancellation, and
  notebook documents among them. Each arrives as `LspUnknownRequest`
  carrying its method name, so a server that implements one reads the
  parameters itself.
- **`LspCompletionKind` carries eight of the specification's
  twenty-five.** The eight a language server for a compiled language
  produces. `completion_kind_of` answers `None` for the other
  seventeen rather than guessing.
