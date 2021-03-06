# Eloquent Repository
[![Latest Stable Version](https://poser.pugx.org/giordanolima/eloquent-repository/v/stable)](https://packagist.org/packages/giordanolima/eloquent-repository) 
[![Total Downloads](https://poser.pugx.org/giordanolima/eloquent-repository/downloads)](https://packagist.org/packages/giordanolima/eloquent-repository) 
[![License](https://poser.pugx.org/giordanolima/eloquent-repository/license)](https://packagist.org/packages/giordanolima/eloquent-repository)
[![StyleCI](https://styleci.io/repos/82729156/shield?branch=master)](https://styleci.io/repos/82729156)

[Documentação em português.](README_PT.md)

Package to assist the implementation of the Repository Pattern using Eloquent ORM.
The main feature is the flexibility and ease of use and a powerful driver for query caching.
## Installation
Installing via Composer
```bash
composer require giordanolima/eloquent-repository
```
To configure the package options, declare the Service Provider in the `config / app.php` file.
```php
'providers' => [
    ...
    GiordanoLima\EloquentRepository\RepositoryServiceProvider::class,
],
```
> If you are using version 5.5 or higher of Laravel, the Service Provider is automatically recognized by Package Discover.

To publish the configuration file:
```shell
php artisan vendor:publish
```
### Usage
To get started you need to create your repository class and extend the `BaseRepository` available in the package. You also have to set the *Model* that will be used to perform the queries.
Example:
```php
namespace App\Repositories;
use GiordanoLima\EloquentRepository\BaseRepository;
class UserRepository extends BaseRepository
{
    protected function model() {
        return \App\User::class;
    }
}
```
Now it is possible to perform queries in the same way as it is used in Elquent.
```php
namespace App\Repositories;
use GiordanoLima\EloquentRepository\BaseRepository;
class UserRepository extends BaseRepository
{
    protected function model() {
        return \App\User::class;
    }
    
    public function getAllUser(){
        return $this->all();
    }
    
    public function getByName($name) {
        return $this->where("name", $name)->get();
    }
    
    // You can create methods with partial queries
    public function filterByProfile($profile) {
        return $this->where("profile", $profile);
    }
    
    // Them you can use the partial queries into your repositories
    public function getAdmins() {
        return $this->filterByProfile("admin")->get();
    }
    public function getEditors() {
        return $this->filterByProfile("editor")->get();
    }
    
    // You can also use Eager Loading in queries
    public function getWithPosts() {
        return $this->with("posts")->get();
    }
}
```
To use the class, just inject them into the controllers.
```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Repositories\UserRepository;
class UserController extends Controller
{
    protected function index(UserRepository $repository) {
        return $repository->getAdmins();
    }
}
```
The injection can also be done in the constructor to use the repository in all methods.
```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Repositories\UserRepository;
class UserController extends Controller
{
    private $repository;
	public function __construct()(UserRepository $repository) {
        $this->repository = $repository;
    }
    
    public function index() {
        return $this->repository->getAllUsers();
    }
    
}
```
> The Eloquent/QueryBuilder methods are encapsulated as protected and are available just into the repository class. Declare your own public data access methods within the repository to access them through the controller.

#### Paginate
As default value, the `paginate` method uses, *15* records per page. This default value can be set in the configuration file to be used in all repositories.
If necessary, you can change the default value for a single repository overwriting the `perPage` property with the desired value.
```php
namespace App\Repositories;
use GiordanoLima\EloquentRepository\BaseRepository;
class UserRepository extends BaseRepository
{
    protected $perPage = 10;
    protected function model() {
        return \App\User::class;
    }
}
```
#### OrderBy
You can declare a field and a default direction to be used for all queries in a particular repository.
You can still choose other fields to sort or just skip the sorting methods.
```php
namespace App\Repositories;
use GiordanoLima\EloquentRepository\BaseRepository;
class UserRepository extends BaseRepository
{
    protected $orderBy = "created_at";
    protected $orderByDirection = "DESC";
	protected function model() {
        return \App\User::class;
    }
    
    public function getAllUser(){
        // This query will use the default ordering of the repository.
        return $this->all();
    }
    
    public function getByName($name) {
        // In this query only the declared sort will be used.
        return $this->orderBy("name")->where("name", $name)->get();
    }
    
    // É possível criar métodos com consultas parciais
    public function getWithoutOrder() {
        // In this query, no sort will be used.
        return $this->skipOrderBy()->all();
    }
    
}
```
#### GlobalScope
You can set a scope to use for all queries used in the repository.
If necessary, you can also ignore this global scope.
```php
namespace App\Repositories;
use GiordanoLima\EloquentRepository\BaseRepository;
class AdminRepository extends BaseRepository
{
    protected function model() {
        return \App\User::class;
    }
    protected function globalScope() {
        return $this->where('is_admin', true);
    }
    
    public function getAdmins() {
        // In this query the declared global scope will be included.
        return $this->all();
    }
    
    public function getAll() {
        // In this query the declared global scope will not be included.
        return $this->skipGlobalScope()->all();
    }
}
```
## Cache
The package comes with a powerful cache driver. The idea is that once the query is done, it will be cached. After the cache is done, it is possible to reduce the number of accesses to the database to zero.
All caching is performed using the cache driver configured for the application.
To use the driver, just use the trait that implements it.
```php
namespace App\Repositories;
use GiordanoLima\EloquentRepository\BaseRepository;
use GiordanoLima\EloquentRepository\CacheableRepository;
class UserRepository extends BaseRepository
{
    use CacheableRepository;
    protected function model() {
        return \App\User::class;
    }
}
```
Usage remains the same, with all cache management logic done automatically.
```php
namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Repositories\UserRepository;
class UserController extends Controller
{
    private $repository;
	public function __construct()(UserRepository $repository) {
        $this->repository = $repository;
    }
    
    public function index() {
        $users = $this->repository->getAllUsers();
        // if you call the same query later (even in other requests)
        // the query is already cached and you do not need to access the database again.
        $users = $this->repository->getAllUsers(); 
    }
    
}
```
Everytime the database data is changed, the cache is automatically cleaned to avoid outdated data.
However, if it is necessary to clear the cache, it can be performed through the `clearCache()` method.
You can also force access to the database and avoid cached data by using the `skipCache()` method.
```php
namespace App\Repositories;
use GiordanoLima\EloquentRepository\BaseRepository;
use GiordanoLima\EloquentRepository\CacheableRepository;
class UserRepository extends BaseRepository
{
    use CacheableRepository;
	protected function model() {
        return \App\User::class;
    }
    
    public function createUser($data) {
        // After inserting the data, the cache
        // is cleaned automatically.
        return $this->create($data);
    }
    
    public function addRules($user, $rules) {
        $user->rules()->attach($rules);
        $this->clearCache(); // Forcing cache cleaning.
    }
    
    public function getWithoutCache() {
        $this->skipCache()->all(); // Finding the data without using the cache.
    }
}
```