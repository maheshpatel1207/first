# PostgreSQL Debugger for VS Code: Plan and Requirements

Oct 17, 2026 · @Mahesh

## Overview and goals

We will build a VS Code extension that debugs PL/pgSQL functions and procedures line by line, with breakpoints, using only `CREATE FUNCTION`, `CREATE PROCEDURE` and `CREATE TABLE` rights. No server plugin, restart or superuser is needed.

The extension reads a routine, builds an instrumented copy of it, and runs the copy. A hook before each statement reports the line and variables to VS Code and pauses when a breakpoint or a step requires it. VS Code resumes the routine through a small control table.

**Goals**

- Run, pause, step into, step over, step out, continue and stop from VS Code's standard debug toolbar.
- Gutter breakpoints, plus conditional breakpoints and logpoints.
- Variables, call stack and watch values for the paused routine, including the routines it calls.
- Safe by default: each run happens in a transaction that is rolled back unless the user chooses to commit.
- Works on managed databases (RDS, Cloud SQL, Azure) where `plugin_debugger` cannot be installed.

**Non-goals for v1**

- Plain SQL statements, trigger functions, and languages other than PL/pgSQL.
- Changing variable values while paused.
- Debugging the original routine in place. The debugger always runs a copy, so anything that depends on the original object itself can behave differently.
- Attaching to an application's existing database session.

If `plugin_debugger` becomes available later, it can be added as a second backend behind the same VS Code interface.

## How it works

VS Code controls a routine running on the server through two database connections: the **target connection** runs the code under debug, and the **control connection** sends it instructions.

&#91;embedded content: architecture · VS Code, adapter, two connections, dbg tables, instrumented copy\]

The adapter writes commands on the control connection and calls the copy on the target connection. The copy reads the control tables at each statement and sends a stop report back whenever it pauses.

**One debug session, step by step**

1. The user starts a launch configuration (routine and arguments). The adapter opens both connections and creates a row in `dbg.session`.
2. The adapter reads the routine, parses it, injects hooks, and creates the instrumented copy, plus copies of the routines it calls, in a scratch schema.
3. The adapter writes the editor's breakpoints to `dbg.breakpoint` and commits. It also writes the first command: `step_into` if the user chose to stop on entry, otherwise `continue`.
4. On the target connection it runs `BEGIN`, puts the scratch schema first in `search_path`, sets `statement_timeout` to 0, and calls the copy.
5. Before each statement, the copy calls `dbg.should_pause`. It returns true at a breakpoint, or when the latest command is a step that applies at this point.
6. When it returns true, `dbg.pause` sends a report with the line, variables and call stack. The adapter receives it while the query is still running and tells VS Code the program has stopped.
7. `dbg.pause` then polls `dbg.command` every 100 ms for a command newer than the one it started with.
8. The user clicks Step or Continue. The adapter inserts a command row on the control connection and it commits at once. The poll sees it, `dbg.pause` returns, and the routine runs on to its next stop.
9. When the routine ends or the user stops, the adapter ends the transaction with `ROLLBACK` (or `COMMIT` if chosen), drops the scratch schema and removes the session rows.

**Three PostgreSQL behaviors make this possible without a plugin**

- A message raised with `RAISE INFO` reaches the client immediately, even while the query is still running.
- Under `READ COMMITTED`, each statement inside a `VOLATILE` function takes a fresh snapshot, so the paused target sees rows that other sessions have committed.
- `GET DIAGNOSTICS ... = PG_CONTEXT` returns the PL/pgSQL call stack, so call depth and stack frames need no counters.

## Requirements

The debugger needs an ordinary developer role, a direct database connection and PL/pgSQL code. The extension builds everything else itself.

**Server prerequisites**

