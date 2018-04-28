---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: page
---

Still in progress.
You SHOULD NOT use this package until release 1.0.

This is something close to a DataMapper, with very high flexibility but need more code than other libraries to be used.

### Run a query

Nothing new, it's the [Doctrine DBAL](https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/)

{% highlight php %}
use Doctrine\DBAL\Configuration;
use Doctrine\DBAL\DriverManager;

$connection = DriverManager::getConnection(['url' => 'mysql://...'], new Configuration());
$queryBuilder = $connection->createQueryBuilder();
$queryBuilder
    ->select('id', 'name')
    ->from('user');

$rows = $queryBuilder->execute();

foreach ($rows as $row) {
    print_r($row);
}
{% endhighlight %}

Output is
{% highlight php %}
Array
(
    [id] => 1
    [name] => Sylvain
)
{% endhighlight %}

### How to hydrate (without class)

Nothing to learn, it's pure PHP, do like you want.

Assuming you have this class
{% highlight php %}
class User
{
    public function __construct(int $id, string $name)
    {
        $this->id   = $id;
        $this->name = $name;
    }

    public function getId(): int
    {
        return $this->id;
    }

    public function getName(): string
    {
        return $this->name;
    }
}
{% endhighlight %}

You can do
{% highlight php %}
// ... from previous example
$rows = $queryBuilder->execute();
$rowsHydrated = array_map(
    function ($row) {
        return new User($row['id'], $row['name']);
    },
    $rows->fetchAll()
);

foreach ($rowsHydrated as $userId => $user) {
    print_r($user);
}
{% endhighlight %}

Output is
{% highlight php %}
User Object
(
    [id] => 1
    [name] => Sylvain
)
{% endhighlight %}

### How to hydrate (with class)

Nothing to learn, it's pure PHP, do how you want.

Assuming you have this class
{% highlight php %}
class UserMapper
{
    public function map(iterable $rows): \ArrayObject
    {
        $collection = new \ArrayObject();

        foreach ($rows as $row) {
            $collection->append(new User($row['id'], $row['name']));
        }

        return $collection;
    }
}
{% endhighlight %}

You can do
{% highlight php %}
// ... from previous example
$rows = $queryBuilder->execute();

$userMapper = new UserMapper();
$rowsHydrated = $userMapper->map($rows);

foreach ($rowsHydrated as $userId => $user) {
    print_r($user);
}
{% endhighlight %}

Output is
{% highlight php %}
User Object
(
    [id] => 1
    [name] => Sylvain
)
{% endhighlight %}

### Type mapping

Almost nothing new, it's [Doctrine DBAL Types](https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/types.html) to convert SQL types to PHP types
But, you have a TypeConverter that's a wrapper to convert your type more easily.

Assuming you have this class
{% highlight php %}
class User
{
    public function __construct(int $id, string $name, DateTime $lastUpdated)
    {
        $this->id          = $id;
        $this->name        = $name;
        $this->lastUpdated = $lastUpdated;
    }

    public function getId(): int
    {
        return $this->id;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getLastUpdated(): DateTime
    {
        return $this->lastUpdated;
    }
}
{% endhighlight %}

You can do
{% highlight php %}
use Blackprism\Nothing\TypeConverter;

// ... from previous example
$queryBuilder
    ->select('id', 'name', 'last_updated')
    ->from('user');
$rows = $queryBuilder->execute();

$typeConverter = new TypeConverter($connection->getDatabasePlatform());

$rowsHydrated = array_map(
    function ($row) use ($typeConverter) {
        return new User(
            $row['id'],
            $row['name'],
            $typeConverter->convertToPHP($row['last_updated'], 'datetime')
        );
    },
    $rows->fetchAll()
);

foreach ($rowsHydrated as $userId => $user) {
    print_r($user);
}
{% endhighlight %}

Output is
{% highlight php %}
User Object
(
    [id] => 1
    [name] => Sylvain
    [lastUpdated] => DateTime Object
        (
            [date] => 2018-04-16 03:02:01.000000
            [timezone_type] => 3
            [timezone] => UTC
        )

)
{% endhighlight %}

### Custom type

Assuming you have this class
{% highlight php %}
use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\StringType;

class PrefixStringType extends StringType
{
    /**
     * @param mixed $value
     * @param AbstractPlatform $platform
     *
     * @return string
     */
    public function convertToPHPValue($value, AbstractPlatform $platform)
    {
        return 'prefixed ' . $value;
    }

