A thin SQLite wrapper for Zig

Consider using [zig-sqlite](https://github.com/vrischmann/zig-sqlite) for a more mature, full-featured library that better leverages Zig.

```zig
const read_ony = false;
// good idea to pass EXResCode to get extended result codes (more detailed errorr codes)
const flags =  zqlite.OpenFlags.Create | zqlite.OpenFlags.EXResCode;
var conn = try zqlite.open("/tmp/test.sqlite", read_ony, flags);
defer conn.deinit();

try conn.exec("create table test (name text)", .{});
try conn.exec("insert into test (name) values (?1), (?2)", .{"Leto", "Ghanima"});

{
    if (try conn.row("select * from test order by name limit 1", .{})) |row| {
      defer row.deinit();
      std.debug.print("name: {s}\n", .{row.text(0)});
    }
}

{
    var rows = try conn.rows("select * from test order by name", .{});
    defer rows.deinit();
    while (rows.next()) |row| {
        std.debug.print("name: {s}\n", .{row.text(0)});
    }
    if (rows.err) |err| return err;
}
```

# Conn
The `Conn` type returned by `open` has the following functions:

* `row(sql, args) !?zqlite.Row` - returns an optional row
* `rows(sql, args) !zqlite.Rows` - returns an iterator that yields rows
* `exec(sql, args) !void` - executes the statement,
* `execNoArgs(sql) !void` - micro-optimization if there are no args, `sql` must be a null-terminated string
* `changes() usize` - the number of rows inserted/updated/deleted by the previous statement
* `lastInsertedRowId() i64` - the row id of the last inserted row
* `lastError() [*:0]const u8` - an error string describing the last error
* `transaction() !void` and `exclusiveTransaction() !void` - begins an transaction
* `commit() !void` and `rollback() void` - commits and rollback the current transaction
* `prepare(sql, args) !zqlite.Stmt` - returns a thin wrapper around a `*c.sqlite3_stmt`. `row` and `rows` wrap this type.
* `deinit() void` and `deinitErr() !void` - closes the database. `deinit()` silently ignores any error, if you care about the error, use `deinitErr()`

# Row and Rows
Both `row` and `rows` wrap an `zqlite.Stmt` which itself is a thin wrapper around an `*c.sqlite3_stmt`. 

While `zqlite.Row` exposes a `deinit` and `deinitErr` method, it should only be called when the row was fetched directly from `conn.row(...)`:

```zig
if (conn.row("select 1", .{})) |row| {
    defer row.deinit();  // must be called
    std.debug.print("{d}\n", .{row.int(0)});

```

When the `row` comes from iterator `rows`, `deinit` or `deinitErr` should not be called on the individual row:

```zig
var rows = try conn.rows("select 1 union all select 2", .{})
defer rows.deinit();  // must be called

while (rows.next()) |row| {
    // row.deinit() should not be called!
    ...
}
```

Note that `zqlite.Rows` as an `err: ?anyerror` field which can be checked at any point. Calls to `next()` when `err != null` will return null. Thus, `err` need only be checked at the end of the loop:

```zig
var rows = try conn.rows("select 1 union all select 2", .{})
defer rows.deinit();  // must be called

while (rows.next()) |row| {
    ...
}

if (rows.err) |err| {
    // something went wrong 
}
```

## Row Getters
A `row` exposes the following functions to fetch data:

* `boolean(index) bool`
* `nullableBoolean(index) ?bool`
* `int(index) i64`
* `nullableInt(index) ?i64`
* `float(index) i64`
* `nullableFloat(index) f64`
* `text(index) []const u8`
* `nullableText(index) ?[]const u8`
* `blob(index) []const u8`
* `nullableBlob(index) ?[]const u8`

The `nullableXYZ` functions can safely be called on a `not null` column. The non-nullable versions avoid a call to `sqlite3_column_type` (which is needed in the nullable versions to determine if the value is null or not).

# Transaction:
The `transaction()`, `exclusiveTransaction`, `commit` and `rollback` functions are simply wrappers to `conn.execNoArgs("begin")`, `conn.execNoArgs("begin exclusive")`, `conn.execNoArgs("commit")` and `conn.execNoArgs("rollback")`

```zig
conn.transaction()
errdefer conn.rollback();

try conn.exec(...);
try conn.exec(...);
try conn.commit();
```

# Blobs
When binding a `[]const u8`, this library has no way to tell whether the value should be treated as an text or blob. It defaults to text. To have the value bound as a blob use `zqlite.blob(value)`:

```zig
conn.insert("insert into records (image) values (?1)", .{zqlite.blob(image)})
```

However, this should only be necessary in specific cases where SQLite blob-specific operations are used on the data. Text and blob are practically the same, except they have a different type.
