<h1 align="center">
  <img src="../docs/logo/pg_bm25.svg" alt="pg_bm25" width="500px"></a>
<br>
</h1>

[![Test pg_bm25](https://github.com/paradedb/paradedb/actions/workflows/test-pg_bm25.yml/badge.svg)](https://github.com/paradedb/paradedb/actions/workflows/test-pg_bm25.yml)

## Overview

`pg_bm25` is a PostgreSQL extension that enables full text search over SQL tables using the BM25 algorithm, the state-of-the-art ranking function for full text search. It is built on top of Tantivy, the Rust-based alternative to Apache Lucene, using `pgrx`.

`pg_bm25` is supported on all versions supported by the PostgreSQL Global Development Group, which includes PostgreSQL 12+.

Check out the `pg_bm25` benchmarks [here](../benchmarks/README.md).

### Roadmap

- [x] BM25 scoring
- [x] Highlighting
- [x] Filtering
- [x] Autocomplete
- [x] Boosted queries
- [x] Fuzzy search
- [x] Custom tokenizers (Chinese, Japanese, Korean, Arabc, Ngram)
- [x] JSON field search
- [x] Hybrid search
- [ ] Datetime fields
- [ ] Faceted search
- [ ] Generative search
- [ ] Multimodal search
- [ ] Distributed search

## Installation

### From ParadeDB

The easiest way to use the extension is to run the ParadeDB Dockerfile:

```bash
docker run \
  -e POSTGRES_USER=<user> \
  -e POSTGRES_PASSWORD=<password> \
  -e POSTGRES_DB=<dbname> \
  -p 5432:5432 \
  -d \
  paradedb/paradedb:latest
```

This will spin up a Postgres instance with `pg_bm25` preinstalled.

### From Self-Hosted PostgreSQL

If you are self-hosting Postgres and would like to use the extension within your existing Postgres, follow these steps:

#### Debian/Ubuntu

We provide pre-built binaries for Debian-based Linux for PostgreSQL 15 (more versions coming soon). You can download the latest version for your architecture from the [releases page](https://github.com/paradedb/paradedb/releases).

ParadeDB collects anonymous telemetry to help us understand how many people are using the project. You can opt-out of telemetry by setting `export TELEMETRY=false` (or unsetting the variable) in your shell or in your `~/.bashrc` file before running the extension.

#### macOS

We don't suggest running production workloads on macOS. As a result, we don't provide prebuilt binaries for macOS. If you are running Postgres on macOS and want to install `pg_bm25`, please follow the [development](#development) instructions, but do `cargo pgrx install --release` instead of `cargo pgrx run`. This will build the extension from source and install it in your Postgres instance.

You can then create the extension in your database by running:

```sql
CREATE EXTENSION pg_bm25;
```

Note: If you are using a managed Postgres service like Amazon RDS, you will not be able to install `pg_bm25` until the Postgres service explicitly supports it.

#### Windows

Windows is not supported. This restriction is [inherited from pgrx not supporting Windows](https://github.com/pgcentralfoundation/pgrx?tab=readme-ov-file#caveats--known-issues).

## Usage

### Indexing

`pg_bm25` comes with a helper function that creates a test table that you can use for quick experimentation.

```sql
CALL paradedb.create_bm25_test_table(
  schema_name => 'public',
  table_name => 'mock_items'
);
```

To index the table, use the following SQL command:

```sql
CALL paradedb.create_bm25(
        index_name => 'search_idx',
        schema_name => 'public',
        table_name => 'mock_items',
        key_field => 'id',
        text_fields => '{description: {tokenizer: {type: "en_stem"}}, category: {}}'
);
```

Note the mandatory `key_field` option in the `WITH` code. Every `bm25` index needs a `key_field`, which should be the name of a column that will function as a row's unique identifier within the index. Usually, the `key_field` can just be the name of your table's primary key column.

Once the indexing is complete, you can run various search functions on it.

### Basic Search

Execute a search query on your indexed table:

```sql
SELECT description, rating, category
FROM search_idx.search('description:keyboard OR category:electronics')
LIMIT 5;
```

This will return:

```csv
         description         | rating |  category
-----------------------------+--------+-------------
 Plastic Keyboard            |      4 | Electronics
 Ergonomic metal keyboard    |      4 | Electronics
 Innovative wireless earbuds |      5 | Electronics
 Fast charging power bank    |      4 | Electronics
 Bluetooth-enabled speaker   |      3 | Electronics
(5 rows)
```

Advanced features like BM25 scoring, highlighting, custom tokenizers, fuzzy search, and more are supported. Please refer to the [documentation](http://localhost:3000/latest/search/bm25) and [quickstart](http://localhost:3000/latest/quickstart) for a more thorough overview of `pg_bm25`'s query support.

## Development

### Prerequisites

Before developing the extension, ensure that you have Rust v1.73.0 installed, ideally via `rustup` (we've observed issues with installing Rust via Homebrew on macOS).

If you are on macOS and using Postgres.app, you'll first need to add the `pg_config` binary to your path:

```bash
export PATH="$PATH:/Applications/Postgres.app/Contents/Versions/latest/bin"
```

Then, install and initialize pgrx:

```bash
# Note: Replace --pg15 with your version of Postgres, if different (i.e. --pg16, --pg14, etc.)
cargo install --locked cargo-pgrx --version 0.11.1
cargo pgrx init --pg15=`which pg_config`
```

### Running the Extension

First, start pgrx:

```bash
cargo pgrx run
```

This will launch an interactive connection to Postgres. Inside Postgres, create the extension by running:

```sql
CREATE EXTENSION pg_bm25;
```

Now, you have access to all the extension functions.

### Modifying the Extension

If you make changes to the extension code, follow these steps to update it:

1. Recompile the extension:

```bash
cargo pgrx run
```

2. Recreate the extension to load the latest changes:

```sql
DROP EXTENSION pg_bm25;
CREATE EXTENSION pg_bm25;
```

### Testing

To run the unit test suite, use the following command:

```bash
cargo pgrx test
```

This will run all unit tests defined in `/src`. To add a new unit test, simply add tests inline in the relevant files, using the `#[cfg(test)]` attribute.

To run the integration test suite, simply run:

```bash
./test/runtests.sh -p sequential
```

This will create a temporary database, initialize it with the SQL commands defined in `fixtures.sql`, and run the tests in `/test/sql` against it. To add a new test, simply add a new `.sql` file to `/test/sql` and a corresponding `.out` file to `/test/expected` for the expected output, and it will automatically get picked up by the test suite.

Note: the bash script takes arguments and allows you to run tests either sequentially or in parallel. For more info run `./test/runtests.sh -h`

## License

`pg_bm25` is licensed under the [GNU Affero General Public License v3.0](../LICENSE).