| Requirement | Why it matters | How the extension checks |
| --- | --- | --- |
| Rights to create functions, procedures and tables in a schema you own | Holds the hook functions, the control tables and the instrumented copies | "Install debug support" tries each and reports exactly what failed |
| `CREATE` on the database (to make one scratch schema per session) | Lets nested calls resolve to copies through `search_path` | Falls back to prefix mode (copies named `dbg_<id>_name` in your schema) if missing |
| `EXECUTE` on the routines to debug, and rights on the tables they use | The copy runs with the debugging user's rights, not the owner's | `has_function_privilege` check before launch |
| A direct connection, not a transaction-mode pooler (PgBouncer, RDS Proxy) | The target must stay on one backend for the whole run | Compares `pg_backend_pid()` between statements and warns on a change |
| `READ COMMITTED` isolation, the default | Lets the paused target see commands the control connection has committed | The adapter sets it on the target transaction |
| Two free connections per debug session | One runs the code, one sends commands | Reads `max_connections` and current usage |
| PostgreSQL 12 or later | Procedures and `GET DIAGNOSTICS ... PG_CONTEXT` for the call stack | Reads `server_version_num` (a cautious floor; older versions may work) |

**Client prerequisites**

- Current stable VS Code on Windows, macOS or Linux.
- Network access from the developer machine to the database, with TLS where the server requires it.
- Nothing else to install: the SQL parser ships inside the extension as WebAssembly.

**Non-functional requirements**

- **Safety.** The original routines are never modified. Everything the debugger creates lives in the `dbg` schema and in `dbg_s<id>` scratch schemas, and a "Clean up debug objects" command removes it. Connections tagged `prod` refuse to debug unless the user overrides it.
- **Reliability.** A crashed VS Code must not leave a routine hanging. The target aborts itself if the adapter's heartbeat stops for 30 seconds, and any single pause ends after 15 minutes (both configurable).
- **Responsiveness.** A step should reach the screen in under 300 ms on a local network: a 100 ms poll plus one round trip. These figures are targets to confirm in the spike.
- **Overhead.** A run with no breakpoints should stay within a small multiple of normal run time. The spike sets the exact target from measurements.
- **Security.** Passwords go in VS Code SecretStorage. Source code is sent only to the user's own database. Debug-console SQL runs with the user's own privileges.
- **Transparency.** An output channel logs every statement the adapter sends, and "Show instrumented SQL" shows exactly what runs.

## Features by release

The MVP proves the pause-and-resume loop on a single routine, v1 makes it usable for a team, and later releases add the power features.

| Release | Feature | Notes |
| --- | --- | --- |
| MVP | Launch a function or procedure with arguments from a saved connection | Source comes from the database as a read-only virtual document |
| MVP | Line breakpoints and continue | Stored in `dbg.breakpoint`, re-read at every pause, so they can change while paused |
| MVP | Step into, step over, step out | Depth comes from the PL/pgSQL call stack |
| MVP | Variables panel and call stack | Parameters and locals shown as JSON, nested values expandable |
| MVP | Stop, roll back and clean up | Rollback is the default ending |
| MVP | "Install debug support" command | Creates the `dbg` schema, tables and hook functions; safe to run twice |
| v1 | Nested calls | Scratch schema holds copies of called routines; step into them; full stack |
| v1 | Debug from workspace `.sql` files as well as the database | Warns when the file and the database version differ |
| v1 | Conditional breakpoints and logpoints | Conditions are built into the copy when the session starts |
| v1 | Pause while running | Sends a step command; stops at the next statement |
| v1 | Watch variable names and field paths | For example `rec.total` or `items[1]` |
| v1 | Procedures with OUT parameters, set-returning functions, `DO` blocks |  |
| v1 | The routine's own `RAISE NOTICE` output in the Debug Console | Hook messages are filtered out |
| v1 | "Show instrumented SQL" diff view | Lets users trust what actually runs |
| v1 | Routine explorer, "Debug this function" CodeLens, argument prompts |  |
| v1 | Safe mode (read-only transaction) and production guard |  |
| v1.1 | Debug-console SQL inside the paused transaction | Sees uncommitted data; locals usable as `:name` |
| v1.1 | Watch expressions in full SQL | Same mechanism as the debug console |
| v1.1 | Break on error | Pauses in an exception handler with variables and the error text |
| v1.1 | Trace mode | Runs without pausing, records every line and its variables, then lets you browse them |
| v1.1 | Return value and result-set viewer |  |
| Later | Listen mode for development databases | Copies installed under the real names so an application call stops in VS Code; sandbox use only |
| Later | Per-line timing from trace timestamps |  |
| Later | Break when a variable changes |  |
| Later | Optional `plugin_debugger` backend | True debugging where the plugin exists |
| Later | Editing variable values | Needs a hook that reads overrides |

