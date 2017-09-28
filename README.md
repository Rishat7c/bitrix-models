[![Latest Stable Version](https://poser.pugx.org/arrilot/bitrix-models/v/stable.svg)](https://packagist.org/packages/arrilot/bitrix-models/)
[![Total Downloads](https://img.shields.io/packagist/dt/arrilot/bitrix-models.svg?style=flat)](https://packagist.org/packages/Arrilot/bitrix-models)
[![Build Status](https://img.shields.io/travis/arrilot/bitrix-models/master.svg?style=flat)](https://travis-ci.org/arrilot/bitrix-models)
[![Scrutinizer Quality Score](https://scrutinizer-ci.com/g/arrilot/bitrix-models/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/arrilot/bitrix-models/)

# Bitrix models (in development)

## Вступление 

Данный пакет привносит Model Layer в Битрикс.
Он логически состоит из двух частей:

1. Модели для сущностей битрикса (Битрикс-модели), которые представляет собой надстройку над традиционным API Битрикса, которая выглядит как Eloquent
2. Модели для произвольных таблицы - интеграция `illuminate/support` в целом и `Eloquent` в частности.

## Установка

1. ```composer require arrilot/bitrix-models```
2. Регистрируем пакет в `init.php` - `Arrilot\BitrixModels\ServiceProvider::register();`

Теперь можно создавать свои модели, наследуя их либо от одного из перечисленных классов. 
```php
Arrilot\BitrixModels\Models\ElementModel
Arrilot\BitrixModels\Models\SectionModel
Arrilot\BitrixModels\Models\UserModel
```

## Использование моделей Bitrix

Для пример далее везде будем рассматривать модель для элемента инфоблока (ElementModel). 
Для других сущностей API практически идентичен.

Создадим, модель для инфоблока Товары.
```php
<?php

use Arrilot\BitrixModels\Models\ElementModel;

class Product extends ElementModel
{
    /**
     * Corresponding iblock id.
     *
     * @return int
     */
    const IBLOCK_ID = 1;
}
```

Для работы модели необходимо лишь задать ID инфоблока в константе.
Для юзеров не требуется и этого.

> Если вы не хотите привязываться к ID инфоблоков, можно переопределить в моделе метод `public static iblockId()` и получать в нём ID инфоблока например по коду. Это дает большую гибкость, но скорее всего вам потребуются дополнительные запросы в БД.

В дальнейшем мы будем использовать наш класс `Product` как в статическом, так и в динамическом контексте.

### Добавление продукта

```php
// $fields - массив, аналогичный передаваемому в CIblockElement::Add(), но IBLOCK_ID в нём можно не указывать.
$product = Product::create($fields);
```

### Обновление

```php
// вариант 1
$product['NAME'] = 'Новое имя продукта';
$product->save();

// вариант 2
$product->update(['NAME' => 'Новое имя продукта']);
```

### Инстанцирование модели без запросов к базе.

Для ряда операций нет необходимости в получении информации из БД, достаточно лишь ID объекта.
В этом случае достаточно инстанцировать объект модели, передав идентификатор.
```php
$product = new Product($id);

//теперь есть возможно работать с моделью, допустим
$product->deactivate();
```

### Получение полей из базы

```php
$product = new Product($id);

// метод `load` обращается к базе, только если информация еще не была получена.
$product->load();

// Если мы хотим принудительно обновить информацию из базы даже если она уже была получена ранее
$product->refresh();

// После любого из этих методов, мы можем работать с полученными полями (`echo $product['CODE'];`)

//Для текущего пользователья есть отдельный хэлпер
$user = User::current();
// В итоге мы получаем инстанс User с заполненными полями. 
// Сколько бы раз мы не вызывали `User::current()` в рамках работы скрипта, запрос в базу происходит только один раз - первый.
// `User::freshCurrent()` - то же самое, но получает данные из базы каждый раз.
```

Описанные методы сохраняют данные из БД внутри экземпляра класса-модели.
Объекты-модели реализуют ArrayAccess, поэтому с ними можно во многом работать как с массивами.
```php
$product->load();
if ($product['CODE'] === 'test') {
    $product->deactivate();
}
```

### Преобразование моделей в массив/json.

```php
$array = $product->toArray();
$json = $product->toJson();
```

### Получение информации из базы

Наиболее распостраненный сценарий работы с моделями - получение элементов/списков из БД.
Для построение запроса используется "Fluent API", который использует в недрах себя стандартный Битриксовый API.

Для начала построения запроса используется статический метод `::query()`.
Данный метод возвращает объект-построитель запроса (`ElementQuery`, `SectionQuery` или `UserQuery`), через который уже строится цепочка запроса.

Простейший пример:
```php
$products = Product::query()->select('ID')->getList();
```

На самом деле данная форма приведена больше для понимания, есть более удобный вид, который использует `__callStatic` для передачи управления в объект-запрос.
```php
$products = Product::select('ID')->getList();
```

Любая цепочка запросов должна заканчиваться одним из следующих методов:

1. `->getList()` - получение коллекции (см. http://laravel.com/docs/master/collections) объектов. По умолчанию ключом каждого элемента является его ID.
2. `->getById($id)` - получение объекта по его ID.
3. `->first()` - получение одного (самого первого) объекта удовлетворяющего параметрам запроса.
4. `->count()` - получение количества объектов.
5. `->paginate() или ->simplePaginate()` - получение спагинированного списка с мета-данными (см. http://laravel.com/docs/master/pagination)
6. Методы для отдельных сущностей:
 `->getByLogin($login)` и `->getByEmail($email)` - получение первого попавшегося юзера с данным логином/email.
 `->getByCode($code)` и `->getByExternalId($id)` - получение первого попавшегося элемента или раздела ИБ по CODE/EXTERNAL_ID

#### Управление выборкой

1. `->sort($array)` - аналог `$arSort` (первого параметра `CIBlockElement::GetList`)

 Примеры:
 
 `->sort(['NAME' => 'ASC', 'ID => 'DESC'])`

 `->sort('NAME', 'DESC') // = ->sort(['NAME' => 'DESC'])`

 `->sort('NAME') // = ->sort(['NAME' => 'ASC'])`


2. `->filter($array)` - аналог `$arFilter`
3. `->navigation($array)`
4. `->select(...)` - аналог `$arSelect`

 Примеры:

 `->select(['ID', 'NAME'])`
 
 `->select('ID', 'NAME')`

 `select()` поддерживает два дополнительных значения - `'FIELDS'` (выбрать все поля), `'PROPS'` (выбрать все свойства).
 Для пользователей также можно указать `'GROUPS'` (добавить группы пользователя в выборку).
 Значение по-умолчанию для `ElementModel` - `['FIELDS', 'PROPS']`

5. `->limit($int)`, `->take($int)`, `->page($int)`, `->forPage($page, $perPage)` - для навигации 


#### Некоторые дополнительные моменты:

1. Для ограничения выборки добавлены алиасы `limit($value)` (соответсвует `nPageSize`) и `page($num)` (соответсвует `iNumPage`)
2. В некоторых местах API более дружелюбный чем в самом Битриксе. Допустим в фильтре по пользователям не обязательно использовать
`'GROUP_IDS'`. При передаче `'GROUP_ID'` (именно такой ключ требует Битрикс допустим при создании пользователя) или `'GROUPS'` 
результат будет аналогичен.


### Query Scopes

Построитель запросов можно расширять добавляя "query scopes" в модель.
Для этого необходимо создать публичный метод начинаюзищися со `scope`.

Пример "query scope"-a уже присутсвующего в пакете.
```php
    /**
     * Scope to get only active items.
     *
     * @param BaseQuery $query
     *
     * @return BaseQuery
     */
    public function scopeActive($query)
    {
        $query->filter['ACTIVE'] = 'Y';
    
        return $query;
    }

...

$products = Product::filter(['SECTION_ID' => $secId])
                    ->active()
                    ->getList();
```

В "query scopes" можно также передавать дополнительные параметры.
```php

    /**
     * @param ElementQuery $query
     * @param string|array $category
     *
     * @return ElementQuery
     */
    public function scopeFromSectionWithCode($query, $category)
    {
        $query->filter['SECTION_CODE'] = $category;

        return $query;
    }

...
$users = Product::fromSectionWithCode('sale')->getList();
```

Данные скоупы уже присутсвуют в пакете, ими можно пользоваться.

#### Остановка действия

Иногда требуется остановить выборку из базы из query scope. 
Для этого достаточно вернуть false.
Пример:
```php
    public function scopeFromCategory($query, $category)
    {
        if (!$category) {
            return false;
        }
        
        $query->filter['SECTION_CODE'] = $category;

        return $query;
    }
...
```
В результате запрос в базу не будет сделан - `getList()` вернёт пустую коллекцию, `getById()` - false, а `count()` - 0.

> Того же самого эффекта можно добиться вызвав вручную метод `->stopQuery()` 


### Accessors

Временами возникает потребность как то модифицировать данные между выборкой их из базы и получением их из модели.
Для этого используются акссессоры.
Также как и для "query scopes", для добавления аксессора необходимо добавить метод в соответсвующую модель.

Правило именования метода - `$methodName = "get".camelCase($field)."Attribute"`.
Пример акссессора который принудительно делает первую букву имени заглавной:
```php

    public function getXmlIdAttribute($value)
    {
        return (int) $value;  
    }
    
    // теперь в $product['XML_ID'] всегда будет целочисленное значение
    
```
Этим надо пользоваться с осторожностью, потому оригинальное значение становится недоступным.

Аксессоры можно создавать также и для несуществущих, виртуальных полей, например:
```php
    public function getFullNameAttribute()
    {
        return $this['NAME']." ".$this['LAST_NAME'];
    }
    
    ...
    
    echo $user['NAME']; // John
    echo $user['LAST_NAME']; // Doe
    echo $user['FULL_NAME']; // John Doe
```

Для того чтобы такие виртуальные аксессоры отображались в toArray() и toJson(), их необходимо явно указать в поле $appends модели.
```php
    protected $appends = ['FULL_NAME'];
```

### События моделей (Model Events)

События позволяют вклиниваться в различные точки жизненного цикла модели и выполнять в них произвольный код.
Например, автоматически проставлять символьный код при создании элемента.

События моделей не используют событийную модель Битрикса.
Обработчик события задается переопределением соответсвующего метода в классе-модели.

```php
class News extends ElementModel
{
    /**
     * Hook into before item create or update.
     *
     * @return mixed
     */
    protected function onBeforeSave()
    {
        $this['CODE'] = CUtil::translit($this['NAME'], "ru");
    }

    /**
     * Hook into after item create or update.
     *
     * @param bool $result
     *
     * @return void
     */
    protected function onAfterSave($result)
    {
        //
    }
}
```

Сигнатуры обработчиков других событий совпадают с приведенными выше.

Список доступных эвентов:

1. `onBeforeCreate` - перед добавлением записи
2. `onAfterCreate` - после добавления записи
3. `onBeforeUpdate` - перед обновлением записи
4. `onAfterUpdate` - после обновления записи
5. `onBeforeSave` - перед добавлением или обновлением записи
6. `onAfterSave` - после добавления или обновления записи
7. `onBeforeDelete` - перед удалением записи
8. `onAfterDelete` - после удаления записи

Если сделать `return false;` из обработчика `onBefore...` то последующее действие будет отменено.

## Использование моделей Eloquent

Вторая часть пакета - модели для пользовательских таблиц, то есть созданных вручную, а не через админку Битрикса.
Для работы с ними мы можем вместо "ORM d7" или прямых запросов в базу подключить полновесную ORM от Laravel - [Eloquent](https://laravel.com/docs/master/eloquent)
Стоит учитывать что в отличии от Битрикса `Eloquent` использует в качестве расширения PHP для доступа к mysql не mysql/mysqli а PDO
А это значит что во-первых необходимо чтобы PDO был установлен и настроен, а во-вторых что у вас будут создаваться два подключения к базе на запрос.

Первым делом нужно поставить дополнительную зависимость - `composer require illuminate/database`
После этого добавляем в `init.php` еще одну строку - `Arrilot\BitrixModels\ServiceProvider::registerEloquent();`
Теперь уже можно создавать Eloquent модели наследуясь от `EloquentModel`

```php
<?php

use Arrilot\BitrixModels\Models\EloquentModel;

class Product extends EloquentModel
{
    protected $table = 'app_products';
}
```
Если таблица называется `products` (множественная форма названия класса), то `protected $table = 'products';` можно опустить - это стандартное для Eloquent поведение.
Из нестандартного
 1. Первичным ключом является `ID`, а не `id`
 2. Поля для времени создания и обновления записи называются `CREATED_AT` и `UPDATED_AT`, а не `created_at` и `updated_at`

> Если вы решили не добавлять в таблицу поля CREATED_AT и UPDATED_AT, то в модели нужно задать `public $timestamps = false;`

Через `Eloquent` можно работать не только с пользовательскими таблицами, но и с Highload-блоками, что очень удобно.
Для этого мы работаем напрямую с таблицей Highload-блока минуя API Битрикса.

Например, представим что мы создали Highload-блок для брендов `Brands`, задали для него таблицу `brands` и добавили в него свойство `UF_NAME`.
Тогда класс-модель для него будет выглядеть так:

```php
<?php

use Arrilot\BitrixModels\Models\EloquentModel;

class Brand extends EloquentModel
{
    public $timestamps = false;
}
```

А для добавления новой записи в него достаточно следующего кода:
```php
$brand = new Brand();
$brand['UF_NAME'] = 'Nike';
$brand->save();

// либо даже такого если настроены $fillable поля.
$brand = Brand::create(['UF_NAME' => 'Nike']);
```

Для полноценной работой с Eloquent-моделями важно ознакомиться с официальной документацией данной ORM [(ссылка еще раз)](http://laravel.com/docs/master/eloquent)

В заключение обращу внимание на то что, несмотря на то что API моделей Битрикса и моделей Eloquent очень похожи (во многом из-за того что bitrix-models разрабатывались под влияением Eloquent)
это всё же разные вещи и внутри они совершенно независимые. Нельзя допустим сделать связь (relation) Eloquent модели и Битриксовой модели.

### Query Builder

При подключении Eloquent мы бесплатно получаем и Query Builder от Laravel [https://laravel.com/docs/master/queries](https://laravel.com/docs/master/queries), который очень полезен если необходима прямая работа с БД минуя уровень абстракции моделей.
Он удобнее и несравнимо безопаснее чем `$DB->Query()` и прочее.

Работа с билдером проводится через глобально доступный класс `DB`.
Например добавление элемента бренда в HL-блок будет выглядеть так:
```php
DB::table('brands')->insert(['UF_NAME' => 'Nike']);
```

## Постраничная навигация (pagination)

И Битрикс-модели и Eloquent-модели поддерживают `->paginate()` и `->simplePaginate()` (см. http://laravel.com/docs/master/pagination)
Для того чтобы затем отобразить постраничную навигацию через `->links()` нужно

 1. Установить [https://github.com/arrilot/bitrix-blade/](https://github.com/arrilot/bitrix-blade/)
 2. Скопировать дефолтные вьюшки из [https://github.com/laravel/framework/tree/master/src/Illuminate/Pagination/resources/views](https://github.com/laravel/framework/tree/master/src/Illuminate/Pagination/resources/views) в `local/views/pagination`

После этого вьюшки можно модифицировать или создавать новые.
