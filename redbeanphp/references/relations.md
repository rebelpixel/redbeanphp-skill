# RedBeanPHP relations reference

All examples assume `require 'rb.php'; R::setup(...);`. Relation list names end in `List` (`ownPageList`). The older form without `List` (`ownPage`) still works; keep it consistent within a codebase.

## Contents
- One-to-many and many-to-one
- Many-to-many
- Aliases: several relations to the same type
- Extended many-to-many with `via`
- Cascade: `xown`
- Filtering and counting
- Tagging
- Exporting and counting traps

## One-to-many and many-to-one

```php
$book = R::dispense('book');
$book->title = 'Atlas';
$page = R::dispense('page');
$page->num = 1;

$book->ownPageList[] = $page;   // adds page.book_id = book.id
R::store($book);                // stores book and its pages

$page = R::findOne('page', '1');
echo $page->book->title;        // parent, loaded on first access
```

Alternative from the child side:

```php
$page->book = $book;            // sets book_id
R::store($page);
```

A one-to-one relation is just a single parent reference (`$a->b = $b`).

## Many-to-many

```php
$tag = R::dispense('tag');
$tag->name = 'travel';
$book->sharedTagList[] = $tag;
R::store($book);

$book = R::load('book', $id);
foreach ($book->sharedTagList as $t) { echo $t->name; }

// Remove one link without deleting the tag
unset($book->sharedTagList[$tagId]);   // keyed by bean id
R::store($book);
```

The link table is `book_tag` (alphabetical). It has `book_id` and `tag_id`. Link rows are removed automatically when either side is trashed.

## Aliases: several relations to the same type

When a bean needs two references to the same type (teacher and student are both `person`), name the property after the role and tell RedBeanPHP which type it is:

```php
$project = R::dispense('project');
$project->teacher = $teacherPerson;    // creates project.teacher_id
$project->student = $studentPerson;
R::store($project);

$teacher = R::load('project', $id)->fetchAs('person')->teacher;   // read the parent as a person

// The reverse direction: projects where this person is the teacher
$projects = $teacherPerson->alias('teacher')->ownProjectList;
```

Without `fetchAs`, RedBeanPHP looks for a table named `teacher` and fails.

## Extended many-to-many with `via`

When the link row needs its own data (role, date, quantity), route through a real bean type:

```php
$book->via('rating')->sharedTagList[] = $tag;   // link table is rating, not book_tag
R::store($book);
```

`rating` can then be loaded and extended like any other bean, for example to store a score.

## Cascade: `xown`

`ownPageList` leaves children behind when the parent is trashed (or removes the reference only). `xownPageList` marks them as exclusively owned: removing a child from the list deletes it, and trashing the parent trashes the children.

```php
$shelf->xownBookList[] = $book;
R::store($shelf);
R::trash($shelf);               // the books are gone too
```

## Filtering and counting

```php
$book->with(' ORDER BY num DESC LIMIT 5 ')->ownPageList;
$book->withCondition(' num > ? ', [10])->ownPageList;
$book->countOwn('page');
$book->countShared('tag');
$book->noLoad()->ownPageList[] = $page;   // append without loading existing children first
```

`with` and `withCondition` apply to a single access; call them again for the next access. Adding to a list that was loaded with a filter and then storing can drop the beans the filter hid, so use `noLoad()` for append-only work.

## Tagging

```php
R::tag($post, 'php,orm');          // sets tags (comma-separated)
R::addTags($post, ['extra']);
R::tag($post);                     // read: returns tag names
$posts = R::tagged('post', 'php'); // beans with the tag
R::untag($post, ['php']);
```

## Exporting and counting traps

- `R::exportAll($beans)` includes only relations already loaded. Access the list first (for example `count($book->ownPageList)`) if you need it in the export.
- Loading `ownXList` on thousands of parents is an N+1 pattern. Prefer one `R::find` / `R::getAll` with a JOIN or an `IN (...)` query and group in PHP.
- `$bean->id` is a string when it comes from the database; compare with `==` or cast.
