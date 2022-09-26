# Laravel Coding Standards
Hello there, An awesome laravel community. We all have our own way of coding but for better readabilty and usage, I've added some common best practices here.

If you have any idea or suggestion then feel free to open a PR.



## Naming conventions
| What  |How   |Good   |Bad   |
| ------------ | ------------ | ------------ | ------------ |
| controllers  | Singular  | ArticleController  |ArticlesController   |
| route  | Plural  |articles/1   |article/1   |
|named route   | snake_case with dot notation  |users.show_active   | users.show-active, show-active-users  |
|model  |Singular   |User   |Users   |
|hasOne or belongsTo relationship (In Model)   |Singular   |articleComment   | articleComments, article_comment  |
|All other relationships   |Plural   |article_comments   |  article_comment, articleComments |
|pivot table  |singular model names in alphabetical order  |article_user (Keep the same table name in database)   | user_article, articles_users  |
|Table column -MySQL   |snake_case without model name   |meta_title   |metaTitle; article_meta_title  |
| Foreign key  |singular model name with _id suffix   |article_id   |ArticleId, id_article, articles_id  |
| Primary key  | -  | id  |custom_id   |
| Method  |Small CamelCase   |getOrders   |get_Orders, GetOrders   |
|Helper functions   |snake_case   | abort_if  | abortIf  |
|Model property  |snake_case |$user->profile_image |$model->profileImage |
| variable  |camelCase   |$anyOtherVariable   |$any_other_variable   |
|Collection   |descriptive, plural   |$activeUsers = User::active()->get()   | $active, $data  |
| objects  |  descriptive, singular |$activeUser = User::active()->first()   | $users, $obj  |
| Config and language files index  |snake_case   |articles_enabled   |articleEnabled,articles-enabled  |
| views  |  kebab-case |show-filtered.blade.php   | showFiltered.blade.php, show_filtered.blade.php  |
|configuration   | kebab-case  |google-calendar.php   |googleCalendar.php,google_calendar.php   |
|Contract (interface)   | adjective or noun  |Authenticatable   |AuthenticationInterface, IAuthentication  |
| trait  | adjective  | Notifiable  | NotificationTrait  |

## Single responsibility principle
A class and a method should have only one responsibility.

**Bad**
```php
public function getFullNameAttribute()
    {
        if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
            return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
        } else {
            return $this->first_name[0] . '. ' . $this->last_name;
        }
    }
```
**Good**
```php
   public function getFullNameAttribute()
    {
        return $this->isVerifiedClient() ? $this->getFullNameLong() :      $this->getFullNameShort();
    }

    public function isVerifiedClient()
    {
        return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
    }

    public function getFullNameLong()
    {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    }

    public function getFullNameShort()
    {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
```
## 2. Fat models, skinny controllers [We can implement MVCS Model]
Put all DB related logic into Eloquent models or into Repository classes if you're using Query Builder or raw SQL queries.

**Bad**
```php
  public function index()
    {
        $clients = Client::verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();

        return view('index', ['clients' => $clients]);
    }

```
**Good**
```php
   public function index()
    {
        return view('index', ['clients' => $this->client->getWithNewOrders()]);
    }

    class Client extends Model
    {
        public function getWithNewOrders()
        {
            return $this->verified()
                ->with(['orders' => function ($q) {
                    $q->where('created_at', '>', Carbon::today()->subWeek());
                }])
                ->get();
        }
    }
```
## 3. Validation
Move validation from controllers to Request classes.

**Bad**
```php
  public function store(Request $request)
    {
        $request->validate([
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ]);

        ....
    }
```
**Good**
```php
   public function store(PostRequest $request)
    {
        ....
    }

    class PostRequest extends Request
    {
        public function rules()
        {
            return [
                'title' => 'required|unique:posts|max:255',
                'body' => 'required',
                'publish_at' => 'nullable|date',
            ];
        }
    }
```
### Custom validation rule
`php artisan make:rule BirthYearRule`
```php
class BirthYearRule implements Rule
{
  public function passes($attribute, $value)
    {
        return $value >= 1990 && $value <= date('Y');
    }

    /**
     * Get the validation error message.
     *
     * @return string
     */
    public function message()
    {
        return 'The :attribute must be between 1990 to '.date('Y').'.';
    }
}
```
**Usage**
```php
class RegisterRequest extends Request
{
public function rules()
    {
        return [
            'name' => 'required',
            'birth_year' => [
                'required',
                new BirthYearRule()
            ]
        ];
    }
}
```
## 4. Business logic should be in service class
A controller must have only one responsibility, so move business logic from controllers to service classes. It will help us to use the same function for web and API.