    /**
     * @return string
     */
    public function getName()
    {
        return 'prefixed_string';
    }
}
{% endhighlight %}

You can do
{% highlight php %}
use Blackprism\Nothing\TypeConverter;
use Doctrine\DBAL\Types\Type;

// ... from previous example
$queryBuilder
    ->select('id', 'name', 'last_updated')
    ->from('user');
$rows = $queryBuilder->execute();

Type::addType('prefixed_string', PrefixStringType::class);
$typeConverter = new TypeConverter($connection->getDatabasePlatform());

$rowsHydrated = array_map(
    function ($row) use ($typeConverter) {
        return new User(
            $row['id'],
            $typeConverter->convertToPHP($row['name'], 'prefixed_string'),
            $typeConverter->convertToPHP($row['last_updated'], 'datetime')
        );
    },
    $rows->fetchAll()
);

foreach ($rowsHydrated as $userId => $user) {
    print_r($user);
}
{% endhighlight %}

Output is
{% highlight php %}
User Object
(
    [id] => 1
    [name] => prefixed Sylvain
    [lastUpdated] => DateTime Object
        (
            [date] => 2018-04-16 03:02:01.000000
            [timezone_type] => 3
            [timezone] => UTC
        )

)
{% endhighlight %}

### AutoMapping

Assuming you have this code
{% highlight php %}
use Blackprism\Nothing\EntityMapping;

$userMapping = new EntityMapping(
    User::class,
    [
       'id'           => 'integer',
       'name'         => 'string',
       'last_updated' => 'datetime',
    ]
);
{% endhighlight %}

You can do
{% highlight php %}
use Blackprism\Nothing\AutoMapping;

// ... from previous example
$queryBuilder
    ->select('id', 'name', 'last_updated')
    ->from('user');
$rows = $queryBuilder->execute();

$autoMapping = new AutoMapping($connection->getDatabasePlatform(), [$userMapping]);
$rowsHydrated = $autoMapping->map($rows);

foreach ($rowsHydrated as $userId => $user) {
    print_r($user);
}
{% endhighlight %}

Output is
{% highlight php %}
Array
(
    [User] => User Object
        (
            [id] => 2
            [name] => François
            [lastUpdated] => DateTime Object
                (
                    [date] => 2018-04-23 00:00:00.000000
                    [timezone_type] => 3
                    [timezone] => UTC
                )
        )
)
{% endhighlight %}

#### You can prefix the column
{% highlight php %}
use Blackprism\Nothing\AutoMapping;

// ... from previous example
$queryBuilder
    ->select('id as user_id', 'name as user_name', 'last_updated as user_last_updated')
    ->from('user');
$rows = $queryBuilder->execute();

$autoMapping = new AutoMapping($connection->getDatabasePlatform(), ['user_' => $userMapping]);
$rowsHydrated = $autoMapping->map($rows);

foreach ($rowsHydrated as $userId => $user) {
    print_r($user);
}
{% endhighlight %}

#### You can embed object into an other
{% highlight php %}
$authorMapping = new EntityMapping(
    Author::class,
    [
        'id'   => 'integer',
        'name' => 'string'
    ]
);

$bookMapping = new EntityMapping(
    Book::class,
    [
       'id'   => 'integer',
       'name' => 'string',
       Author::class => AutoMapping::SUB_OBJECT
    ]
);
{% endhighlight %}

#### You can add named constructor
{% highlight php %}

$bookMapping = new EntityMapping(
    Book::class,
    [
       'id'   => 'integer',
       'name' => 'string'
    ]
);

$bookMapping->buildWith(
    'withAuthor',
    [
        'id'   => 'integer',
        'name' => 'string',
        Author::class => AutoMapping::SUB_OBJECT
    ]
);
{% endhighlight %}

AutoMapping will do :
{% highlight php %}
Book::withAuthor($row['id'], $row['name'], new Author(...
{% endhighlight %}

#### AutoAlias helper
When you build your query you'll often use alias like this :
{% highlight php %}
$queryBuilder->select('book.id as book_id', 'book.name as book_name', 'author.id as author_id', 'author.name as author_name')
{% endhighlight %}

AutoAlias help you to make this easier :
{% highlight php %}
$queryBuilder->select((new AutoAlias)('book.id', 'book.name', 'author.id', 'author.name'))
{% endhighlight %}

### Sample

https://github.com/blackprism/nothing-sample