# Team knowledge base — an MCP server

> A team's runbooks and decisions, readable by the assistant you already use: documents and tags
> served over MCP, called with an API key, within what the key's owner may do.

Built with [Sovrium](https://sovrium.com) — a configuration-as-code interpreter: one config
file in, a complete self-hosted web application out.

## Use this template

Click **Use this template** on GitHub to copy this app into your own repository (clean
history, yours to modify), or scaffold it locally:

```bash
curl -fsSL https://sovrium.com/install | sh
sovrium init my-mcp-server --template mcp-server
```

## What's inside

The knowledge base of a fictional engineering team: eight documents — runbooks, decisions, an
incident write-up, onboarding notes — under four tags. The page at `/` says how to switch the
server on and connect a client, and lists the tools. The admin console at `/_admin` is where
keys, roles and documents are managed.

Optional: an admin can import a runbook the team keeps in a GitHub repository, filed as a
draft, with the `import-runbook` automation; it reads the file through the `github` connection
from the Sovrium library (`sovrium library add connection/github`). Set `GITHUB_TOKEN` to turn
it on — without it the app runs exactly the same.

Everything is declared in [`app.yaml`](./app.yaml) and the [`config/`](./config) tree — no
application code. Replace the tables in `config/tables/` and the documents in `seed/`.

The team's reasons live beside the config: `config/decisions/` holds three decision records —
why an assistant may add a document but never edit one, why a client acts as the person whose
key it holds, and why the data stays in SQLite until the server is shared. The admin console
lists them at `/_admin/decisions`. When you change one of those choices, add a record that
supersedes the old one rather than editing it.

### The tools

Each table's `aiAccess` block says which tools exist and which fields they return. The names
below are the ones `tools/list` returns; each carries the app's `name` as its prefix, so they
follow when you rename the app:

| Tool                                  | Kind      | Who may call it        |
| ------------------------------------- | --------- | ---------------------- |
| `mcp-server-example_documents_list`   | read-only | every role             |
| `mcp-server-example_documents_read`   | read-only | every role             |
| `mcp-server-example_documents_create` | writes    | an editor and an admin |
| `mcp-server-example_tags_list`        | read-only | every role             |
| `mcp-server-example_tags_read`        | read-only | every role             |

The documents tools return `title`, `body`, `tag` and `status` — the author and the timestamps
stay on the server. There is no update or delete tool: adding a document is easy to review and
undo, a silent edit is not.

### The seeded accounts

Nobody can open an account from outside. The admin comes from `AUTH_ADMIN_EMAIL` and
`AUTH_ADMIN_PASSWORD`; the editor and the viewer from `seed/users.yaml`, with the password in
`SOVRIUM_SEED_PASSWORD`:

| Account                        | Role   | What their key may do                              |
| ------------------------------ | ------ | -------------------------------------------------- |
| `priya.nair@teamkb.example`    | admin  | everything, drafts included; the admin console     |
| `marco.bianchi@teamkb.example` | editor | list and read documents and tags, create documents |
| `elin.sundberg@teamkb.example` | viewer | list and read documents and tags                   |

A tool call runs as the key's owner, with their role read afresh on every call: the assistant
can never write what its owner could not.

## Connect a client

Switch the server on, seed the knowledge base, and start:

```bash
sovrium seed app.yaml        # needs SOVRIUM_SEED_PASSWORD, see .env.example
MCP_ENABLED=true MCP_TRANSPORT=streamable-http sovrium start app.yaml
```

A key is minted by a signed-in person and carries their role. Sign in as the editor and ask for
one (the value is shown once):

```bash
curl -c jar -H "content-type: application/json" \
  -d '{"email":"marco.bianchi@teamkb.example","password":"<SOVRIUM_SEED_PASSWORD>"}' \
  localhost:3000/api/auth/sign-in/email
curl -b jar -H "content-type: application/json" -d '{"name":"my assistant"}' \
  localhost:3000/api/auth/api-key/create      # → { "key": "…" }
```

An admin can also create keys in the console, at `/_admin/api-keys`. Then give your client the
server's address and the key:

```json
{
  "mcpServers": {
    "team-kb": {
      "type": "http",
      "url": "http://localhost:3000/mcp",
      "headers": { "x-api-key": "<your key>" }
    }
  }
}
```

The server speaks MCP protocol revision 2026-07-28 over streamable HTTP. A client that signs in
through the browser can use OAuth instead: `/mcp` also accepts an access token on
`Authorization: Bearer`, and the key's owner is then whoever signed in.

Then ask: "How do I rotate the database credentials?" The assistant lists the documents, reads
the runbook of that name, and answers with its four steps.

## What to try

- **Without a key**, `/mcp` answers `401` and names no tool.
- **With Elin's key** (the viewer), the client lists four tools, and a call to
  `mcp-server-example_documents_create` is refused — nothing is written.
- **With Marco's key**, ask the assistant to record a postmortem draft ("Postmortem — disk
  full"). Sign in to `/_admin` as Priya and it is there, as a draft.
- **The draft on MCP transports** is in the seed but not in what the list tool returns to Elin:
  a draft is read by its author and by an admin only.

## Run locally

```bash
curl -fsSL https://sovrium.com/install | sh
cp .env.example .env
sovrium seed app.yaml        # needs SOVRIUM_SEED_PASSWORD in .env
sovrium start app.yaml --watch
```

Zero-config: embedded SQLite, local file storage, no variable required to boot.
[`.env.example`](./.env.example) lists the optional ones this app reads.
The first administrator comes from `AUTH_ADMIN_EMAIL` and `AUTH_ADMIN_PASSWORD` on the
first start, or from `sovrium admin create <email>`.
`--watch` reloads the app each time you save a config file.

## Deploy

Ship it to Sovrium Cloud from this directory: sign in once, then deploy. The CLI finds or
creates the app it deploys to, checks the variables it needs, bundles and uploads it, and
ends only once the app answers at its address.

```bash
sovrium login
sovrium deploy
```

Any other host works the same way: run the `sovrium` binary with this config (see the
[deployment guides](https://sovrium.com/en/docs/installation)).

> **The demo accounts are for trying the app, not for running it.** The seed gives every
> account it creates, the admin included, the one password in `SOVRIUM_SEED_PASSWORD`, and
> their addresses are in `seed/`. Before anyone else can reach the app, choose a long
> password or skip the seed, and change or delete the demo admin.

## About this repository

This repository is **auto-published from the
[Sovrium monorepo](https://github.com/sovrium/sovrium)** on every release
(source: [`src/templates/mcp-server`](https://github.com/sovrium/sovrium/tree/main/src/templates/mcp-server)).
Issues are welcome here; please send code contributions upstream to the monorepo so the
template stays in sync with the engine. The pinned engine release lives in
`.sovrium-version`.

License: [MIT](./LICENSE). The Sovrium engine itself is licensed separately
([BSL 1.1](https://github.com/sovrium/sovrium/blob/main/LICENSE.md)).
