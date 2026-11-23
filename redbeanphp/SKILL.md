---
name: redbeanphp
description: Write, review, and debug PHP code that uses RedBeanPHP (the zero-config ORM, `R::dispense`, `R::store`, `R::find`, beans). Use this skill whenever the user mentions RedBeanPHP, RedBean, `rb.php`, beans, `R::` calls, `ownBookList` / `sharedTagList` relations, fluid vs frozen mode, FUSE models, or asks for lightweight PHP database code with an automatic schema. Also use it when migrating a RedBeanPHP prototype to production, hardening it, or explaining unexpected RedBeanPHP errors, even if the user does not name the library explicitly but the code is clearly using `R::`.
---

# RedBeanPHP

RedBeanPHP 5 (current: 5.7.x, LTS, PHP 5.2 through 8.5) is a single-file ORM built around *beans*: plain objects that map to table rows. In **fluid mode** it creates tables and columns, and widens column types, as you store beans. In **frozen mode** it never alters the schema. Most RedBeanPHP bugs come from not being clear about which mode the code runs in, so settle that first.

Supported databases: MySQL/MariaDB, PostgreSQL, SQLite, CUBRID, Firebird.

## Orientation checklist

Before writing or reviewing code, establish:

1. **How is it installed?** Single file (`require 'rb.php'`, recommended upstream, global `R` class) or Composer (`gabordemooij/redbean`, class is `\RedBeanPHP\R`, so add `use RedBeanPHP\R;`).
2. **Fluid or frozen?** Look for `R::freeze(true)`. If absent, the app is fluid and schema changes happen at runtime.
3. **Feature set?** `R::useFeatureSet('novice/latest')` is recommended for new code. `novice` disables dangerous calls such as `R::nuke()`.
4. **PHP/DB versions**, only if behaviour is surprising.

## Setup

```php
require 'rb.php';

// MySQL / MariaDB (be explicit about charset)
R::setup('mysql:host=localhost;dbname=app;charset=utf8mb4', 'user', 'pass');
// PostgreSQL
// R::setup('pgsql:host=localhost;dbname=app', 'user', 'pass');
// SQLite file
// R::setup('sqlite:/path/to/app.db');
// R::setup();  // no args = throwaway SQLite database, handy for tests

R::useFeatureSet('novice/latest');  // call right after setup

// Production: stop schema changes (see references/production.md)
// R::freeze(true);
```

`R::setup()` is lazy: it does not connect until the first query. Use `R::testConnection()` to check up front.

## Naming rules (RedBeanPHP 5 enforces these)

- **Bean types** (= table names): lowercase letters and digits only. `book`, `bookpage`. `bookPage` and `book_page` throw `Invalid bean type`.
- **Properties** (= columns): use `snake_case`, e.g. `$book->page_count`. camelCase works but is converted to snake_case in the column (`someProp` becomes `some_prop`), which causes confusion; stay snake_case.
- **Relation properties** are special (`ownPageList`, `sharedTagList`, `xownPageList`); don't use these names for ordinary columns.
- Link tables for many-to-many are named after both types, alphabetically: `book_tag`.

## CRUD

```php
// Create
$book = R::dispense('book');
$book->title = 'Secrets of Christmas';
$book->page_count = 120;
$id = R::store($book);          // returns the id

// Read
$book = R::load('book', $id);   // a non-existent id gives an EMPTY bean with id 0, not null
if (!$book->id) { /* not found */ }

// Update
$book->title = 'New title';
R::store($book);

// Delete
R::trash($book);
R::trashAll($beans);
```

Always check `$bean->id` after `R::load()`. It never returns null and never throws for a missing row.

## Finding beans

```php
$books = R::find('book', 'title = ?', ['Secrets of Christmas']);   // array keyed by id
$one   = R::findOne('book', 'id = ?', [$id]);                      // bean or NULL
$all   = R::findAll('book', ' ORDER BY title LIMIT 10 ');          // no WHERE: pass tail SQL
$n     = R::count('book', 'page_count > ?', [100]);
$book  = R::findOrCreate('book', ['title' => 'X']);
$likes = R::findLike('book', ['title' => ['A', 'B']]);             // match any of these values
$ids   = [1, 2, 3];
$in    = R::find('book', ' id IN (' . R::genSlots($ids) . ') ', $ids);
```

The `find*` family takes only the SQL *after* `WHERE` (or the tail clause for `findAll`). Use named slots (`:name` with `[':name' => $v]`) or `?`, but not both in one query.

Raw SQL when beans are not needed (these return plain arrays):

