# RedBeanPHP models (FUSE)

FUSE lets a bean automatically use a model class of the same name. For bean type `book`, the class is `Model_Book`.

```php
class Model_Book extends \RedBeanPHP\SimpleModel
{
    public function update()
    {
        // runs before the bean is stored; throw to veto the save
        if (!$this->bean->title) {
            throw new Exception('A book needs a title');
        }
    }

    public function after_update()
    {
        // runs after the bean is stored; $this->bean->id is available for new beans
    }
}
```

With the single-file install the base class may be written `RedBean_SimpleModel` in old code; `\RedBeanPHP\SimpleModel` also works. With Composer, always use `\RedBeanPHP\SimpleModel`.

Inside a model, the bean is `$this->bean`; reading `$this->title` also works through magic access.

## Hooks, in the order they matter

| Method          | Fires                                   |
|-----------------|-----------------------------------------|
| `dispense()`    | when the bean is created                |
| `open()`        | after the bean is loaded                |
| `update()`      | before store (insert or update)         |
| `after_update()`| after store                             |
| `delete()`      | before trash                            |
| `after_delete()`| after trash                             |

Verified: an exception thrown in `update()` aborts the store, and `after_update()` runs after a successful one.

## Namespaced models

Tell RedBeanPHP where to look before the first dispense:

```php
define('REDBEAN_MODEL_PREFIX', '\\App\\Model\\');
// bean type 'book' now maps to \App\Model\Book
```

## Custom methods

Public methods on the model are reachable through the bean:

```php
class Model_Book extends \RedBeanPHP\SimpleModel {
    public function shortTitle() { return substr($this->bean->title, 0, 20); }
}
echo R::load('book', 1)->box()->shortTitle();
```

`$bean->box()` returns the model; `$model->unbox()` returns the bean.

## When not to use FUSE

For small procedural projects, keep validation in a plain function that runs before `R::store()`. FUSE hooks are global side effects (they fire from anywhere a bean of that type is stored), which can surprise people reading the call site.
