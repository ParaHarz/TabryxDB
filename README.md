# TabryxDb

<p align="center">
  <strong>An encrypted, single-file database for compact embedded applications.</strong>
</p>

TabryxDb is a small database engine written in Rust. It stores multiple string-based tables in one encrypted file, exposes a stable C ABI, and includes a Windows desktop manager for browsing and editing data.

> This GitHub repository distributes prebuilt binaries. Source code is not included in public releases.

The project is designed for small databases that can be kept in memory. It favors a simple deployment model—one native library and one database file—over the complexity of a client/server database.

## Highlights

- **Single-file storage** — an entire database is stored in one `.tbyx` file.
- **Encryption by default** — the complete serialized database is protected with AES-256-GCM.
- **Password hardening** — keys are derived with Argon2id and a fresh random salt.
- **Tamper detection** — authenticated encryption detects an incorrect password or modified data.
- **Atomic saves** — data is written to a temporary file and then replaced to reduce partial-write corruption.
- **Small SQL subset** — create tables, alter columns, insert, query, update, delete, sort, filter, and paginate.
- **Real SQL `NULL` values** — distinct from empty strings.
- **Simple native integration** — UTF-8 C ABI suitable for C, C++, C#, AutoIt, and other FFI-capable languages.
- **Desktop manager** — a WinForms application for table, row, column, SQL, and export operations.

## Intended Use

TabryxDb is suitable for:

- desktop application settings and structured local data;
- small encrypted catalogs, test profiles, and utility databases;
- embedded tools that need a native DLL without a database server;
- datasets small enough to load fully into memory.

It is not intended to replace SQLite or a client/server database when transactions, concurrent writers, indexes, joins, constraints, or very large datasets are required.

## Architecture

```text
Application
    |
    | C ABI / P/Invoke (UTF-8)
    v
TabryxDb engine (Rust)
    |
    +-- in-memory tables: columns + rows of optional strings
    +-- SQL parser and executor
    +-- bincode serialization
    +-- Argon2id key derivation
    +-- AES-256-GCM authenticated encryption
    v
Single .tbyx file
```

Changes are made in memory. Call `tabryx_save()` explicitly to persist them. Closing a database does **not** save automatically.

## Supported SQL

```sql
CREATE TABLE [IF NOT EXISTS] users (id, name, email)
DROP TABLE [IF EXISTS] users

ALTER TABLE users ADD [COLUMN] age
ALTER TABLE users DROP [COLUMN] email

INSERT INTO users VALUES ('1', 'Alice', NULL)
INSERT INTO users (id, name) VALUES ('2', 'Bob')

SELECT * FROM users
SELECT id, name FROM users
  WHERE name LIKE 'A%' AND email IS NOT NULL
  ORDER BY name ASC
  LIMIT 100 OFFSET 0
SELECT COUNT(*) FROM users WHERE email IS NULL

UPDATE users SET name = 'Alicia' WHERE id = '1'
DELETE FROM users WHERE id = '2'
```

Supported query features include:

- `=` and `!=` / `<>` comparisons;
- `LIKE` with `%` and `_` wildcards;
- `IS NULL` and `IS NOT NULL`;
- `AND` and `OR` expressions (`AND` has higher precedence);
- multi-column `ORDER BY` with `ASC` or `DESC`;
- `LIMIT` and `OFFSET`;
- optional ASCII case-insensitive matching per database handle.

All stored non-null values are strings. TabryxDb currently has no numeric types, primary keys, indexes, joins, subqueries, `GROUP BY`, or concurrent-write coordination.

## Download

Download the latest prebuilt package from [GitHub Releases](../../releases/latest).

Release assets may include:

- **Tabryx Manager** — the Windows desktop database manager;
- **Tabryx SDK** — the native DLL, C header, import library, and integration examples;
- **checksums** — hashes for verifying downloaded archives.

## System Requirements

- 64-bit Windows
- .NET Framework 4.8 for Tabryx Manager

## Run Tabryx Manager

1. Download and extract the Tabryx Manager archive.
2. Keep `TabryxManager.exe` and `tabryx.dll` in the same directory.
3. Run `TabryxManager.exe`.
4. Open an existing `.tbyx` file or enter a new file name and password.
5. Select **Save** after making changes. Changes remain in memory until explicitly saved.

## SDK Package

The native SDK package contains:

```text
tabryx.dll        Native x64 database engine
tabryx.dll.lib    MSVC import library
tabryx.h          Public C API header
LICENSE           MIT license text
examples/         Integration examples, when included
```

Deploy `tabryx.dll` beside your application executable or in another location available to the Windows DLL loader.

## Quick Start with C