**Bad**
```php
public function store(Request $request)
    {
        if ($request->hasFile('image')) {
            $path= $request->file('image')->store(‘image’,’public’);
        }
        //here the public is storage disk
        ....
    }
```
**Good**

create a service class in app/Services/ArticleService.php
```php
public function store(Request $request)
    {
        $this->articleService->handleUploadedImage($request->file('image'));

        ....
    }

    class ArticleService
    {
        public function handleUploadedImage($image)
        {
            if (!is_null($image)) {
                $path= $image->store(‘image’,’public’);
            }
        }
    }
```
## 5. Don't repeat yourself (DRY)
Reuse code when you can. SRP is helping you to avoid duplication. So, reuse Blade templates, use Eloquent scopes etc.

**Bad**
```php
public function getActive()
    {
        return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
    }

    public function getArticles()
    {
        return $this->whereHas('user', function ($q) {
                $q->where('verified', 1)->whereNotNull('deleted_at');
            })->get();
    }
```
**Good**
```php
public function scopeActive($q)
    {
        return $q->where('verified', 1)->whereNotNull('deleted_at');
    }

    public function getActive()
    {
        return $this->active()->get();
    }

    public function getArticles()
    {
        return $this->whereHas('user', function ($q) {
                $q->active();
            })->get();
    }
```
## 6. Prefer to use Eloquent over using Query Builder and raw SQL queries. Prefer collections over arrays
Eloquently allows you to write readable and maintainable code. Also, Eloquent has great built-in tools like soft deletes, events, scopes etc.

**Bad**
```php
SELECT *
    FROM `articles`
    WHERE EXISTS (SELECT *
                  FROM `users`
                  WHERE `articles`.`user_id` = `users`.`id`
                  AND EXISTS (SELECT *
                              FROM `profiles`
                              WHERE `profiles`.`user_id` = `users`.`id`)
                  AND `users`.`deleted_at` IS NULL)
    AND `verified` = '1'
    AND `active` = '1'
    ORDER BY `created_at` DESC
```
**Good**
```php
Article::has('user.profile')->verified()->latest()->get();
```
## 7. Do not execute queries in Blade templates and use eager loading(N+1 problem)
From Laravel 8.43 onwards prevent lazy loading:
```php
Path : /provides/AppServiceProvider

     /**
      * Bootstrap any application services.
      *
      * @return void
      */
     public function boot()
     {
       Model::preventLazyLoading(! app()->isProduction());
     }
```
**Bad (for 100 users, 101 DB queries will be executed):**
```php
  @foreach (User::all() as $user)
      $user->profile->image;
  @endforeach
```
**Good**
```php
$users = User::with('profile')->get();
    ...

    @foreach ($users as $user)

    @endforeach
```
## 8. Comment your code, but prefer descriptive method and variable names over comments
**Bad**
```php
if (count((array) $builder->getQuery()->joins) > 0)
```
**Good**
```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```
**Better**
```php
if ($this->hasJoins())
```
### Comment your function with author and date of modified for your team
```php
/**
* @param  Request  $request
* @return \Symfony\Component\HttpFoundation\BinaryFileResponse
* @author Dev A
* @updatedBy Dev B
*/
public function export(Request $request)
{
      return Excel::download(new OrdersExport(),'orders.xls');
}
```
## 9. Do not put JS and CSS in Blade templates and do not put any HTML in PHP classes
**Bad**
```javascript
let article = ``;
```
**Good**
```html
<input id="article" type="hidden" value="">
```
**Usage in JS file**
```javascript
let article = $('#article').val();
```
### Instead of using html in controller use view rendering
**Bad**
```php
//Long HTML tags in the php like below

return "<span>". {{$user->name}}." </span>"; 
```
**Good**
```php
return View::make('users.name', ['user' => $user])->render();
```
## 10. Use config and language files, constants instead of text in the code
**Bad**
```php
public function isAdmin()
    {
        return $this->role === 'admin';
    }
```
**Good**
```php
public function isAdmin()
    {
        return $this->role === User::TYPE_ADMIN;
    }
```
## 11. Do not get data from the ".env" file directly
Pass the data to config files instead and then use the  config() helper function to use the data in an application.

