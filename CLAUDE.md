# Archi Database Plugin — Project Notes

## Project

Eclipse plugin for the Archimate Tool (`com.archimatetool.editor`) that exports/imports models to/from a central relational database and a graph database.

- Current version: **4.9.8** (`sources/META-INF/MANIFEST.MF`)
- Target: **JavaSE-17**
- Build: pure Eclipse PDE (`.classpath`, `.project`, `build.properties`, `MANIFEST.MF`) — no Maven/Gradle
- Author: Herve Jouin (single maintainer)
- License: see `license.txt`
- Supported backends: PostgreSQL, MySQL, MS SQL Server, Oracle, SQLite, Neo4J (drivers bundled in `sources/lib/`)
- Entry points: `DBExporter` (export handler) and `DBImporter` (import handler), wired via `sources/plugin.xml`
- Plugin activator: `org.archicontribs.database.DBPlugin`

## Structure

~25k lines of Java, ~75 files.

| Package | Role | Notable sizes |
|---|---|---|
| `org.archicontribs.database` | Plugin lifecycle, logging, DB entry config, importer/exporter entry points | `DBPlugin` 390, `DBDatabaseEntry` 582 |
| `.connection` | JDBC layer | `DBDatabaseConnection` 1910, `DBDatabaseExportConnection` 2321, `DBDatabaseImportConnection` 1608, `DBStatement` 219 |
| `.gui` | SWT dialogs | `DBGuiExportModel` 3409, `DBGuiImportComponents` 2638, `DBGui` 1641, `DBGuiImportModel` 1161 |
| `.model` | Archi model extensions, EMF factory override | `DBArchimateModel` 915, `DBMetadata` 1174 |
| `.model.commands` | GEF undoable import/delete commands | ~2.5k combined |
| `.menu`, `.preferences`, `.commandline`, `.data`, `.model.propertysections` | Supporting UI, CLI hooks, value objects | — |

## Key findings

### Positives

- Parameterized SQL throughout via `DBStatement` (`sources/src/org/archicontribs/database/connection/DBStatement.java`) — no string concatenation into queries. Supports String/Integer/Timestamp/Boolean/ArrayList/byte[] parameters.
- `AutoCloseable` on connections and statements; try-with-resources friendly.
- No `printStackTrace()` anywhere — errors route through log4j/reload4j.
- Clean EMF factory override (`DBArchimateFactory`) keeps Archi model decoration separate.
- Undo/redo via GEF command pattern (`model/commands/DBImport*FromIdCommand.java`).
- PostgreSQL-specific transaction handling: savepoint + rollback-on-error (`DBStatement.java:91-110`).

### Concerns / bugs

1. **`plugin.xml:74` references missing class** `org.archicontribs.database.menu.DBMenuMergeModelsHandler` — no such file exists. Dead extension; will throw when `mergeModelsCommand` is invoked.
2. **Weak password "encryption"** — `DBDatabaseEntry.getCipher()` at line 529:
   - Blowfish (legacy).
   - Key derived from first NIC's MAC address — obfuscation, not security.
   - Hardcoded fallback key `"VerySimpleKey."` if MAC enumeration fails (line 559).
   - No IV, no salt, no authentication tag.
3. **God classes**: `DBGuiExportModel` (3.4k), `DBDatabaseExportConnection` (2.3k), `DBGui` (1.6k), `DBMetadata` (1.2k).
4. **No tests** — no `test/` dir, no JUnit dependency. Commit log is dominated by "fix X for Y database" entries across 5 SQL dialects.
5. **Legacy collections** — `Hashtable` used in four files.
6. **Static singletons** (`DBPlugin.INSTANCE`, static `preferenceStore`, static `logger`) — standard Eclipse-plugin pattern but blocks unit testing.
7. **String-typed preference keys** duplicated as magic strings across `DBPlugin.java:194-225` and `DBDatabaseEntry.java:~410-456`. No central constants.
8. TODO/FIXME markers in 12 files; a `TO-DO list:` block in `DBPlugin.java:106` pre-dates several releases.

## Fork direction (decided 2026-04-24)

**Goal**: fork to keep DB connectivity (SQL + Neo4J) as the core, drop unneeded features, add own features, likely simplify the schema.

**Strategy**: strip hard, don't preserve.

**Keep** (reusable core, ~2k LOC):
- `connection/DBDatabaseConnection.java` (schema-init portions will shrink with simpler tables)
- `connection/DBStatement.java` — parameterized-statement helper, keep as-is
- `connection/DBSelect.java`, `connection/DBRequest.java` — thin wrappers
- `DBDatabaseEntry.java` — connection config (consider replacing the encryption with AES-GCM + proper key management, or drop it)
- `DBDatabaseDriver.java`, `DBColumn.java`, `DBColumnType.java`, `DBTable.java` — driver/schema primitives
- `DBLogger.java`, `DBException.java`, `DBPlugin.java` (trim)
- `sources/lib/*.jar` — JDBC drivers

**Drop** (most of the ~23k LOC):
- All `gui/DBGui*.java` dialogs — tied to vanilla-plugin workflows
- History/versioning/checksum machinery — this is where the complexity lives; simplifying tables removes the reason for it (`data/DBChecksum.java`, `data/DBVersion.java`, most of `DBDatabaseExportConnection` and `DBDatabaseImportConnection`)
- `model/commands/DBImport*FromIdCommand.java` — coupled to the current import workflow
- `menu/*` handlers — rebuild around new feature set
- `DBCheckAndUpdatePlugin.java`, `DropinsPluginHandler.java` — auto-update machinery, not needed

**Main tradeoff**: simplifying tables breaks compatibility with vanilla-plugin central repositories. Acceptable for a private deployment; dealbreaker for sharing a DB with stock-plugin users.

**Suggested sequencing**:
1. Fix the dead `DBMenuMergeModelsHandler` reference in `plugin.xml` (or delete the extension).
2. Set up a SQLite-backed smoke test harness (no server needed) before touching anything else.
3. Strip GUI + history machinery, verify the connection layer still compiles standalone.
4. Define simplified schema; rewrite export/import against it.
5. Rebuild minimal GUI / menu around the new feature set.

## Build / run

- Requires an Eclipse workspace with Archi's target platform.
- No CI-friendly `./gradlew build` exists; adding one would be a prerequisite to sustainable maintenance.
- Bundle-ClassPath in `MANIFEST.MF` lists all bundled JARs — any lib change must update both `MANIFEST.MF` and `build.properties`.