```php
$rows = R::getAll('SELECT id, title FROM book WHERE id < ?', [10]);
$row  = R::getRow('SELECT * FROM book WHERE id = ?', [1]);
$col  = R::getCol('SELECT title FROM book');
$cell = R::getCell('SELECT COUNT(*) FROM book');
$map  = R::getAssoc('SELECT id, title FROM book');   // [id => title]
R::exec('UPDATE book SET page_count = ? WHERE id = ?', [200, 1]);
$beans = R::convertToBeans('book', $rows);            // raw rows back into beans
```

Values returned from the database come back as **strings** (`"1"`, not `1`), so cast when strict types matter.

For large result sets use `R::findCollection('book', ...)` and loop with `$collection->next()` to avoid loading everything into memory.

## Relations (summary)

```php
// One-to-many: a book owns pages
$book->ownPageList[] = $page;  R::store($book);
$page->book;                   // the parent (many-to-one), via page.book_id

// Many-to-many: link table book_tag is created automatically
$book->sharedTagList[] = $tag; R::store($book);

// Filter / order / count a relation
$book->with(' ORDER BY num DESC LIMIT 5 ')->ownPageList;
$book->withCondition(' num > ? ', [10])->ownPageList;
$book->countOwn('page');

// Cascade delete children when the parent is trashed: xown
$shelf->xownBookList[] = $book;
```

Aliases (`fetchAs`, `alias`), extended many-to-many (`via`), and tagging are in `references/relations.md`. Read it when the schema is anything beyond a simple parent/child.

## Gotchas that bite in practice

1. **`R::transaction()` does nothing in fluid mode.** It returns `FALSE` without running your callback unless the database is frozen or you call `R::setAllowFluidTransactions(true)`. In fluid mode use `R::begin()`, `R::commit()`, `R::rollback()` in a try/catch. When it does run, it commits on success, rolls back and rethrows on exception, and returns the callback's return value.
2. **`R::nuke()` is disabled** by `novice/*` feature sets (it throws). That is intentional; it drops every table.
3. **Dynamic property names from user input create columns** in fluid mode. Never do `foreach ($_POST as $k => $v) $bean->$k = $v;`. Use `$bean->import($_POST, 'title,author')` with a whitelist and freeze in production.
4. **Type widening is one-way.** Storing `'hello'` in a column that held integers silently converts it to TEXT. Mixed data in a column makes it wider, never narrower. Freeze and define columns yourself once the schema stabilises.
5. **Missing `ownList` children are not auto-deleted** on `R::trash()`. Use `xown` for cascade, or let a real foreign key with `ON DELETE CASCADE` do it.
6. **Never concatenate user input into SQL.** Always bind with `?` or `:name`. Bean property reads and writes are safe; the `find*`, `get*`, `exec` SQL strings are where injection happens.
7. **Namespaced/Composer installs:** models must extend `\RedBeanPHP\SimpleModel`, and `R` is `\RedBeanPHP\R`.
8. **Unique constraints:** don't rely on bean meta for them. Add them in a SQL migration (`CREATE UNIQUE INDEX ...`) and catch `\RedBeanPHP\RedException\SQL` on store. This was not reliable when tested on a fresh SQLite table.

## Dates and types

RedBeanPHP infers column types from the values it sees: ints, floats, strings, and ISO date/datetime strings. Use the helpers so values are recognised:

```php
$e->start_date = R::isoDate();       // 2026-11-22
$e->created_at = R::isoDateTime();   // 2026-11-22 10:51:46
```

`R::inspect('book')` shows the actual columns and types. Use it, plus `R::debug(true)` to log SQL, when diagnosing surprises.

## Models (FUSE) in brief

Define `class Model_Book extends \RedBeanPHP\SimpleModel` (or the `\RedBeanPHP\SimpleModel` form under Composer) and RedBeanPHP calls its hooks automatically: `dispense`, `open`, `update`, `after_update`, `delete`, `after_delete`. Throw an exception from `update()` to veto a save. Details in `references/models.md`.

## Debugging

```php
R::debug(true, 1);        // log every query (mode 1 = readable output)
echo R::inspect();        // list all tables
print_r(R::inspect('book'));
$bean->export();          // bean as array
R::exportAll($beans);     // many beans as arrays (with loaded relations)
$bean->hasChanged('title');
```

When the user reports an error, ask for or look at: the exact exception class, whether the DB is frozen, and the generated SQL (from `R::debug`).

## Output conventions for this skill

- Prefer the procedural `R::` facade style; it is idiomatic RedBeanPHP and needs no framework.
- Always bind parameters; show the bound form even in throwaway examples.
- State whether an example assumes fluid or frozen mode when it matters.
- Mention the production-hardening step (`references/production.md`) whenever code is shown that will ship to a live site.
- Point to the official docs at https://redbeanphp.com for anything this skill does not cover, and verify version-specific behaviour against the installed `rb.php` rather than guessing.