**Bad**
```php
$stripeKey = env('STRIPE_KEY');
$stripeSecret= env('STRIPE_SECRET');

```
**Good**
```php
    // config/services.php
   'stripe'=>[
          'key' => env('STRIPE_KEY'),
	  'secret' => env('STRIPE_SECRET'),
     ];

    // Use the data
    $stripeKey = config('services.stripe.key');
```
also use .env.example file as reference for the other developer so he can get to know about the third party keys and secret.

We can make our custom config file for the specific feature like for ex. we have used a OpenTok so we can make config file.

```php
//config/opentok.php
//config/params.php
```
## 12. Use config filesystem for the storage of media
Store your local media files (profile, product photo etc...) to the storage folder instead of public folder:

Execute `php artisan storage:link`  for creating symlink in laravel  (public/storage)

```php
//   config/filesystem.php
'disks' => [
     'public' => [
       'driver' => 'local',
       'root' => storage_path('app/public/uploads'),
       'url' => env('APP_URL') . '/storage/uploads/',
       'visibility' => 'public',
   ],
```
**Store Image**
```php
 $path= Storage::disk('public')->put('avatars/1', $request->image);
```
**Usage**
```php
@if (Storage::disk('s3')->exists($user->avatar))
    <img src={{Storage::disk('s3')->get($user->avatar)}} title="avatar"></img>
@endif
```
## 13.Route service provider for web / api.
```php
//app/Providers/RouteServiceProvider.php
public function boot()
{
    $this->configureRateLimiting();

    $this->routes(function () {
        Route::prefix('api') // to make it dynamic
            ->middleware('api')
            ->namespace($this->namespace)
            ->group(base_path('routes/api.php'));

        Route::middleware('web')
            ->namespace($this->namespace)
            ->group(base_path('routes/web.php'));
    });
}
```