## Data model and hook functions

Four small tables and three hook functions carry all communication, and the target connection only ever reads the tables. Everything lives in a `dbg` schema installed once per database.

```sql
create schema if not exists dbg;

create table dbg.session (
  id             bigint generated always as identity primary key,
  owner_name     text        not null default current_user,
  target_pid     int,
  scratch_schema text        not null,
  status         text        not null default 'starting', -- starting, running, paused, finished, aborted
  heartbeat_at   timestamptz not null default now(),
  created_at     timestamptz not null default now()
);

create table dbg.routine (
  session_id    bigint not null references dbg.session(id) on delete cascade,
  routine_id    int    not null,  -- number baked into the copy
  original_sig  text   not null,  -- app.calc_total(integer,integer)
  copy_sig      text   not null,  -- dbg_s12.calc_total(integer,integer)
  body_offset   int    not null,  -- lines in the file view before body line 1
  primary key (session_id, routine_id)
);

create table dbg.breakpoint (
  session_id    bigint  not null references dbg.session(id) on delete cascade,
  routine_id    int     not null,
  line          int     not null,  -- body-relative; first body line is 1
  enabled       boolean not null default true,
  primary key (session_id, routine_id, line)
);

create table dbg.command (
  session_id    bigint not null references dbg.session(id) on delete cascade,
  seq           bigint generated always as identity,
  cmd           text   not null,  -- continue, step_into, step_over, step_out, eval
  ref_depth     int,              -- call depth of the stop this command answers
  arg           text,             -- SQL text for eval
  created_at    timestamptz not null default now(),
  primary key (session_id, seq)
);
```

| Function | Called | What it does |
| --- | --- | --- |
| `dbg.should_pause(sid, routine_id, line)` | Before every statement | Cheap check. Returns true if the line has an enabled breakpoint, or if the latest command is a step that applies at the current call depth |
| `dbg.pause(sid, routine_id, line, vars)` | Only when `should_pause` returned true | Remembers the highest `seq`, sends the stop report, then polls every 100 ms for a newer command, sending a keep-alive report every 20 seconds. Runs `eval` SQL and reports the result. Returns on any resume command |
| `dbg.log(sid, routine_id, line, text)` | At logpoints | Sends the message to the Debug Console without pausing |

**Rules that make it work**

- **The target never writes to `dbg` tables.** Its writes would stay invisible until its transaction ends, and the final rollback would undo them. State lives in the commands the adapter writes.
- **Hooks are `VOLATILE`.** Each statement inside then takes a fresh snapshot under `READ COMMITTED` and sees what the control connection just committed.
- **Reports go out as `RAISE INFO`.** INFO always reaches the client, even if the session sets `client_min_messages` higher. The payload is `DBG|<session id>|<json>` with the routine id, line, variables, and the raw `PG_CONTEXT` text, which the adapter parses into stack frames.
- **Stepping is stateless on the server.** The adapter stores the stop's call depth in the next command's `ref_depth`. Step over stops at the next statement at that depth or shallower; step out stops at a shallower depth; step into stops at the next statement.
- **Stop uses `pg_cancel_backend(target_pid)`.** It interrupts the sleep loop at once, and `EXCEPTION WHEN OTHERS` handlers in user code cannot catch it.
- **Orphans end themselves.** If `heartbeat_at` is older than 30 seconds, or a pause exceeds its limit, the hook ends its own backend with `pg_terminate_backend(pg_backend_pid())`.
- **The target session runs with `statement_timeout = 0`** (set locally), because the whole debug run is one long statement.

