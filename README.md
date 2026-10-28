![Version](https://img.shields.io/badge/version-3.0.0-blue)
![Java](https://img.shields.io/badge/Java-26-orange)
![Depends](https://img.shields.io/badge/depends%20on-axiom-informational)
![License](https://img.shields.io/badge/license-Limited%20Commercial-red)

# Axiom Warp SQLite Driver (`axiom-warp-sqlite-driver`)

> The sovereign, framework-agnostic native SQLite embedded-engine and Panama memory-mapped driver for Axiom.

`axiom-warp-sqlite-driver` talks to SQLite through `libsqlite3` / `sqlite3.dll` using the Java Foreign Function & Memory API (Project Panama).
There is **no JDBC, no connection pool, no reflection and no third-party Java dependency** — just `axiom-spec`
(and transitively `axiom`) on the Java side and the vendor's native client on the other.

---

## Architecture overview

```
 ConfigSource ──► SQLite.connect(config)  ─ curried provisioning ─►  SQLite.Executioner
                                                                     │
          wield(conn -> …)  ── BoundScope(Arena.ofConfined) ──► engage() native connection
                                                                     │
           ┌──────────────────────────┬──────────────────────────────┼──────────────────────────┐
           ▼                          ▼                              ▼                          ▼
   shot(sql)                  dynamicQuery(sql)              fromFile(csv)               transaction(conn, …)
   Strike (raw SQL)           SqlParser + Strike             Hammer (bulk ingest)        commit / rollback
           │                  :java.name → native params     CsvEngine streaming
           └──────────────────────────┴──────────────┬───────────────┘
                                                     ▼
                         Result<PersistentList<PersistentMap<String, Object>>>
                                                     │
                                       disconnect()  ◄── always, success or failure
```

| Package (`com.ensemblu.axiom.warp.sqlite.driver.*`) | Responsibility |
|---|---|
| `api.SQLite` | Static `connect(..)` entry point and the `Executioner` operations gateway |
| `provision.RawProvisioner` | Validates configuration fields and materialises the `Executioner` |
| `engine.Strike` | Raw statements (`shot`) and parameterised queries (`dynamicQuery`) |
| `engine.SqlParser` | Rewrites `:java.name` markers into the native parameter syntax (max **21** per statement) |
| `ingest.Hammer` | Contract-verified bulk CSV ingestion |
| `io.CsvEngine` | Streaming CSV reader (classpath or filesystem) |
| `scope.BoundScope` | Binds a confined `Arena` to the current call via `ScopedValue`, freed when `wield` returns |
| `catalog.SqliteType` | Native type catalog used for bindings and ingest contracts |

**Connection model.** Each `wield(..)` opens a dedicated native connection, runs your lambda, and closes the
connection and its arena before returning — nothing leaks between calls, and nothing is pooled. Keep work that must
share a session (transactions, temp tables) inside a single `wield`.

Dependency graph:

```
axiom  ◄──  axiom-spec  ◄──  axiom-warp-sqlite-driver
```

---

## Installation

**Maven**

```xml
<dependency>
    <groupId>com.ensemblu</groupId>
    <artifactId>axiom-warp-sqlite-driver</artifactId>
    <version>3.0.0</version>
</dependency>
```

**Gradle (Kotlin DSL)**

```kotlin
implementation("com.ensemblu:axiom-warp-sqlite-driver:3.0.0")
```

JPMS automatic module name: `com.ensemblu.axiom.warp.sqlite.driver`

### Runtime requirements

| Requirement | Value |
|---|---|
| JDK | **26**, run with `--enable-preview` |
| Native access | `--enable-native-access=ALL-UNNAMED` (avoids restricted-method warnings) |
| Native client | `libsqlite3` / `sqlite3.dll` |

Looked up as `sqlite3` on Windows and `libsqlite3` elsewhere. Download the precompiled DLL from sqlite.org and put it on `PATH` (Windows), or install your OS package (`libsqlite3`).

---

## Configuration guide

Create a properties file on the classpath (e.g. `src/main/resources/database.properties`):

```properties
engine.path=data/axiom.db
engine.readonly=false
```

| Key | Required | Notes |
|---|---|---|
| `engine.path` | yes | Database file path, or `:memory:`; must be non-blank |
| `engine.readonly` | no | `true` opens read-only; default `false` (read-write, create if missing) |

Native connect string: `sqlite3_open_v2(path, flags)` — `READWRITE|CREATE`, or `READONLY` when `engine.readonly=true`.

Provisioning is a curried chain that mirrors the required fields, so a missing step is a compile error and a missing
or invalid value fails fast at boot with `🚨 INFRASTRUCTURE BOOT FAILURE`:

```java
import static com.ensemblu.axiom.api.Axiom.Config.file;
import com.ensemblu.axiom.warp.sqlite.driver.api.SQLite;

SQLite.Executioner executioner = SQLite
        .connect(file("database.properties"))//
        .requiredPath()//
        .getSqliteDriverOrThrow();
```

Keep credentials out of source control: load from a file outside the repo, or build the config at runtime with
`Axiom.Config.source("engine.password=" + System.getenv("DB_PASSWORD") + "\n…")`.

---

## Usage examples

### Raw statements

```java
var rows = executioner.wield(conn ->
        executioner.shot("SELECT 1 AS ok").withConnection(conn));

rows.map(list -> list.first().toStringKeyMapVal().get("ok"));   // values arrive as text
```

Run one statement per `shot(..)` call.

### Parameterised queries

```java
import com.ensemblu.axiom.warp.sqlite.driver.catalog.SqliteType;

Result<PersistentList<PersistentMap<String, Object>>> accounts =
        executioner.wield(conn ->
                executioner.dynamicQuery("""
                                SELECT id, balance FROM accounts
                                WHERE status = :java.status
                                LIMIT :java.limit""")
                        .to("ACTIVE").bind(SqliteType.TEXT)
                        .to(10).bind(SqliteType.INTEGER)
                        .execute(conn));
```

Markers are rewritten to positional parameters and bound natively with `sqlite3_bind_*` according to the declared storage class.

Rules: bind **every** marker, **in the order it appears** in the SQL; a repeated name is a separate marker. Result
values are returned as strings — convert at the edge (`row.targetKey("balance").toDoubleVal()`).

### Transactions

```java
Result<String> outcome = executioner.wield(conn ->
        executioner.transaction(conn, () -> {
            executioner.shot("UPDATE accounts SET status = 'FROZEN' WHERE owner = 'X'")
                       .withConnection(conn).getOrThrow();   // throwing → rollback
            executioner.shot("INSERT INTO audit_logs (action_type) VALUES ('FREEZE')")
                       .withConnection(conn).getOrThrow();
            return "committed";
        }));
```

Mechanism: `BEGIN` / `COMMIT`, with `ROLLBACK` on any exception. The failure message is prefixed with `Transaction failed, rolled back:`.

For schema bootstrapping there is a shortcut that runs several statements in one transaction:

```java
executioner.wieldTransactional(
        () -> "CREATE TABLE accounts (...)",
        () -> "CREATE INDEX idx_accounts_status ON accounts(status)"
).getOrThrow();
```

### Bulk CSV ingestion

```java
import com.ensemblu.axiom.warp.sqlite.driver.catalog.SqliteType;

Result<Long> inserted = executioner.wield(conn ->
        executioner.fromFile("csv/initial_accounts.csv")        // classpath first, then filesystem
                   .withConnection(conn)
                   .usingFileHeaders()                          // or .withHeaders("owner", "balance", ...)
                   .withDataTypes(SqliteType.TEXT, SqliteType.FLOAT, SqliteType.TEXT, SqliteType.TEXT)
                   .onTableName("accounts"));
```

One `sqlite3_prepare_v2` statement reused for every row (`bind → step → reset → clear_bindings`) inside an explicit transaction, committed every 1,000 rows — SQLite's documented high-throughput insert path.

Declare exactly **one type per header, in header order**; header names must match column names exactly.

### Error handling

Every operation returns a `Result`; nothing throws unless you call `getOrThrow()`:

```java
accounts
    .peekFailure(e -> Axiom.Io.log("❌ " + e.getMessage()).run())
    .recover(e -> Axiom.Data.emptyList());
```

Pair with [`axiom-telemetry`](../../axiom-telemetry) to attach a pre-failure snapshot to every database failure.

---

## License

**Limited Commercial License** — free for evaluation, testing, and non-commercial development.
Commercial or production use requires a paid annual contract.
See [LICENSE](https://raw.githubusercontent.com/ensemblu-corp/axiom/main/LICENSE) · Contact: contact@ensemblu.com

© Ensemblu Corp
