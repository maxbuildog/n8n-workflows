# Nightly Notion archive: move finished tasks from a TODO database to a DONE database

An n8n workflow that, every night, copies finished tasks (with their full page content) from a Notion TODO database to a DONE database, verifies the copy block by block, and only then moves the original to the trash.

## The problem

A task board fills up with finished items, but Notion's native automations cannot move a page between two databases with its content. Doing it with a naive "create then delete" is dangerous: a failed copy silently loses notes, and a retry creates duplicates. This workflow archives nightly and refuses to delete anything it could not verify.

## How it works

1. A schedule trigger fires every night at 1am.
2. A single code node (`Archive engine`) acts as a state machine. It returns one Notion request at a time; an HTTP node executes it and the response is fed back into the engine, in a loop with a short pause for Notion's rate limit.
3. The engine reads the DONE database schema, then queries the TODO database for finished tasks (stage equals the done option, or the checkbox is ticked).
4. For each task it reads the whole block tree, checks that every block type can be copied, and that the select values exist in DONE.
5. It looks for an existing archive of the same task in DONE (by the source URL). Zero hits: create the page and copy the blocks. One hit: reuse it. More than one: keep the task and report.
6. It re-reads the archive, compares properties and a block-by-block signature with the source.
7. It re-checks the source (still finished, not edited since the query), then moves it to the trash (`archived: true`, reversible for 30 days).
8. A summary node counts archived, already present, kept and errors. If there is any anomaly (and the run is not a test), a Telegram message lists the tasks that were kept in TODO and why.

## Design choices

- Delete last, after verification. The source is trashed only after the archive's properties and block signature match, and after a second read of the source.
- Idempotent by design. The archive stores a link to its source, so a rerun after a partial failure finds the existing archive instead of duplicating it.
- Rollback of its own work only. If a copy fails after creation, the archive created in this pass is trashed; a pre-existing archive is never touched.
- Everything unusual is kept, not guessed: Notion-hosted files, unsupported block types, very large pages, a task edited during the run, or duplicate archives all leave the task in TODO with a reason in the log.
- A dry-run mode copies and verifies but keeps the source (see Setup).

## Setup

Credentials:

- `notionApi` credential (Notion internal integration), selected in the `Notion API` node. Share both databases with the integration.
- `telegramApi` credential, selected in the `Telegram alert` node.

Placeholders to replace:

- `YOUR_TODO_DATABASE_ID` and `YOUR_DONE_DATABASE_ID` at the top of the `Archive engine` code node (database ids, not data source ids; databases with a single source).
- `YOUR_TELEGRAM_CHAT_ID` in the `Telegram alert` node.

Notion properties expected (names are configurable in the `P` object and `SELECTS` / `STAGE_DONE` constants at the top of the code node):

- TODO database: `Task` (title), `Stage` (select, with a `Done` option), `Is done` (checkbox), `Finish date` (date), plus the select properties `Project`, `Priority`, `Category`.
- DONE database: `Task` (title), `Source task` (url), `Archived on` (date), `Completed date` (date), plus the same select properties with the same options as TODO.
- `NOTION_PAGE_BASE_URL` (default `https://www.notion.so/`) is the prefix of the link written to `Source task` in DONE; the page id, without dashes, is appended. Notion redirects `notion.so/<page id>` links to the page for members of the workspace, so the default works for any workspace. Change it only if you prefer another domain, for example a custom or workspace-specific one. The duplicate check searches DONE by page id, not by domain, so changing the prefix later does not create duplicate archives.
- Set `FINISH_FALLBACK` to `'archive'` if you want the archive date in `Completed date` when `Finish date` is empty; `'empty'` leaves it empty. `TIMEZONE` defaults to `UTC`.

Testing without risk: add a Webhook trigger (or edit the engine's initial state) that passes a JSON body to the `Archive engine` node. Three optional test parameters are read from it:

- `dryRun` (boolean, default `false`): when `true`, the task is copied and verified but the source is never trashed, and the summary is flagged as a test (no Telegram alert).
- `prefix` (string, default empty): restricts the run to tasks whose title starts with it, for example `"[TEST]"`.
- `simErr` (string, default empty): failure simulation. When a task's title equals this string exactly, the page creation is sent to a nonexistent database id so that Notion returns a real error. Use it with a `[TEST]` task to check that the failure is caught, the task is kept in TODO, and the error appears in the summary.

For example `{"dryRun": true, "prefix": "[TEST]", "simErr": "[TEST] fail on purpose"}`. An example of the expected layout is in `examples/databases.md`.

## Limits

- Notion-hosted files and images are not copied (only external URLs), so such tasks stay in TODO.
- Tasks with more than 800 blocks, or tables over 100 rows, are kept.
- Relations, people and other property types are not copied, only the title, the configured selects and the dates.
- The Notion API version is pinned to 2022-06-28.
- It does not restore from the trash; recovery is manual within 30 days.

Derived from a workflow that runs on a real store, generalized so you can adapt it: property names and select options are values to set for your own databases.
