# Provedor de serviços Capsule para o Silex

Este provedor é um provedor de serviços para o [Micro-framework Silex](http://silex.sensiolabs.org/) que integra [Laravel's Eloquent ORM](http://laravel.com/docs/5.0/eloquent) via [Capsule](https://github.com/illuminate/database), é uma implementação de um wrapper.

## Requisitos

Para utilizar o provedor de serviços você precisa utilizar o **PHP 5.4+**

## Instalação

Para instalar o provedor de serviços utilize o [Composer](https://getcomposer.org):

````shell
composer require electrolinux/silex-capsule:2.*
````

Alternativamente, você pode adicioná-lo diretamente no arquivo `composer.json`: 

````json
{
    "require": {
        "singularphp/silex-capsule": "2.*"
    }
}
````

## Uso básico

Para usá-lo em sua aplicação, apenas registr o provedor de serviços com o Silex: 

````php
<?php        
$app = new Silex\Application;

$app->register(new Electrolinux\Silex\Provider\CapsuleServiceProvider, [
    'capsule.connection' => [
        'driver'    => 'mysql',
        'host'      => 'localhost',
        'database'  => 'database',
        'username'  => 'username',
        'password'  => 'password',
        'charset'   => 'utf8',
        'collation' => 'utf8_unicode_ci',
        'prefix'    => '',
        'logging'   => true,
    ],
]);
````

Para maiores informações sobre as opções disponíveis você pode consultar a [Documentação do Capsule](https://github.com/illuminate/database).

Uma vez registrado o provedor de servios um objeto Capsule será criado e o Eloquent será iniciado antes de qualquer rota ser chamada. Se você está interessado em aspectos técnicos, Capsule é registrado como um middleware `before` com o Silex usando `Application::EARLY_EVENT`, assim só estará disponível quando sua aplicação  é executada e antes de qualquer coisa acontecer.

Se você precisa que o Capsule e Eloquent sejam inicializados antes da sua aplicação ser iniciada (`$app->run()`), por exemplo em um comando de linha do Symfony, você precisa apenas acessar o serviço dentro do container de injeção de dependência: 

````php
$app['capsule']; 
````

Capsule estará disponível globalmente por default, permitindo que você escreva queries diretamente em seus controladores como quiser:

````php
use Illuminate\Database\Capsule\Manager as Capsule;

$app->get('/book/{id}', function(Application $app, $id) {
    $book = Capsule::table('books')->where('id', $id)->get();
    return $app->json($book);
});
````

Se você não quiser que o Capsule seja inicializado então defina a configuração `capsule.global` para `false`. Se você não planejar usar o Eloquente para construir modelos então você pode evitar que ele seja inicializado definindo a configuração `capsule.eloquent` para `false`.

Criar modelos no Eloquent é identico à forma como eles são criados no Laravel: 

````php
use Illuminate\Database\Eloquent\Model;

class Book extends Model
{
    protected $table = 'books';

    protected $fillable = [
        'title', 
        'author',
    ];

    protected $casts = [
        'title'  => 'string',
        'author' => 'string',
    ];

    // The rest of your model code...
}
````

Então você pode usá-lo, e todos os seus recursos, em seus controladores assim como você faria no Laravel: 

````php
$app->get('/books', function(Application $app) {
    $books = Book::with('tags')->all();
    return $app->json($books);
});

$app->post('/book', function(Application $app, Request $request) {
    $book         = new Book();
    $book->title  = $request->request->get('title');
    $book->author = $request->request->get('author');
    $book->save();
});
````

## Uso avanado

Você pode configurar multiplas conexões e mesmo cachear com o provedor de serviços; simplismente usando a opção `capsule.connections`:

````php
<?php
$app = new Silex\Application;

$app->register(new Electrolinux\Silex\Provider\CapsuleServiceProvider, [
    // Connections
    'capsule.connections' => [
        'default' => [
            'driver'    => 'mysql',
            'host'      => 'localhost',
            'database'  => 'dname1',
            'username'  => 'root',
            'password'  => '',
            'charset'   => 'utf8',
            'collation' => 'utf8_unicode_ci',
            'prefix'    => '',
            'logging'   => false,
        ],

        'other' => [
            'driver'    => 'mysql',
            'host'      => 'localhost',
            'database'  => 'dbname2',
            'username'  => 'root',
            'password'  => '',
            'charset'   => 'utf8',
            'collation' => 'utf8_unicode_ci',
            'prefix'    => '',
            'logging'   => true,
        ],
    ],
]);
````

Se você habilitou o log de queries em sua conexão, você pode recuperá-los através do Capsule: 

````php
Capsule::connection($name)->getQueryLog();
````

Você pode alternar o mecanismo de log por conexão usando a opção `logging` em suas credenciais de conexão.

Você pode também usar o criador de esquema do Eloquent para construir migrations: 

````php
$app['capsule']->schema()->create('books', function($table) {
    $table->increments('id');
    $table->string('title');
    $table->string('author');
    $table->timestamps();
});
````

Por default o provedor de serviços instala o pacote Laravel Events, assim você pode também usar observadores para os modelos: 

````php
<?php
class BookObserver 
{
    public function saving($model)
    {
        // Do something
    }
}

Book:observe(new BookObserver());
````

## Exemplo de opções do Capsule

A seguir um exemplo completo de todas as opções que você pode passar para o provedor de serviços: 

````php
<?php
$app = new Silex\Application;

$app->register(new Electrolinux\Silex\Provider\CapsuleServiceProvider, [
    // Multiplas conexões
    'capsule.connections' => [
        'default' => [
            'driver'    => 'mysql',
            'host'      => 'localhost',
            'database'  => 'dname1',
            'username'  => 'root',
            'password'  => '',
            'charset'   => 'utf8',
            'collation' => 'utf8_unicode_ci',
            'prefix'    => '',
            'logging'   => false, // Desabilita o log para esta conexão
        ],

        'other' => [
            'driver'    => 'mysql',
            'host'      => 'localhost',
            'database'  => 'dbname2',
            'username'  => 'root',
            'password'  => '',
            'charset'   => 'utf8',
            'collation' => 'utf8_unicode_ci',
            'prefix'    => '',
            'logging'   => true,  // Habilita o log para esta conexão
        ],
    ],

    /*
    // Conexão única
    'capsule.connection' => [
        'driver'    => 'mysql',
        'host'      => 'localhost',
        'database'  => 'dbname',
        'username'  => 'root',
        'password'  => '',
        'charset'   => 'utf8',
        'collation' => 'utf8_unicode_ci',
        'prefix'    => '',
        'logging'   => true, // Habilita o log para esta conexão
    ],
    */

    /*
    // Outras opções
    'capsule.global'   => true, // Habilita o acesso global ao query builder do Capsule.
    'capsule.eloquent' => true, // Automaticamente inicializa o Eloquent ORM.
    */
]);
````

## Testes

Há alguns testes básicos para certificar que o objeto Capsule está corretamente registrado com o Silex. Você pode executá-los usando [PHPUnit](https://phpunit.de/), e você irá tambm precisar do SQLite para fazer testes de banco de dados em memória. 

Se você fizer um pull request certifique-se de que adicionou os testes que o acopanham.
