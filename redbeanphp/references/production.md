# Taking RedBeanPHP to production

Fluid mode is a development convenience. For a live site, lock the schema down.

## Contents
- Freeze the schema
- Workflow: develop fluid, export, freeze
- Input handling
- Foreign keys and indexes
- Multiple databases
- Transactions
- Checklist

## Freeze the schema

```php
R::setup($dsn, $user, $pass);
R::useFeatureSet('novice/latest');
R::freeze(true);
```

Frozen mode never creates or alters tables or columns. It is faster (no schema inspection per store) and prevents user-controlled property names from adding columns. Storing a bean with an unknown column then fails with a `RedBeanPHP\RedException\SQL` error such as "table book has no column named x". That error in production means a column is missing, not that RedBeanPHP is broken.

You can freeze only some tables: `R::freeze(['audit', 'user'])`.

## Workflow: develop fluid, export, freeze

1. Develop and test in fluid mode against a scratch database.
2. Dump the generated schema (`mysqldump --no-data`, `pg_dump -s`, `.schema` in SQLite) into a SQL migration file under version control.
3. Review it by hand: tighten column types (`VARCHAR(191)` not `TEXT`, `DECIMAL(10,2)` for money, `DATETIME`), add NOT NULL, defaults, and unique indexes.
4. Apply it on the live server, then enable `R::freeze(true)`.
5. For later changes, write a new migration. Do not unfreeze in production.

## Input handling

```php
$bean = R::dispense('user');
$bean->import($_POST, 'name,email');          // whitelist the keys you accept
```

Never loop over raw request arrays into properties. Validate and cast before store. Hash passwords with `password_hash()`, never store them raw in a bean.

All SQL strings passed to `find`, `findOne`, `findAll`, `getAll`, `exec`, `count` must use `?` or `:name` bindings. The only fragments you may interpolate are values you chose yourself, such as a whitelisted column name for `ORDER BY`:

```php
$allowed = ['title', 'created_at'];
$col = in_array($sort, $allowed, true) ? $sort : 'title';
$books = R::findAll('book', " ORDER BY $col LIMIT ? ", [20]);
```

## Foreign keys and indexes

In fluid mode RedBeanPHP adds foreign keys and indexes for relation columns where the database supports it, but it will not index your own search columns. Add indexes for columns you filter or sort on:

```sql
CREATE INDEX idx_book_title ON book (title);
CREATE UNIQUE INDEX uq_user_email ON user (email);
```

Catch the failure when storing a duplicate:

```php
try { R::store($user); }
catch (\RedBeanPHP\RedException\SQL $e) { /* duplicate email, etc. */ }
```

## Multiple databases

```php
R::setup($dsnMain, $u, $p);                                   // becomes 'default'
R::addDatabase('stats', 'sqlite:/var/data/stats.db', null, null, true); // last arg: frozen
R::selectDatabase('stats');
// ... queries here go to 'stats' ...
R::selectDatabase('default');
```

Always switch back; the facade is global state.

## Transactions

```php
R::begin();
try {
    // several stores
    R::commit();
} catch (\Throwable $e) {
    R::rollback();
    throw $e;
}
```

`R::transaction(function () { ... })` does the same with one call, but it returns `FALSE` and skips the callback in fluid mode unless `R::setAllowFluidTransactions(true)` is set. In frozen mode it commits on success, rolls back and rethrows on exception, and returns the callback's result.

## Checklist

- [ ] `R::freeze(true)` on production
- [ ] schema exported to a reviewed migration
- [ ] `novice/latest` feature set (no `R::nuke()` / `R::wipe()` surprises)
- [ ] all SQL bound, all input whitelisted
- [ ] indexes on filter and sort columns, unique constraints in SQL
- [ ] credentials outside the web root or in environment variables
- [ ] `R::debug()` removed