## Instrumentation engine

The engine turns a routine's source into a debuggable copy by inserting text only, never adding or removing a line, so every reported line number matches the editor. It is the riskiest component and gets the first real prototype.

**Pipeline**

1. **Fetch.** Read the definition with `pg_get_functiondef(oid)`, or take the text of the open `.sql` file.
2. **Parse.** The extension's own tokenizer finds the start offset of each statement, handling strings, dollar quotes, comments and nested blocks. `libpg-query` (WebAssembly, with a PL/pgSQL parser that reports a line per statement) cross-checks statement kinds and lines and finds function calls inside SQL. Confirm its exact API in the spike.
3. **Inject.** Put a hook call before each steppable statement, on the same line.
4. **Rewrite the header.** Point the name at the scratch schema and add that schema to any `SET search_path` clause. Keep language, volatility, `SECURITY` setting and parameters identical.
5. **Create.** Run `CREATE OR REPLACE` for the copy and record it in `dbg.routine`.
6. **Resolve callees.** Repeat for the routines the copy calls (see nested calls below).

For a statement on line 14 of routine 3 in session 12, the injected text looks like this. Session and routine ids are constants, because each session has its own copy.

```sql
IF dbg.should_pause(12, 3, 14) THEN PERFORM dbg.pause(12, 3, 14, jsonb_build_object('qty', qty, 'total', total)); END IF; total := total + qty * price;
```

**Where hooks go**

| Construct | Hook placement |
| --- | --- |
| Assignment, `PERFORM`, `SELECT INTO`, `INSERT`, `UPDATE`, `DELETE`, `EXECUTE`, `CALL`, `RAISE`, `RETURN`, `RETURN NEXT`, `RETURN QUERY`, `GET DIAGNOSTICS`, `EXIT`, `CONTINUE` | Immediately before the statement |
| `IF` and `CASE` | Before the statement, then before each statement in every branch |
| `ELSIF` and `WHEN` conditions | Not steppable in v1. A breakpoint there is moved to the next statement and VS Code shows the adjusted line |
| `LOOP`, `WHILE`, `FOR`, `FOREACH` | Before the loop header, and first in the loop body so every iteration stops |
| Nested `BEGIN ... END` blocks and `EXCEPTION` handlers | Before each statement inside, including the first statement of a handler |
| `DECLARE` initial values | Not steppable. The values appear at the first stop |
| `COMMIT` and `ROLLBACK` in procedures | Hook plus a warning: they end the debug transaction early, so the rollback safety no longer applies |

**Variables.** At a stop, the copy builds one JSON object of everything in scope: parameters, `DECLARE`d variables of enclosing blocks, and loop variables. The build sits inside the `IF`, so its cost is paid only when pausing. `jsonb_build_object` takes at most 50 pairs, so larger scopes are joined with `||`. Values over 4 KB are cut and marked as truncated. Unassigned `record` variables raise an error when converted, so the injector guards their capture.

**Nested calls.** The adapter creates a scratch schema `dbg_s<id>`, copies each PL/pgSQL routine the target can reach, and runs the call with `search_path` set to the scratch schema first. Unqualified calls, including ones built by dynamic SQL, then find the copies. Schema-qualified calls such as `app.calc_tax(...)` are rewritten by the injector. Reachable routines are found by following calls in the parsed bodies, limited to the schemas in settings, a configurable depth and 50 routines. Routines in other languages run as the originals, and a SQL function that calls back into PL/pgSQL reaches the original, so that call runs without stops.

**Prefix mode.** If the user cannot create schemas, copies go into their own schema as `dbg_<id>_<name>` and the injector rewrites calls by name. Calls hidden in dynamic SQL are not intercepted in this mode.

