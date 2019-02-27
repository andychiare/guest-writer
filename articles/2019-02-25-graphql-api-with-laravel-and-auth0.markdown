---
layout: post
title: "Create a secure GraphQL API with Laravel and Auth0"
description: "This article shows how to create a GraphQL API with Laravel and how to secure it by integrating with Auth0"
date: "2019-02-25 08:30"
author:
  name: "Andrea Chiarelli"
  url: "andychiare"
  mail: "andrea.chiarelli.ac@gmail.com"
  avatar: "https://twitter.com/andychiare/profile_image?size=original"
related:
- 2017-11-15-an-example-of-all-possible-elements
---

**TL;DR:** This article will show how to implement a basic GraphQL API by using the Laravel framework and how to secure it by integrating with Auth0 services. Throughout the article, you will learn how to implement the API step by step up to the final result. You can find the final code on this [GitHub repository](https://github.com/andychiare/laravel-graphql-auth0).

## Introducing the Wine Store API Project

The project you are going to build is an API providing a list of wines from all over the world. As said before, you will build the API following the [GraphQL](https://graphql.org/) model, that is a model that allows a client to request the exact data it needs. You will implement the GraphQL API by using [Laravel](https://laravel.com/), one of the most popular PHP frameworks that allows you to set up an application in minutes by exploiting its powerful infrastructure. Finally, you will secure the API by integrating it with Auth0 services.

Before starting the project, ensure you have [PHP](http://www.php.net/) and [MySQL](https://www.mysql.com/) installed on your machine. You also need [Composer](https://getcomposer.org/), the dependency manager for PHP.

Once you are these tools installed on your machine, you are ready to build *Wine Store* API.

## Setting up the Laravel Project

The first step to create the Laravel project is to run  the following command in a shell window:

```shell
composer create-project --prefer-dist laravel/laravel winestore
```

This command asks *Composer* to create a *Laravel* project named `winestore`. Its result is a `winestore` folder with a few files and subfolders as shown by the following picture:

![](images/laravel-app-folder-structure.PNG)

You will learn the role of most folders in the following, while you will build the application.

> **Note**: You can also create a Laravel project by downloading the *Laravel installer* with the following command:
>
> ```shell
> composer global require laravel/installer
> ```
>
> Then you can create a new project by typing the following command:
>
> ```shell
> laravel new winestore
> ```
>
> 

## Creating the Wine Store Model

Now you can start to modify the scaffolded project in order to implement the *Wine Store* data model.

### Creating the Model and the Migration Script

So, type the following command in a console window:

```shell
php artisan make:model Wine -m
```

This command will create the *Wine* model and a migration script for its database persistence, thanks to the `-m` flag.

The *Wine* model is implemented by the `Wine.php` file that you will find inside the `app` folder. In fact, the `app` folder contains the code representing the application's domain. The content of the `Wine.php` file will be as follows:

```php
//app/Wine.php
<?php
namespace App;

use Illuminate\Database\Eloquent\Model;

class Wine extends Model
{
    //
}
```

As you can see, it is an empty class extending the `Model` class from [Eloquent](https://laravel.com/docs/5.7/eloquent), the *Object-Relational Mapping* (ORM) provided by Laravel. This class represents the template for *Wine* objects that your application is going to manage.

The `make:model` command also generated a migration script, that is a script defining the table structure in a database. You will find the migration script for the *Wine* model in the `database/migrations` folder. Here you will find three files whose name starts with a timestamp, as shown by the following picture:

![](images/migrations-folder.png)

The first two files were generated during the project initialization and are related to the built-in user management provided by Laravel. The last file, ending with `_create_wines_table.php` is the migration script for the *Wine* model. The migration scripts are used by *Eloquent* to create or update the schema of the tables in the application's database. The timestamp prefixes for each file tell *Eloquent* which changes to apply and in which order.

Now, open the file ending with `_create_wines_table.php` and put the following code inside it:

```php
//database/migrations/2019_02_23_110720_create_wines_table.php
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateWinesTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('wines', function (Blueprint $table) {
            $table->increments('id');
        	$table->string('name', 50);
        	$table->text('description');
        	$table->string('color', 10);
        	$table->string('grape_variety', 50);
        	$table->string('country', 50);
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('wines');
    }
}
```

The class `CreateWinesTable` has two methods:

- `up()`
  This method is executed when an upgrade is applied to the database

- `down()`

  This method is executed when a downgrade is applied to the database

As you can see, the `up()` method defines the structure of the table `wines` that represents the *Wine* model in the database.

### Seeding the Database

In order to test the API you are going to create, you need a few initial data inside the table created by the migration script. You can populate the table by creating a *seeder*, that is a class having the task to populate a database table. In order to create a *seeder*, type the following command:

```shell
php artisan make:seeder WinesTableSeeder
```

This command will generate a `WineTableSeeder.php` file in the `database/seeds` folder. Open this file and change its content as follows:

```php
//database/seeds/WinesTableSeeder.php
<?php

use Illuminate\Database\Seeder;
use App\Wine;

class WinesTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        Wine::create([
        	'name' => 'Classic Chianti',
        	'description' => 'A medium-bodied wine characterized by a marvelous freshness with a lingering, fruity finish',
        	'color' => 'red',
        	'grape_variety' => 'Sangiovese',
        	'country' => 'Italy'
        ]);
    
        Wine::create([
        	'name' => 'Bordeaux',
        	'description' => 'A wine with fruit scents and flavors of blackberry, dark cherry, vanilla, coffee bean, and licorice. The wines are often concentrated, powerful, firm and tannic',
        	'color' => 'red',
        	'grape_variety' => 'Merlot',
        	'country' => 'France'
        ]);
    
        Wine::create([
        	'name' => 'White Zinfandel',
        	'description' => 'Often abbreviated as White Zin, it is a dry to sweet wine, pink-colored rosé',
        	'color' => 'rosé',
        	'grape_variety' => 'Zinfandel',
        	'country' => 'USA'
        ]);
 
        Wine::create([
        	'name' => 'Port',
        	'description' => 'A fortified sweet red wine, often served as a dessert wine',
        	'color' => 'red',
        	'grape_variety' => 'Touriga Nacional',
        	'country' => 'Portugal'
        ]);
  
        Wine::create([
        	'name' => 'Prosecco',
        	'description' => 'It is a dry white wine (brut) sometimes with a sweet flavor of green apple, honeydew melon, pear, and honeysuckle',
        	'color' => 'white',
        	'grape_variety' => 'Glera',
        	'country' => 'Italy'
        ]);
    }
}
```

The run() method of the `WinesTableSeeder` class creates instances of the `Wine` class based on the specified values. Now, edit the `DatabaseSeeder.php` file you find in the same folder and invoke the `WinesTableSeeder` class inside the `run()` method. The following is the resulting code:

```php
//database/seeds/DatabaseSeeder.php
<?php

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        $this->call(WinesTableSeeder::class);
    }
}
```

Now, open the `.env` file you find in the project's root folder and configure the database parameters shown below accordingly to your development environment:

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret
```

After all this preparation, you are ready to create the table schema and to populate it. Type the following in a console window:

```shell
php artisan migrate:fresh --seed
```

This command will clear the database, execute the migration scripts and run the database *seeder*.

> **Note**: you may ask how the *Wine* model defined through the `Wine` class can be mapped to the `wines` table. This happens because, by convention, Eloquent maps a model to a table having the same lowercase name in the plural.

## Introducing GraphQL

So far, you defined the model and its database persistence. Now you can build the API upon that.

### Why GraphQL

The API you are going to implement is based on [GraphQL](https://graphql.org/), a specification from Facebook defining a query language for APIs. With respect to classic REST APIs, GraphQL allows you to define a single endpoint providing multiple resources to the clients. This contributes to reduce network traffic and to potentially speed up client applications. In addition, GraphQL allows a client to request just the data it needs, avoiding to receive a resource with all possible information. Again, this reduces the network traffic and optimizes the data processing on the client side.

GraphQL achieves this result by defining an abstract language to describe queries, schemas, and types, in a similar way as in a database. As said before, GraphQL is a [specification](https://facebook.github.io/graphql/). This means that it is independent of any programming language. If you want to use it in your application, you need to choose among the [several available implementations](https://graphql.org/code/) in almost any language. 

### Installing the GraphQL Library 

To support GraphQL in the application you're going to build, you need to install a library that allows you to define schemas and queries in a simple way. The [Laravel GraphQL](https://github.com/rebing/graphql-laravel) library is one of the best. To install it, write the following command:

```shell
composer require rebing/graphql-laravel
```

After the installation, you need to run the following command:

```shell
php artisan vendor:publish --provider="Rebing\GraphQL\GraphQLServiceProvider"
```

This command extracts the `graphql.php` configuration file from the `vendor` folder and put it into the `config` folder. This is a common approach that allows you to get one or more configuration files from a third party package, so that you can change it for the needs of your application. You will use the `graphql.php` later.

## Creating the GraphQL Schema

Since GraphQL is a query language, you need to know how you can build your query, what type of object you can receive as a response, what fields you can request from the server. A GraphQL API endpoint provides a complete description of what your client can query to it. This description is called *schema*, a collection of data defining the queries that a client can request, the type of the returned resources, the allowed change requests to the resources, also known as *mutations*, and other.

To keep things simple, your API will allow you just to retrieve the list of wines in the *Wine Store* and a specific wine. So its schema will consist of queries and types.

### Creating the Wine Type

Create the API's schema by starting with the type definition of the resource returned. So, create the `GraphQL` folder inside the `app` folder. This folder will contain all the definitions you need for the GraphQL schema of the API. In the `app/GraphQL` folder, create the `Types` folder and put in it a file named `WineType.php` with the following content:

```php
// app/GraphQL/Types/WineType.php
<?php
namespace App\GraphQL\Types;

use GraphQL;
use Rebing\GraphQL\Support\Type as GraphQLType;
use GraphQL\Type\Definition\Type;
use App\Wine;

class WineType extends GraphQLType {

    protected $attributes = [
        'name'          => 'Wine',
        'description'   => 'Details about a wine',
        'model'         => Wine::class
    ];

    public function fields()
    {
        return [
            'id' => [
                'type'          => Type::nonNull(Type::int()),
                'description'   => 'Id of the wine',
            ],
            'name' => [
                'type'          => Type::nonNull(Type::string()),
                'description'   => 'The name of the wine',
            ],
            'description' => [
                'type'          => Type::nonNull(Type::string()),
                'description'   => 'Short description of the wine',
            ],
            'color' => [
                'type'          => Type::nonNull(Type::string()),
                'description'   => 'The color of the wine',
            ],
            'grape_variety' => [
                'type'          => Type::nonNull(Type::string()),
                'description'   => 'The grape variety of the wine',
            ],
            'country' => [
                'type'          => Type::nonNull(Type::string()),
                'description'   => 'The country of origin of the wine',
            ]
        ];
    }
}
```

This file contains the definition of the `WineType` by extending the `GraphQLType` class. You notice the definition of three protected attributes that assign the name of the type (`Wine`), a description and the model the type is associated with (the `Wine` class you defined above). The `fields()` method returns an array with the property definitions of the resources your API will return.

### Creating the Queries

Now, create a `Queries` folder inside the `app/GraphQL` folder and put here a file named `WinesQuery.php` with the following content:

```php
// app/GraphQL/Queries/WinesQuery.php
<?php
namespace App\GraphQL\Queries;

use GraphQL;
use Rebing\GraphQL\Support\Query;
use Rebing\GraphQL\Support\SelectFields;
use GraphQL\Type\Definition\Type;
use App\Wine;

class WinesQuery extends Query {

    protected $attributes = [
        'name'  => 'wines',
    ];

    public function type()
    {
        return Type::listOf(GraphQL::type('Wine'));
    }

    public function resolve($root, $args)
    {
      return Wine::all();
    }
}
```

The `WinesQuery` class defined in this file represents a query returning the list of wines from the *Wine Store*. You see that the query's name is `wines`. The `type()` method returns the type of the resource returned by the query, expressed as a list of `Wine` type items. The `resolve()` method actually returns the list of wines by using the `all()` method of the `Wine` model.

In the same way, create a second file in the `app/GraphQL/Queries` folder with the name `WineQuery.php` and with the following content:

```php
<?php
namespace App\GraphQL\Queries;

use GraphQL;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Query;
use Rebing\GraphQL\Support\SelectFields;
use App\Wine;

class WineQuery extends Query {

    protected $attributes = [
        'name'  => 'wine',
    ];

    public function type()
    {
        return GraphQL::type('Wine');
    }

    public function args()
    {
        return [
            'id'    => [
                'name' => 'id',
                'type' => Type::int(),
            	'rules' => ['required']
            ],
        ];
    }

    public function resolve($root, $args)
    {
        return Wine::findOrFail($args['id']);
    }

}
```

In this case, the `WineQuery.php` file contains the definition of the query returning a single wine identified by the `id` field returned by the `args()` method. Notice that the definition of the `id` argument specifies that the argument must be an integer and that it is mandatory. You should be able to read the meaning of the other members of the `WineQuery` class: the query's name is `wine`, the returned type is `Wine` and the returned resource is the wine identified by the `id` field.

### Registering the Schema

After you created the type and the queries, you need to register these items as the GraphQL schema for your API. So, open the `graphql.php` file inside the `config` folder and replace the current definition of `'schemas'` with the following:

```php
 // config/graphql.php
//...

	'schemas' => [
    	'default' => [
       		'query' => [
           		'wine' => App\GraphQL\Queries\WineQuery::class,
            	'wines' => App\GraphQL\Queries\WinesQuery::class,
        	]
    	],
	],
//...
```

Here you are saying that the schema of your GraphQL API consists of two queries named `wine` and `wines`, mapped to `WineQuery` and `WinesQuery` classes respectively.

Then, in the same file, replace the current definition of '`types'` with the following:

```php
// config/graphql.php
//...

	'types' => [
        'Wine' => App\GraphQL\Types\WineType::class,
    ],
//...
```

This definition maps the type  GraphQL`Wine` to the class `WineType`.

## Testing the API with GraphiQL

At this point, you are ready to use your GraphQL API. You could test the API by using [curl](https://curl.haxx.se/), [Postman](https://www.getpostman.com/) or any other HTTP client. But a specialized client can help you to better appreciate the power and the flexibility of GraphQL. The client you are going to use is [GraphiQL](https://github.com/graphql/graphiql), a browser-based client that allows you to write and interactively test GraphQL queries.

Install it in the current project as a development dependency by typing the following command:

```shell
composer require noh4ck/graphiql --dev
```

Then, publish the configuration options of the dependency by running this command:

```shell
php artisan vendor:publish --provider="Graphiql\GraphiqlServiceProvider"
```

Now, launch your development server by typing the following command:

```shell
$ php artisan serve
```

You should be able to access the GraphQL API by pointing your browser to the address http://localhost:8000/graphql-ui. In the left panel you will write your query and in the right panel you will get the result after pressing the *play* button in the top bar, as shown by the following picture:

![](images/graphiql.png)

As you can see from the picture, the GraphQL query submitted to the API is the following expression:

```
{
  wines {
  	id, name, description, color
	}
}
```

This expression specifies the name of the query (`wines`) and the fields of the resource you are interested in. The response to this request is a JSON object containing an array of wines with only the requested fields.

If you want more details about a specific wine, you can get it by using the `wine` query and passing its identifier, as shown in the following picture:

![](images/graphiql-with-arg.png)



## Integrating with Auth0

Now you have a working GraphQL API, but you want to restrict the access to only authorized clients. You can integrate your application with [Auth0](https://auth0.com) services. In this project, you will protect your GraphQL API from unauthorized access and will authorize the GraphiQL client by setting up a [machine to machine](https://auth0.com/docs/flows/concepts/m2m-flow) approach.

### Securing the API

The first step is to sign up for a [free account](https://auth0.com/signup), if you don't have one. From your [Auth0 dashboard](https://manage.auth0.com), create a machine to machine application and take note of the following data from the *Settings* tab:

- Domain
- Audience
- Client Secret

Open the `.env` file in the root folder of the project and add the following keys:

```
AUTH0_DOMAIN=<YOUR_DOMAIN>
AUTH0_AUDIENCE=<YOUR_AUDIENCE>
AUTH0_CLIENT_SECRET=<YOUR_CLIENT_SECRET>
```

Replace <YOUR_DOMAIN>, <YOUR_AUDIENCE> and <YOUR_CLIENT_SECRET> placeholders with the appropriate values taken from the Auth0 dashboard.

Then install the [Auth0 PHP SDK](https://github.com/auth0/auth0-PHP) by running the following command:

```shell
composer require auth0/auth0-php
```

Now, edit the `app/GraphQL/WinesQuery.php` file and add the `authorize()` method as shown below:

```php
<?php
namespace App\GraphQL\Queries;

use GraphQL;
use Rebing\GraphQL\Support\Query;
use Rebing\GraphQL\Support\SelectFields;
use GraphQL\Type\Definition\Type;
use App\Wine;
use Auth0\SDK\JWTVerifier;

class WinesQuery extends Query {

    protected $attributes = [
        'name'  => 'wines',
    ];

	public function authorize(array $args)
    {
    	$isAuthorized = true;
    
    	if (!empty(env( 'AUTH0_AUDIENCE' )) && !empty(env( 'AUTH0_DOMAIN' ))) {
			$verifier = new JWTVerifier([
    			'valid_audiences' => [env( 'AUTH0_AUDIENCE' )],
    			'authorized_iss' => [env( 'AUTH0_DOMAIN' )],
            	'client_secret' => [env( 'AUTH0_CLIENT_SECRET' )]
			]);
        	$request = request();
        	$token = $request->bearerToken();
            
			$decoded = $verifier->verifyAndDecode($token);
        	$isAuthorized = (boolean) $decoded;
        }
    
    	return $isAuthorized;
    }

    public function type()
    {
        return Type::listOf(GraphQL::type('Wine'));
    }

    public function resolve($root, $args)
    {
      return Wine::all();
    }
}
```

As you can see, the `authorize()` method uses the `JWTVerifier` class to verify and decode the authorization token. The `JWTVerifier` instance is built by using the parameter you set into the `.env` file. The authorization check only happens if both audience and domain values have been defined in the `.env` file. You should make the same changes also to the `app/GraphQL/WineQuery.php` file.

These changes protect the GraphQL API from unauthorized accesses. You can try to access the API by using the GraphiQL client in order to verify.

### Authorizing the GraphiQL Client

In order to enable the GraphiQL client to access the GraphQL API, you need to generate an access token from the [Auth0 dashboard](https://manage.auth0.com). Open the *Quick Start* tab of your machine to machine Auth0 application. Here you will find a few examples of code getting an access token and the correspondent response in the form of a JSON similar to the following:

```JSON
{
  "access_token": "DaEeyJ0eXAiOiJKV1QiLCJhbGciOiJSUz...",
  "token_type": "Bearer"
}
```

Copy the value of the access token and add a new key in the `.env` file in the root of the project, as in the following:

```
AUTH0_ACCESS_TOKEN=DaEeyJ0eXAiOiJKV1QiLCJhbGciOiJSUz...
```

Now, edit the `graphiql.php` file in the `config` folder and add the `Authorization` header to the existing headers, as shown in the following example:

```php
    'headers' => [
        'Accept' => 'application/json',
        'Content-Type' => 'application/json',
    	'Authorization' => 'Bearer '.env(AUTH0_ACCESS_TOKEN) 
    ]
```

This ensures that GraphiQL will provide the access token when sending the request to your GraphQL API.

## Summary

In this article, you learned how to create a GraphQL API with Laravel and how to secure it with Auth0.

You started by creating a model with its migration script to persist it in a database. You also created a seeder for the model, in order to populate the database with some initial data. Then you continued to build your API by defining a GraphQL type and two GraphQL queries. You used the GraphiQL browser-based client to interactively test the GraphQL API.

Finally, you used Auth0 to secure the GraphQL and to authorize the GraphQL client following a machine to machine approach.





