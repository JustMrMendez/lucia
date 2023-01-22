---
_order: 0
title: "Kysely"
---

An adapter for [Kysely SQL query builder](https://github.com/koskimas/kysely). This adapter currently supports [all 3 dialects officially supported by Kysely](https://github.com/koskimas/kysely#installation):

- PostgreSQL via [`pg`](https://github.com/brianc/node-postgres)
- MySQL via [`mysql2`](https://github.com/sidorares/node-mysql2)
- SQLite via [`better-sqlite3`](https://github.com/WiseLibs/better-sqlite3)

```ts
const adapter: (
	db: Kysely<Database>,
	dialect: "pg" | "mysql" | "better-sqlite3"
) => AdapterFunction<Adapter>;
```

### Parameter

See [Database interfaces](#database-interfaces) for more information regarding the `Database` type used for the Kysely instance.

| name    | type                                  | description     |
| ------- | ------------------------------------- | --------------- |
| db      | `Kysely<Database>`                    | Kysely instance |
| dialect | `"pg" \| "mysql" \| "better-sqlite3"` | dialect used    |

### Errors

The adapter and Lucia will not not handle [unknown errors](/learn/basics/error-handling#known-errors), database errors Lucia doesn't expect the adapter to catch. When it encounters such errors, it will throw one of

- `DatabaseError` for `pg`.
- `QueryError` for `mysql2`
- `SqliteError` for `better-sql3`

## Installation

```bash
npm i @lucia-auth/adapter-kysely
pnpm add @lucia-auth/adapter-kysely
yarn add @lucia-auth/adapter-kysely
```

## Usage

```ts
import {
	default as kysely,
	type KyselyLuciaDatabase
} from "@lucia-auth/adapter-kysely";
import { Kysely } from "kysely";

const db = new Kysely<KyselyLuciaDatabase>(options);

const auth = lucia({
	adapter: kysely(db, "pg") // change "pg" to "mysql2", "better-sqlite3"
});
```

### Database interfaces

Define interfaces for each table in your database or generate them automatically using [kysely-codegen](https://github.com/RobinBlomberg/kysely-codegen). Then, pass those interfaces to the `Kysely` constructor. See the [Kysely github repo](https://github.com/koskimas/kysely#minimal-example) for an example on creating the `Kysely` instance.

The `Database` interface must be of the following structure.

```ts
import { ColumnType, Generated } from "kysely";

type BigIntColumnType = ColumnType<number | bigint>;

interface Session {
	expires: BigIntColumnType;
	id: string;
	idle_expires: BigIntColumnType;
	user_id: string;
}

interface User {
	id: Generated<string>;
	hashed_password: string | null;
	provider_id: string;
	// Plus other columns that you defined
}

interface Database {
	session: Session;
	user: User;
	// Plus other table interfaces
}
```

You can also import the interfaces from `adapter-kysely` and extend them.

```ts
import type {
	KyselyLuciaDatabase,
	KyselyUser
} from "@lucia-auth/adapter-kysely";

// Add a column for username in your user table
interface User extends KyselyUser {
	username: string;
}

interface Database extends Omit<KyselyLuciaDatabase, "user"> {
	user: User;
	other_table: {
		//...
	};
}

const db = new Kysely<Database>(options);

const auth = lucia({
	adapter: kysely(db, dialect)
});
```

## Database structure

### PostgreSQL

#### `user`

`id` may be `text` if you generate your own user id. You may add additional columns to store user attributes. Refer to [Store user attributes](/learn/basics/store-user-attributes).

| name            | type   | foreign constraint | default             | nullable | unique | primary |
| --------------- | ------ | ------------------ | ------------------- | -------- | ------ | ------- |
| id              | `UUID` |                    | `gen_random_uuid()` |          | true   | true    |
| provider_id     | `TEXT` |                    |                     |          | true   |         |
| hashed_password | `TEXT` |                    |                     | true     |        |         |

```sql
CREATE TABLE public.user (
	id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
	provider_id TEXT NOT NULL UNIQUE,
	hashed_password TEXT
);
```

#### `session`

| name         | type     | foreign constraint | nullable | unique | primary |
| ------------ | -------- | ------------------ | -------- | ------ | ------- |
| id           | `TEXT`   |                    |          | true   | true    |
| user_id      | `UUID`   | `public.user(id)`  |          |        |         |
| expires      | `BIGINT` |                    |          |        |         |
| idle_expires | `BIGINT` |                    |          |        |         |

```sql
CREATE TABLE public.session (
  	id TEXT PRIMARY KEY,
	user_id UUID REFERENCES public.user(id) NOT NULL,
	expires BIGINT NOT NULL,
	idle_expires BIGINT NOT NULL
);
```

### MySQL

#### `user`

The length of the `VARCHAR` type of `id` should be of appropriate length if you generate your own user ids. You may add additional columns to store user attributes. Refer to [Store user attributes](/learn/basics/store-user-attributes).

| name            | type           | nullable | unique | primary |
| --------------- | -------------- | -------- | ------ | ------- |
| id              | `VARCHAR(36)`  |          | true   | true    |
| provider_id     | `VARCHAR(255)` |          | true   |         |
| hashed_password | `VARCHAR(255)` | true     |        |         |

```sql
CREATE TABLE user (
    id VARCHAR(36) NOT NULL,
    provider_id VARCHAR(255) NOT NULL UNIQUE,
    hashed_password VARCHAR(255),
    PRIMARY KEY (id)
);
```

#### `session`

| name         | type                | foreign constraint | nullable | unique | identity |
| ------------ | ------------------- | ------------------ | -------- | ------ | -------- |
| id           | `VARCHAR(127)`      |                    |          | true   | true     |
| user_id      | `VARCHAR(36)`       | `user(id)`         |          |        |          |
| expires      | `BIGINT` (UNSIGNED) |                    |          |        |          |
| idle_expires | `BIGINT` (UNSIGNED) |                    |          |        |          |

```sql
CREATE TABLE session (
    id VARCHAR(127) NOT NULL,
    user_id VARCHAR(36) NOT NULL,
    expires BIGINT UNSIGNED NOT NULL,
    idle_expires BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY (user_id) REFERENCES user(id)
);
```

### SQLite

#### `user`

The length of the `VARCHAR` type of `id` should be of appropriate length if you generate your own user ids. You may add additional columns to store user attributes. Refer to [Store user attributes](/learn/basics/store-user-attributes).

| name            | type           | nullable | unique | identity |
| --------------- | -------------- | -------- | ------ | -------- |
| id              | `VARCHAR(36)`  |          | true   | true     |
| provider_id     | `VARCHAR(255)` |          | true   |          |
| hashed_password | `VARCHAR(255)` | true     |        |          |

```sql
CREATE TABLE user (
    id VARCHAR(31) NOT NULL,
    provider_id VARCHAR(255) NOT NULL UNIQUE,
    hashed_password VARCHAR(255),
    PRIMARY KEY (id)
);
```

#### `session`

Type for `user_id` should match the type of `user(id)`.

| name         | type           | foreign constraint | nullable | unique | identity |
| ------------ | -------------- | ------------------ | -------- | ------ | -------- |
| id           | `VARCHAR(127)` |                    |          | true   | true     |
| user_id      | `VARCHAR(36)`  | `user(id)`         |          |        |          |
| expires      | `BIGINT`       |                    |          |        |          |
| idle_expires | `BIGINT`       |                    |          |        |          |

```sql
CREATE TABLE session (
    id VARCHAR(127) NOT NULL,
    user_id VARCHAR(36) NOT NULL,
    expires BIGINT NOT NULL,
    idle_expires BIGINT NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY (user_id) REFERENCES user(id)
);
```