**Also create different php files for the different role and include in web.php or api.php for better readabilty**
```php
    `routes/admin.php`
```
## 14. User SQL to eloquent convertor for the raw and complex SQL
[Raw SQL to eloquent convertor online](https://jjlabajo.github.io/SQLtoEloquent/ "Raw SQL to eloquent convertor online")

## 15. Encryption in laravel
Laravel uses AES-256 and AES-128 encrypter, which uses Open SSL for encryption. All the values included in Laravel are signed using the protocol Message Authentication Code so that the underlying value cannot be tampered with once it is encrypted.

`php artisan key:generate`

**Encryption**
```php
public function storeSecret(Request $request, $id) {
      $user = User::findOrFail($id);
      $user->fill([
         'secret' => encrypt($request->secret)
      ])->save();
   }
```
**Decryption**
```php
$decrypted = decrypt($encryptedValue);
```
**Note: Please note that do not change the APP_KEY in the production. Otherwise you can’t decrypt the values.**

### Hashing
for ex : The hashed passwords are stored using make method. This method allows managing the work factor of the bcrypt hashing algorithm, which is popularly used in Laravel.
```php
$request->user()->fill([
         'password' => Hash::make($request->newPassword) // Hashing
      ])->save();

....

if (Hash::check('plain-text', $hashedPassword)) {
// The passwords match...
}
```
## 16. Use softdeletes() as per the requirement
Softdeletes just updates the deleted_at column in the database instead of deleting records completely. kind of archieve.
```php
Schema::create('posts', function(Blueprint $table)
    {
        $table->increments('id');
        // more fields
        $table->softDeletes(); //<--This will add a deleted_at field
        $table->timeStamps();
    });

//Post.php
class Post extends Model {

    use SoftDeletes;

    protected $table = 'posts';

    // ...
}
//PostController.php

public function destroy(Post $post)
{
     $post->delete();
}
```
## 17. User Seeder and faker to dump data in the database
The factory and seeder are used to test performace of the application with good amount of dummy data.
```php
class StudentFactory extends Factory
{
    /**
     * The name of the factory's corresponding model.
     * @var string
     */
    protected $model = Student::class;

    /**
     * Define the model's default state.
     * @return array
     */
    public function definition()
    {
        return [
            "name" => $this->faker->name(),
            "email" => $this->faker->safeEmail,
            "mobile" => $this->faker->phoneNumber,
            "age" => $this->faker->numberBetween(25, 45),
            "gender" => $this->faker->randomElement([
                "male",
                "female",
                "others"
            ]),
            "address_info" => $this->faker->address
        ];
    }
}
```
**To generate 1000 students**
```php
class DatabaseSeeder extends Seeder
{       public function run()
    {
         \App\Models\Student::factory(1000)->create();
    }
}
```
## 18.Laravel blade components

Please refer the [components](https://laravel.com/docs/9.x/blade#components "documentation") for more information.

for example here we are creating a simple alert component to show success/error message.

`php artisan make:component Alert`

This command will create two files:


    app\View\Components\Alert.php
    resources\views\components\alert.blade.php

alert.php  file:
```php
<?php

namespace App\View\Components;

use Illuminate\View\Component;

class Alert extends Component
{
    public $type;
    public $message;

    public function __construct($type, $message)
    {
        $this->type = $type;
        $this->message = $message;
    }

    public function render()
    {
        return view('components.alert');
    }
}
```
and in the alert.blade.php:

    <div class="alert alert-{{ $type }}">
        {{ $message }}
    </div>

and here we go, we can call the component as below in our blade file
```php
<x-alert type="error" :message="$message" class="mt-4"/>
```
##19. env file

Don't forget to change APP_URL in your .env file from` http://localhost` to the real URL, cause it will be the basis for any links in your email notifications and elsewhere.

    APP_NAME=Laravel
    APP_ENV=local
    APP_KEY=base64:9PHz3TL5C4YrdV6Gg/Xkkmx9btaE93j7rQTUZWm2MqU=
    APP_DEBUG=true
    APP_URL=http://localhost/project-name/public

##20. Composer dependency commands

- ` composer install` to install all the dependencies as specified in the **composer.lock** file.
-  `composer update` will update your depencencies as they are specified in c**omposer.json** file.
-  `composer update --dry-run`  to see all the dependencies being updated just like you would see when you run the composer update command. The only difference is it’s just a visual cue of which package will get updated to which version **without the actual dependency updation**.
-  `composer outdated` shows you the list of outdated packages.
- `composer show -l` returns the complete list of packages. packages in need of update are colored **red**. the (still) up to date ones are colored **green**.

##21. Steps to make your Laravel app ready for production
1 . Apply changes to your `.env` file:

`APP_ENV=production`
`APP_DEBUG=false`

2 . Make sure that you are optimizing Composer's class autoloader map
- `composer dump-autoload --optimize`
- or along install: `composer install --optimize-autoloader --no-dev`

3 . Cache the framework bootstrap files:
- ` php artisan optimize`

4 .  Compile all of the application's Blade templates:
-  `php artisan view:cache`

5 . (Optional) Compiling assets
-  `npm run production`  You can also run this command in your local machine and directly add those minifield files to your master/main branch.

6 . (Optional) Start Laravel task scheduler by adding the following Cron entry
-  `* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1`

7 . (Optional) Install, config and start the Supervisor ([docs](https://laravel.com/docs/master/queues#supervisor-configuration "docs")):

8 . (Optional) Create a symbolic link to `public/storage` from `storage/app/public`
-  `php artisan storage:link`

for more information visit the [laravel deployment documentation](https://laravel.com/docs/master/deployment "laravel deployment documentation").