**Line numbers.** The copy keeps every original line, so line N in a stop report is line N of the original body. `dbg.routine.body_offset` adds the header lines when the editor shows the full `CREATE FUNCTION` text. The spike must confirm how PostgreSQL counts the first line, because error contexts count from the start of the body string.

**Call stack.** `PG_CONTEXT` lists every PL/pgSQL frame with its function and current line. The adapter maps copy names back to the originals. A calling frame shows the variables from its last report, which is its state when it made the call. In continue mode a frame may not have reported yet, so v1 shows "not captured" there, and v1.1 makes statements that call another copy always report.

**Edge cases for the test corpus.** Labels before loops (`<<outer>>`), nested dollar quotes (`$a$ ... $b$`), `FOR ... IN EXECUTE`, `CASE` in both forms, several statements on one line, `EXIT WHEN`, unnamed parameters (`$1`), and `RETURNS TABLE` functions.

## VS Code extension design

The extension is a thin VS Code layer around one debug adapter that runs inside the extension host and speaks the Debug Adapter Protocol (DAP). VS Code supplies the breakpoint gutter, toolbar, variables, call stack and watch panels, so we write no custom debug UI.

**Components**

- **Extension (TypeScript).** Activation, connection manager, routine explorer, commands, CodeLens and the virtual document provider.
- **Debug adapter.** A `PgDebugSession` class built on `@vscode/debugadapter` and run inline with `DebugAdapterInlineImplementation`. It owns the two `pg` connections and the session state.
- **Injector library.** Pure TypeScript with no VS Code dependency, so it can be unit tested in isolation.
- **Parser.** `libpg-query` as WebAssembly, bundled with the extension.
- **Server support scripts.** Versioned SQL for the `dbg` schema, bundled and applied by "Install debug support".

**Launch configuration.** Arguments are SQL expressions passed exactly as written.

```json
{
  "type": "pgsql",
  "request": "launch",
  "name": "Debug app.calc_total",
  "connection": "dev-db",
  "routine": "app.calc_total(integer, date)",
  "args": ["42", "'2026-01-01'"],
  "source": "database",
  "stopOnEntry": true,
  "commit": false,
  "readOnly": false,
  "instrument": { "schemas": ["app"], "maxDepth": 3 }
}
```

**DAP mapping**

| DAP request or event | Adapter behavior |
| --- | --- |
| `initialize` | Advertises conditional breakpoints, logpoints, terminate and configurationDone. No step-back, no setVariable |
| `launch` | Opens both connections, checks that support is installed, builds the copies, creates the session row |
| `setBreakpoints` | Maps editor lines to routine and body line, moves unsteppable lines and returns the adjusted line. While running, rewrites `dbg.breakpoint` for that routine |
| `configurationDone` | Starts the target: `BEGIN`, local settings, then the call, without waiting for it to finish |
| `threads` | One thread named "PL/pgSQL session" |
| `stackTrace`, `scopes`, `variables` | Built from the latest stop report. Nested JSON becomes expandable `variablesReference` entries |
| `continue`, `next`, `stepIn`, `stepOut` | Insert a command row with the current `ref_depth`, send `continued`, then send `stopped` when the next report arrives |
| `pause` | While running, inserts a `step_into` command |
| `evaluate` | v1: variable names and paths from the last report. v1.1: sends an `eval` command |
| `terminate`, `disconnect` | Cancels the target, ends with `ROLLBACK` or `COMMIT` per setting, drops the scratch schema, deletes the session rows |
| `stopped` event | Reasons: entry, breakpoint, step, pause (and exception in v1.1) |
| `output` event | The routine's own notices, logpoints and hook errors |

**Showing the source**

- **Database mode.** A read-only virtual document `pgdebug://<connection>/<schema>/<signature>.sql` holds the text of `pg_get_functiondef`. VS Code allows breakpoints in any document whose language is listed in the `breakpoints` contribution.
- **File mode (v1).** The user's own `.sql` file. The adapter finds the `CREATE FUNCTION` that matches the launch configuration and instruments that text. It warns when the database's definition differs and offers to debug the file's version.

