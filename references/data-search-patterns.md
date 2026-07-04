# Data And Search Patterns

Use this reference for model generation, repository ownership, ORM choices, database inspection, Mongo support, and search scaffolds.

## Model Generation

Generate models from SQL DDL:

```bash
rozectl model generate example/user.sql --out services/user-api --format sql
```

Generate models from the Roze model DSL:

```bash
rozectl model generate example/user.model --out services/user-api
```

Inspect existing SQL tables:

```bash
rozectl model inspect users --db-kind postgres --db-url postgres://postgres:postgres@127.0.0.1:5432/roze --schema public --out services/user-api
rozectl model inspect users --db-kind mysql --db-url mysql://root:root@127.0.0.1:3306/roze --out services/user-api
```

Inspect Mongo collections:

```bash
rozectl model inspect users --db-kind mongo --db-url mongodb://127.0.0.1:27017/roze --sample-size 100 --out services/user-api
```

Use `--schema` to disambiguate shared table names. For Mongo, `--schema` is the database name when it is not included in `--db-url`.

## ORM Choices

Toasty is the default SQL model scaffold:

```bash
rozectl model generate example/user.sql --out services/user-api --format sql
```

Generate SeaORM-style modules instead:

```bash
rozectl model generate example/user.sql --out services/user-api --format sql --orm sea-orm
```

Generated REST/API services do not depend on database crates by default. Add models only when the application needs persistence.

## Repository Surface

Generated Toasty repositories should support the stable Roze model contract where applicable:

- primary-key and cache-key lookup
- list, insert, update, delete, count
- paginated query
- equality, IN, range, nullable, and `IS NULL` filters
- typed sorting
- batch insert/delete
- soft-delete helpers
- tenant-scoped lookup when inferred or configured
- transaction-compatible executor arguments

Generated SeaORM output should expose the same practical repository surface where possible.

Handwritten extensions belong in application-owned files such as `src/model/_ext.rs`. Do not place business-specific query orchestration in generator-owned files.

## SQL And Mongo Input Notes

Supported SQL DDL focuses on common MySQL and Postgres `CREATE TABLE` forms with a single primary key, common scalar column types, default values, comments, and auto-increment forms. Unsupported features such as composite keys and foreign keys should fail fast with clear errors.

Mongo inspection samples collection documents for field and type inference, maps `_id` to `id`, preserves unique/index metadata, emits helpers for unique and compound indexes, and can generate an `ObjectId` id model for empty collections.

## Search Generation

Generate search repositories:

```bash
rozectl search generate example/user.search --engine elasticsearch --out services/user-api
```

Inspect existing search indexes:

```bash
rozectl search inspect users --engine opensearch --url http://127.0.0.1:9200 --out services/user-api
rozectl search inspect users --engine meilisearch --url http://127.0.0.1:7700 --sample-size 100 --out services/user-api
```

Search generation writes:

```text
src/search/mod.rs
src/search/*.rs
```

Search generated code should use `roze-search` for health checks, document indexing, document deletion, and text search. Generated structs should preserve original index field names with `serde(rename = "...")`.

Keep database models under `src/model` and search indexes under `src/search`; do not merge the generated ownership boundaries.

Handwritten ranking, boosting, filter composition, recall strategy, result reranking, and business search orchestration belong in application modules that call generated repositories.

## Verification

For generator-only model/search work, start with:

```bash
cargo test -p rozectl -- --skip postgres --skip mysql --skip mongo
```

Run database-specific tests only when credentials or local integration services are available:

```bash
cargo test -p rozectl postgres
cargo test -p rozectl mysql
```

For `roze-search` runtime changes, run:

```bash
cargo check -p roze-search
cargo test -p rozectl
```