```c
#include <stdio.h>
#include "tabryx.h"

int main(void) {
    TabryxStatus status = TabryxStatus_Ok;
    Tabryx *db = tabryx_open("example.tbyx", "strong-password", &status);
    if (!db) {
        fprintf(stderr, "Open failed: %d\n", (int)status);
        return 1;
    }

    if (tabryx_exec(db, "CREATE TABLE IF NOT EXISTS notes (id, text)", NULL)
            != TabryxStatus_Ok ||
        tabryx_exec(db, "INSERT INTO notes VALUES ('1', 'Hello, TabryxDb')", NULL)
            != TabryxStatus_Ok) {
        fprintf(stderr, "SQL error: %s\n", tabryx_last_error(db));
        tabryx_close(db);
        return 1;
    }

    TabryxResult *result = tabryx_query(db, "SELECT * FROM notes", &status);
    if (result) {
        for (int row = 0; row < tabryx_result_row_count(result); ++row) {
            printf("%s: %s\n",
                tabryx_result_value(result, row, 0),
                tabryx_result_value(result, row, 1));
        }
        tabryx_result_free(result);
    }

    if (tabryx_save(db) != TabryxStatus_Ok)
        fprintf(stderr, "Save failed: %s\n", tabryx_last_error(db));

    tabryx_close(db);
    return 0;
}
```

Example MSVC command when the SDK files are in the current directory:

```powershell
cl example.c /I . tabryx.dll.lib
```

Place `tabryx.dll` beside the resulting executable before running it.

## Quick Start with C#

The following example uses P/Invoke directly, so no additional NuGet package is required. Build the application for **x64** and place `tabryx.dll` beside the executable.

```csharp
using System;
using System.Runtime.InteropServices;

internal static class Program
{
    private const string Dll = "tabryx.dll";
    private const CallingConvention Cdecl = CallingConvention.Cdecl;

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern IntPtr tabryx_open(
        [MarshalAs(UnmanagedType.LPUTF8Str)] string path,
        [MarshalAs(UnmanagedType.LPUTF8Str)] string password,
        out int status);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern void tabryx_close(IntPtr db);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern int tabryx_exec(
        IntPtr db,
        [MarshalAs(UnmanagedType.LPUTF8Str)] string sql,
        out int affected);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern IntPtr tabryx_query(
        IntPtr db,
        [MarshalAs(UnmanagedType.LPUTF8Str)] string sql,
        out int status);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern int tabryx_result_row_count(IntPtr result);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern IntPtr tabryx_result_value(IntPtr result, int row, int column);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern void tabryx_result_free(IntPtr result);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern int tabryx_save(IntPtr db);

    [DllImport(Dll, CallingConvention = Cdecl)]
    private static extern IntPtr tabryx_last_error(IntPtr db);

    private static string Utf8(IntPtr value) =>
        value == IntPtr.Zero ? null : Marshal.PtrToStringUTF8(value);

    private static void Main()
    {
        IntPtr db = tabryx_open("example.tbyx", "strong-password", out int status);
        if (db == IntPtr.Zero)
        {
            Console.Error.WriteLine($"Open failed with status {status}.");
            return;
        }

        try
        {
            if (tabryx_exec(db,
                    "CREATE TABLE IF NOT EXISTS notes (id, text)", out _) != 0 ||
                tabryx_exec(db,
                    "INSERT INTO notes VALUES ('1', 'Hello, TabryxDb')", out _) != 0)
            {
                Console.Error.WriteLine("SQL error: " + Utf8(tabryx_last_error(db)));
                return;
            }

            IntPtr result = tabryx_query(db, "SELECT id, text FROM notes", out status);
            if (result == IntPtr.Zero)
            {
                Console.Error.WriteLine("Query error: " + Utf8(tabryx_last_error(db)));
                return;
            }

            try
            {
                for (int row = 0; row < tabryx_result_row_count(result); row++)
                {
                    string id = Utf8(tabryx_result_value(result, row, 0));
                    string text = Utf8(tabryx_result_value(result, row, 1));
                    Console.WriteLine($"{id}: {text}");
                }
            }
            finally
            {
                tabryx_result_free(result);
            }

            if (tabryx_save(db) != 0)
                Console.Error.WriteLine("Save failed: " + Utf8(tabryx_last_error(db)));
        }
        finally
        {
            tabryx_close(db);
        }
    }
}
```

`Marshal.PtrToStringUTF8()` is available in modern .NET. Applications targeting .NET Framework 4.8 should decode the returned null-terminated pointers with `Marshal.Copy()` and `Encoding.UTF8` instead.

## Quick Start with Python

Python can call the native C API directly through the standard-library `ctypes` module. No third-party package is required. Use **64-bit Python** and place `tabryx.dll` beside the script.