**Interface**

- **Commands:** Add connection, Install debug support, Debug routine, Show instrumented SQL, Clean up debug objects.
- **Explorer view:** connections, then schemas, then routines, with Debug on the context menu. A CodeLens above each `CREATE FUNCTION` in `.sql` files.
- **Argument prompt:** parameter names and defaults come from `pg_proc`; the last values are remembered per routine.
- **Status bar:** the connection and whether a session is running, paused or idle.
- **Settings:** `pgDebugger.pollIntervalMs` (100), `maxPauseMinutes` (15), `heartbeatSeconds` (30), `defaultCommit` (false) and `prodPatterns` for the production guard. Connection profiles hold no passwords.

## Build plan

Build in six phases over about 10 weeks for one developer, or 6 to 7 weeks for two (these are estimates), proving the riskiest parts first. Each phase ends at a gate that must pass before the next begins.

&#91;embedded content: build roadmap · 6 phases, 6 gates\]

Phases are drawn at equal width, not to scale, and the week counts are estimates.

- **Phase 0, spike (week 1).** In `psql` with two terminals: create the `dbg` tables and hooks, hand-instrument one routine, and prove pause, resume and `RAISE INFO` delivery. Answer the open questions: line numbering, `PG_CONTEXT` format, privileges on your platform, step latency, hook overhead. *Gate: pause and resume work on the real target database.*
- **Phase 1, injector (weeks 2 and 3).** Tokenizer, hook placement table, header rewrite, variable capture, golden-file tests and the differential test harness. A command-line tool prints the instrumented SQL. *Gate: every corpus routine compiles and passes the differential test.*
- **Phase 2, adapter core (weeks 4 and 5).** Scaffold with `yo code`, add the `debuggers` contribution, `PgDebugSession`, connections and "Install debug support". Launch, line breakpoints, continue, the three step commands, stop and cleanup, variables and stack for one routine, and the database source view. *Gate: alpha, a developer debugs one routine end to end.*
- **Phase 3, nested calls and robustness (weeks 6 and 7).** Scratch schema with `search_path`, callee discovery, the full call stack, stepping across calls, heartbeat and orphan handling, prefix mode, conditional breakpoints and logpoints, procedures, set-returning functions and `DO` blocks. *Gate: beta, the failure-injection tests pass.*
- **Phase 4, experience (weeks 8 and 9).** Routine explorer, CodeLens, argument prompts, file mode with the mismatch warning, safe mode, production guard, Debug Console output, settings and documentation. *Gate: three developers who did not build it debug a routine without help.*
- **Phase 5, hardening and release (week 10).** Version matrix, managed-service checklist, performance pass, README with the limitations, packaging with `vsce` and publishing. *Gate: v1.0 meets the exit criteria in the testing section.*

v1.1 (debug-console SQL, break on error, trace mode) follows as a separate four-week effort.

## Testing and quality

Correctness rests on one property: an instrumented routine run without stops must behave exactly like the original. Most of the test plan exists to prove that.

- **Injector unit tests.** Golden files map an input routine to the expected instrumented text. Every test also checks that the line count is unchanged and that removing the injected text gives back the original. The corpus is the PL/pgSQL regression tests from the PostgreSQL source tree plus the team's own routines.
- **Differential tests.** Run each corpus routine twice in Docker, original and instrumented in continue mode with no breakpoints, and compare results, errors and side effects after rollback.
- **Server tests.** Docker images for each currently supported PostgreSQL major version, plus older ones your servers still run. Check notice delivery, snapshot visibility, `PG_CONTEXT` format and `pg_cancel_backend` behavior on each.
- **Adapter tests.** Drive the adapter with the `DebugClient` from `@vscode/debugadapter-testsupport` using scripted sessions: set a breakpoint, hit it, step, continue, stop, nested call, runtime error, orphan timeout.
- **Extension end-to-end.** `@vscode/test-electron` starts VS Code with the extension against a Docker database.
- **Failure injection.** Kill VS Code mid-pause, drop the network, kill the target backend, set `statement_timeout` at role level, and connect through a pooler. Each case must end cleanly with no stray schemas, sessions or hung backends.
- **Managed-service check before each release.** Run a manual checklist on the real platform (RDS or similar): connection limits, proxy and load-balancer idle timeouts, permission failures.

