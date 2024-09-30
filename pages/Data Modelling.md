- `schema.prisma`
- To see what sql would have to be executed to make a new database from your existing database: `sqlite3 prisma/data.db .dump > data.sql`
  collapsed:: true
	- ```sql
	  PRAGMA foreign_keys=OFF;
	  BEGIN TRANSACTION;
	  CREATE TABLE IF NOT EXISTS "User" (
	      "id" TEXT NOT NULL PRIMARY KEY,
	      "email" TEXT NOT NULL,
	      "username" TEXT NOT NULL,
	      "name" TEXT,
	      "createdAt" DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
	      "updatedAt" DATETIME NOT NULL
	  );
	  INSERT INTO User VALUES('clklmcxa50000x7drb1zc5bbu','kody@kcd.dev','kody','Kody',1690490436990,1690490427769);
	  CREATE UNIQUE INDEX "User_email_key" ON "User"("email");
	  CREATE UNIQUE INDEX "User_username_key" ON "User"("username");
	  COMMIT;
	  ```
- ## Relationships
  collapsed:: true
	- ### One to One
	  collapsed:: true
		- Need: to keep entities separate: User, Aadhar. Can't put Aadhar into User as one entity otherwise there might be
			- Data leaks
			- Unwanted fields when that one Entity say `User'` is requested.
			- Might not make sense for say a baby to have an Aadhar card just yet.
	- ### One to Many
	  collapsed:: true
		- `Person` -> Many -> `PhoneNumber`
		- `PhoneNumber` -> Has one ( belongs to one) -> `Person`
	- ### Many to Many
	  collapsed:: true
		- `Post` <-(each can have many of each other)-> `Tags`
		- You cannot represent a many-to-many relationship between two database tables.
			- Have to create a third **junction/bridge/associative** table
			- ```sql
			  -- CreateTable
			  CREATE TABLE "Post" (
			      "id" TEXT NOT NULL PRIMARY KEY,
			      "title" TEXT NOT NULL
			  );
			  
			  -- CreateTable
			  CREATE TABLE "Tag" (
			      "id" TEXT NOT NULL PRIMARY KEY,
			      "name" TEXT NOT NULL
			  );
			  
			  -- CreateTable
			  CREATE TABLE "_PostToTag" (
			      "A" TEXT NOT NULL,
			      "B" TEXT NOT NULL,
			      CONSTRAINT "_PostToTag_A_fkey" FOREIGN KEY ("A") REFERENCES "Post" ("id") ON DELETE CASCADE ON UPDATE CASCADE,
			      CONSTRAINT "_PostToTag_B_fkey" FOREIGN KEY ("B") REFERENCES "Tag" ("id") ON DELETE CASCADE ON UPDATE CASCADE
			  );
			  
			  -- CreateIndex
			  CREATE UNIQUE INDEX "_PostToTag_AB_unique" ON "_PostToTag"("A", "B");
			  
			  -- CreateIndex
			  CREATE INDEX "_PostToTag_B_index" ON "_PostToTag"("B");
			  
			  ```
			- **Explanation**
				- Each row in this table represents a single association between a Post and a Tag.
				- If a Post has multiple Tags, it will have multiple rows in this table, one for each Tag.
				- If a Tag is used on multiple Posts, it will appear in multiple rows, one for each Post.
			- **Example**:
				- A Post with id "post1"
				- Tags with ids "tag1", "tag2", "tag3"
				  
				  If "post1" has all these tags, the _PostToTag table would have these rows:
				  
				  | A | B |
				  | post1 | tag1 |
				  | post1 | tag2 |
				  | post1 | tag3 |
			- **Benefits**:
				- Flexibility: You can add or remove associations without changing the structure of the Post or Tag tables.
				- Querying: It's easy to find all Tags for a Post or all Posts with a specific Tag.
			- **Constraints:**
				- The foreign key constraints ensure data integrity. You can't add a Post-Tag association if either the Post or Tag doesn't exist.
				- The unique index on (A, B) prevents duplicate associations.
	- Entity Relationship Diagrams ([for prisma](https://github.com/keonik/prisma-erd-generator))
	- ## One to Many Relationships
		- `User` ->(many)-> `Note`. User doesn't know anything about the note. Note is the one which knows which user I am a note of. So it's really `User` <- `Note`
	- > polymorphism and databases don't mix well. Don't try to make generic models which one could extend from like a class
- ## Migrations
  collapsed:: true
	- Prisma create SQL migration files – which one can change / edit if need be, which need to be committed to the repository
	- Prisma will also keep track of which migrations have been applied to your database so you don't have to worry about that. (It creates a very small table in your database `_prisma_migrations` for this)
	- `npx prisma migrate deploy`
	- Example of a migration SQL file
	  ```sql
	  -- CreateEnum
	  CREATE TYPE "Role" AS ENUM ('ADMIN', 'MEMBER');
	  
	  -- AlterTable
	  ALTER TABLE "User" ADD COLUMN     "role" "Role" DEFAULT E'MEMBER';
	  
	  -- Manually written stuff:
	  
	  -- Update all users to be members:
	  update "User" set role = E'MEMBER';
	  
	  -- update me@kentcdodds.com to be ADMIN:
	  update "User" set role = E'ADMIN' where email = 'me@kentcdodds.com';
	  
	  -- make role required
	  ALTER TABLE "User" ALTER COLUMN "role" SET NOT NULL;
	  ```
	- ### 0 Downtimes
		- **WHY**
			- Don't want to shut app while migrations
			- Don't want the users' app to fault while migrations are running
		- **HOW**
			- Widen then narrow -> Idea is to make database schema more permissive
			- https://github.com/epicweb-dev/epic-stack/blob/main/docs/database.md#migrations
				- Widen app to consume A or B
				- Widen db to provide A and B and the app to write to both A and B
				- Narrow app to consume B and only write to B
				- Narrow db to provide B
			- **EXAMPLE**
				- So, let's say that today your app allows users to provide a "name" and you want to change that to `firstName` and `lastName` instead. Here's how you'd do that (again, each of these steps end in a deploy):
				- Widen app to consume `firstName` and `lastName` or `name`. So all new code that references the `firstName` and `lastName` fields should fallback to the `name` field and not error if the `firstName` and `lastName` fields don't exist yet, which it won't at this point.
				- Widen db to provide `firstName` and `lastName` and `name`. So the `name` field should be populated with the `firstName` and `lastName` fields. You can do this as part of the migration SQL script that you run. The easiest way to do this is to generate the migration script to add the fields using `prisma migrate` and then modify the script to copy the existing data in the `name` field to the `firstName` field (maybe with the help of VSCode Copilot 😅).
				- Narrow app to consume `firstName` and `lastName` by only writing to those fields and removing the fallback to the `name` field.
				- Narrow db to provide `firstName` and `lastName` by removing the `name` field. So now you can remove the `name` field from the db schema.
	- Links to read
		- TODO [Prisma: Switch Data Providers](https://www.prisma.io/docs/orm/prisma-migrate/understanding-prisma-migrate/limitations-and-known-issues#you-cannot-automatically-switch-database-providers)
		- TODO [kentcdodds: I Migrated from a Postgres Cluster to Distributed SQLite with LiteFS](https://kentcdodds.com/blog/i-migrated-from-a-postgres-cluster-to-distributed-sqlite-with-litefs)
		- TODO [fly.io: I'm All-In on Server-Side SQLite](https://fly.io/blog/all-in-on-sqlite-litestream/)
		- DONE [Prisma: Getting started with prisma migrate](https://www.prisma.io/docs/orm/prisma-migrate/getting-started)
		- TODO [Prisma: About migration histories](https://www.prisma.io/docs/orm/prisma-migrate/understanding-prisma-migrate/migration-histories)
		-
- ## Seeding
  collapsed:: true
	- ### Production Seeding
		- Backup
		  logseq.order-list-type:: number
		- Create a new database file (locally)
		  logseq.order-list-type:: number
		- Dump
		  logseq.order-list-type:: number
			- ```sh
			  sqlite3 seed.db .dump > seed.sql
			  ```
		- Copy `seed.sql` to prod. Change this as required in case of existing database, for example for indexes
		  logseq.order-list-type:: number
		- Run `seed.sql` to prod
		  logseq.order-list-type:: number
			- ```sh
			  sqlite3 data.db < seed.sql
			  ```
		- Verify. If something wrong, restore to backup
		  logseq.order-list-type:: number
	- ### Prisma built in support for seed script
	  logseq.order-list-type:: number
		- Runs the seed after `npx prisma migrate reset` or manually `npx prisma db seed`
		  logseq.order-list-type:: number
	- ## Links to read
	  logseq.order-list-type:: number
		- DONE [Prima: Seeding](https://www.prisma.io/docs/orm/prisma-migrate/workflows/seeding)
		  logseq.order-list-type:: number
		- DONE [Nested writes](https://www.prisma.io/docs/orm/prisma-client/queries/transactions#nested-writes)
		  logseq.order-list-type:: number
		- DONE [Nested queries API reference](https://www.prisma.io/docs/orm/reference/prisma-client-reference#create-1)
		  logseq.order-list-type:: number
- ## Generating Seed Data
  collapsed:: true
	- faker.js things.
	  logseq.order-list-type:: number
	- https://github.com/supabase-community/seed : try this in your project
	  logseq.order-list-type:: number
- ## Querying Data
  collapsed:: true
	- Do the following for dev server, so that server doesn't need a restart every refresh/hot reload
		- ```typescript
		  import { PrismaClient } from '@prisma/client'
		  
		  if (!global.prisma) {
		  	global.prisma = new PrismaClient()
		  }
		  
		  export const prisma = global.prisma
		  ```
	- #sql #sqlite #link https://www.sqlite.org/lang_keywords.html
	- ### Usage
		- **SQL**
		  
		  ```sql
		  SELECT id, name, email FROM user WHERE id = 1;
		  ```
		  
		  **Prisma**
		  ```typescript
		  const user1 = await prisma.user.findUnique({
		  	where: { id: 1 },
		  	select: { id: true, name: true, email: true },
		  })
		  ```
	- ### Links
		- TODO [Prisma: Tracing](https://www.prisma.io/docs/orm/prisma-client/observability-and-logging/opentelemetry-tracing)
		- TODO [Prisma: Metrics](https://www.prisma.io/docs/orm/prisma-client/observability-and-logging/metrics)
- ## Updating Data
	-