```python
import ctypes
from pathlib import Path


dll_path = Path(__file__).with_name("tabryx.dll")
tabryx = ctypes.CDLL(str(dll_path))

# Function signatures. TabryxStatus and Windows C long are both 32-bit values.
tabryx.tabryx_open.argtypes = [
    ctypes.c_char_p,
    ctypes.c_char_p,
    ctypes.POINTER(ctypes.c_int32),
]
tabryx.tabryx_open.restype = ctypes.c_void_p

tabryx.tabryx_close.argtypes = [ctypes.c_void_p]
tabryx.tabryx_close.restype = None

tabryx.tabryx_exec.argtypes = [
    ctypes.c_void_p,
    ctypes.c_char_p,
    ctypes.POINTER(ctypes.c_int32),
]
tabryx.tabryx_exec.restype = ctypes.c_int32

tabryx.tabryx_query.argtypes = [
    ctypes.c_void_p,
    ctypes.c_char_p,
    ctypes.POINTER(ctypes.c_int32),
]
tabryx.tabryx_query.restype = ctypes.c_void_p

tabryx.tabryx_result_row_count.argtypes = [ctypes.c_void_p]
tabryx.tabryx_result_row_count.restype = ctypes.c_int

tabryx.tabryx_result_value.argtypes = [ctypes.c_void_p, ctypes.c_int, ctypes.c_int]
tabryx.tabryx_result_value.restype = ctypes.c_char_p

tabryx.tabryx_result_is_null.argtypes = [ctypes.c_void_p, ctypes.c_int, ctypes.c_int]
tabryx.tabryx_result_is_null.restype = ctypes.c_bool

tabryx.tabryx_result_free.argtypes = [ctypes.c_void_p]
tabryx.tabryx_result_free.restype = None

tabryx.tabryx_save.argtypes = [ctypes.c_void_p]
tabryx.tabryx_save.restype = ctypes.c_int32

tabryx.tabryx_last_error.argtypes = [ctypes.c_void_p]
tabryx.tabryx_last_error.restype = ctypes.c_char_p


def utf8(value):
    return None if value is None else value.decode("utf-8")


status = ctypes.c_int32()
db = tabryx.tabryx_open(
    b"example.tbyx",
    b"strong-password",
    ctypes.byref(status),
)

if not db:
    raise RuntimeError(f"Open failed with status {status.value}")

try:
    affected = ctypes.c_int32()
    statements = [
        "CREATE TABLE IF NOT EXISTS notes (id, text)",
        "INSERT INTO notes VALUES ('1', 'Hello from Python')",
    ]

    for sql in statements:
        result = tabryx.tabryx_exec(
            db,
            sql.encode("utf-8"),
            ctypes.byref(affected),
        )
        if result != 0:
            raise RuntimeError(utf8(tabryx.tabryx_last_error(db)))

    query = tabryx.tabryx_query(
        db,
        b"SELECT id, text FROM notes",
        ctypes.byref(status),
    )
    if not query:
        raise RuntimeError(utf8(tabryx.tabryx_last_error(db)))

    try:
        for row in range(tabryx.tabryx_result_row_count(query)):
            values = []
            for column in range(2):
                if tabryx.tabryx_result_is_null(query, row, column):
                    values.append(None)
                else:
                    values.append(utf8(tabryx.tabryx_result_value(query, row, column)))
            print(f"{values[0]}: {values[1]}")
    finally:
        tabryx.tabryx_result_free(query)

    if tabryx.tabryx_save(db) != 0:
        raise RuntimeError(utf8(tabryx.tabryx_last_error(db)))
finally:
    tabryx.tabryx_close(db)
```

Always declare both `argtypes` and `restype`. Without explicit signatures, `ctypes` may truncate 64-bit database and result pointers. Encode input strings as UTF-8, and decode returned strings before releasing their owning result handle.

## C API Overview

| API | Purpose |
|---|---|
| `tabryx_open` / `tabryx_close` | Open or create a database and release its handle |
| `tabryx_exec` | Execute a non-query SQL statement |
| `tabryx_query` | Execute a query and return a result handle |
| `tabryx_tables` | List tables |
| `tabryx_save` | Encrypt and persist all in-memory changes |
| `tabryx_rekey` | Change the password and immediately rewrite the database |
| `tabryx_set_case_insensitive` | Change comparison behavior for the current handle |
| `tabryx_move_row` | Reorder a row by physical index |
| `tabryx_move_column` | Reorder a column and its values by physical index |
| `tabryx_copy_column` | Copy a column and its values to the rightmost position |
| `tabryx_last_error` | Read the latest detailed error message |
| `tabryx_file_version` | Inspect the file-format version without a password |

Result strings are UTF-8 and owned by the result handle. Release each query result once with `tabryx_result_free()`; do not free individual strings.

## Tabryx Manager

`TabryxManager` provides a graphical interface for Windows with:

- database creation, opening, saving, and password changes;
- table creation and deletion;
- row add, edit, delete, copy, and physical reordering;
- column add, delete, copy, and physical reordering;
- paginated data browsing and structure inspection;
- multi-statement SQL execution;
- complete-table CSV and JSON export.

## Security Notes

- The database contents are encrypted at rest; the file header contains non-secret format and KDF metadata.
- A fresh salt and nonce are generated whenever the database is saved.
- The full plaintext database exists in process memory while open.
- Password strength remains important even with Argon2id.
- TabryxDb does not automatically save on close; applications should handle save failures explicitly.
- Back up important data. This project is currently version `1.0.0` and should be evaluated carefully before production use.

## Project Status

TabryxDb is an early-stage project. Its API and file format may evolve before a stable release. Review the release notes before upgrading applications or important databases. Issue reports are welcome.

## License

TabryxDb binaries and accompanying documentation are released under the [MIT License](LICENSE). The MIT License permits binary-only distribution and does not require public source-code publication. Include the license text when redistributing TabryxDb.