**Proposed exit criteria for v1**

- Every corpus routine passes the differential test.
- 100 randomized aborted runs leave no leaked schemas, session rows or backends.
- Step latency meets the target on a local network.
- A new user installs support and debugs a routine in under ten minutes without help.

## Risks, limitations and mitigations

The two risks that matter most are an injector that mishandles unusual syntax and a paused session that holds locks. Both have clear mitigations, listed first.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Injector mishandles unusual syntax | Copy fails to compile or behaves differently from the original | Own tokenizer cross-checked by the parser, differential tests, and "Show instrumented SQL" so a failure is easy to read |
| A paused session holds locks and an open transaction | Blocks other users and delays vacuum | Pause limit (15 minutes), a warning at the first pause, read-only safe mode, production guard |
| Side effects that rollback cannot undo | Sequence values, `dblink` or other external calls, and `COMMIT` inside procedures survive the rollback | Parser flags these before launch and asks for confirmation; rollback stays the default |
| A pooler or proxy sits in the path | Hooks fail, or the target moves to another backend | Direct-connection check at launch; documented as unsupported |
| Idle network timeout drops a paused connection | Session lost or target orphaned | The pause loop sends a report every 20 seconds so the line carries traffic, TCP keepalives on both connections, and self-termination of orphans |
| Role-level settings (`statement_timeout`, `lock_timeout`) kill the run | Debug run ends mid-session | Set locally on the target at start, and warn when the role has unusually low values |
| A `SECURITY DEFINER` routine is copied | The copy runs with the debugging user's rights, not the original owner's | Warn at launch and list the objects that may be inaccessible |
| Hook overhead in loops with many iterations | Slow debugging of heavy routines | Continue mode checks breakpoints only; trace mode and conditional breakpoints for long loops |
| Server version drift (`PG_CONTEXT` format, parser grammar) | Stack frames or parsing break | Version matrix in CI; the adapter falls back to a stack without frames rather than failing |
| File and database versions differ | Debugging the wrong code | Compare definitions at launch and warn |
| Several developers debug at once | Name clashes and connection use | Everything is keyed by session id, one scratch schema per session, connection headroom check |

**Limitations to state plainly in the README**

- The debugger runs an instrumented copy, not the original object.
- Only PL/pgSQL is supported. SQL, C and other languages run as the originals.
- Variable values cannot be edited in v1.
- Dynamic SQL runs as one step, and its internals cannot be stepped.
- `ELSIF` and `WHEN` conditions are not separate stops.
- Trigger functions are debugged only through a direct call, not when a table event fires them.
- Calling frames show variables from their last report, which may be missing in continue mode until v1.1.

## Next steps

Start with the spike: it answers the open technical questions before any extension code exists.

- [ ] Check your target database allows `CREATE SCHEMA`, `CREATE FUNCTION` and `CREATE TABLE`, and note its `max_connections` headroom.
- [ ] Pick two routines for the spike and the test corpus: one self-contained, one that calls another.
- [ ] Run the spike by hand in `psql` with two terminals: create the `dbg` tables, hand-instrument one routine, and prove pause, resume and `RAISE INFO` delivery. Measure step latency and hook overhead.
- [ ] Confirm how PostgreSQL numbers the first line of a function body and the exact format of `PG_CONTEXT` on your server version.
- [ ] Decide whether v1 debugs the database copy of a routine, the workspace `.sql` file, or both.
- [ ] Confirm team size and timeline. This plan assumes one developer.
- [ ] Choose the extension name, publisher and license.
