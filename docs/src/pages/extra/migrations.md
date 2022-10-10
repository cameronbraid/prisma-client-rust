---
title: Migrations
layout: ../../layouts/MainLayout.astro
---

Enabling the `migrations` feature for `prisma-client-rust` and `prisma-client-rust-cli`
will cause the generated client to expose some methods for using Prisma's
[migration engine](https://www.prisma.io/docs/concepts/components/prisma-migrate).
Specifically, the Prisma CLI's `db push`, `migrate deploy` and `migrate resolve` functions will have equivalent functions in the client.

Using the migration engine in this way is not recommended unless you can't use the Prisma CLI,
such as for desktop apps (like those built with [Tauri](https://tauri.app/))
and server deployments where the CLI is not available.
When the CLI is available, use it manually or in CI to deploy migrations.

## In Development

While modifying your Prisma schema use `PrismaClient::_db_push` to synchronize your database and schema.
As explained in [Prisma's docs](https://www.prisma.io/docs/reference/api-reference/command-reference#db-push), this does not use migrations,
instead it pushes the state of your schema directly to the database.

`_db_push` returns a builder than has additional functions specifying additional options that are available in the CLI:

```rust
client
  ._db_push()
  .accept_data_loss() // --accept-data-loss in CLI
  .force_reset()      // --force-reset in CLI
  .await?;
```

## In Production

After you have finalised your schema changes and generated migrations via the CLI,
use `PrismaClient::_migrate_deploy` to  apply all pending migrations with the migration engine 
([Prisma docs](https://www.prisma.io/docs/reference/api-reference/command-reference#migrate-deploy)).

## Baselining

Prisma provides the ability to baseline existing database in order to make them compatible with Prisma migrate.
The baselining process will not be detailed here as 
[the Prisma docs](https://www.prisma.io/docs/guides/database/developing-with-prisma-migrate/baselining)
already do a great job explaining it.

`PrismaClient::_migrate_resolve` is available as an in-code alternative to the CLI's `migrate resolve` command. It requires specifying a migration name as its first argument.

```rust
client
  ._migrate_resolve("20210426141759_initial-migration-for-db")
  .await?;
```


## Examples

### New Project

The following migration setup may be used in a new application using Prisma Client Rust.
Baselining is not necessary as the database will not have existing migrations.

During development, `_db_push` will be used to synchronize the schema and database without generating migrations.
It may be necessary to occasionally add `accept_data_loss` or `force_reset` if major changes are made to your schema.
It is necessary to generate migrations using the CLI's `migrate dev` command once schema changes are finalized

When the application is ran in release mode,
`_migrate_deploy` will be used to apply all migrations generated by `migrate dev`.

```rust
#[cfg(debug_assertions)]
client._db_push().await?;
#[cfg(not(debug_assertions))]
client._migrate_deploy().await?;
```


### Existing Project

For a project switching from using another ORM or raw SQL a similar approach to new projects can be taken,
but [baselining](https://www.prisma.io/docs/guides/database/developing-with-prisma-migrate/baselining) must be done.

After introspecting your database schema into a `schema.prisma` file,
generate the initial migration with `migrate dev`,
then use that migration's name in `_migrate_resolve` before applying further migrations.

```rust
client
  ._migrate_resolve("INITIAL MIGRATION NAME")
  .await?;

#[cfg(debug_assertions)]
client._db_push().await?;
#[cfg(not(debug_assertions))]
client._migrate_deploy().await?;
```