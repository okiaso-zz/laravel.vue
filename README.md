# Laravel.vue

## Installation
<pre>laravel new App --jet</pre>
### Options
- Inertia,
- Teams

## Node Packages
---
<pre>npm install && npm run dev [&& npm run watch]</pre>

## .env
---
<pre>
APP_NAME=App
APP_DOMAIN=app.test
APP_URL="http://${APP_DOMAIN}"
APP_OFFICE="office.${APP_DOMAIN}"
APP_TIMEZONE="Africa/Lagos"
ASSET_URL="/assets"
AUTH_USERNAME=username
AUTH_PREFIX=
AUTH_DOMAIN=${APP_DOMAIN}
LOG_CHANNEL=daily
DB_PREFIX=
SESSION_DOMAIN=null
</pre>

## Permission (package)
### #Permission: Installation
<pre>
composer require spatie/laravel-permission
</pre>

### #Permission: config & migration files
<pre>
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
</pre>

#### #Permission: app/Http/Kernel.php
<pre>
class Kernel extends HttpKernel {
	// ...

	protected $routeMiddleware = [
		// ...

		# Packages
		'role' => \Spatie\Permission\Middlewares\RoleMiddleware::class,
		'permission' => \Spatie\Permission\Middlewares\PermissionMiddleware::class,
		'role_or_permission' => \Spatie\Permission\Middlewares\RoleOrPermissionMiddleware::class,
	];
}
</pre>

### #Permission: app.php
<pre>
'providers' => [
	# ...
	Spatie\Permission\PermissionServiceProvider::class,
];
</pre>

### #Permission: Models\User.php
<pre>
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable {
	#...
	use HasRoles;
}
</pre>

## Packages
<pre>
composer require spatie/laravel-sluggable
composer require spatie/laravel-tags
</pre>

## Configs
---
### app.php
<pre>
'debug' => isset($_REQUEST['dev']) ? true : (bool) env('APP_DEBUG', false),
</pre>

### database.php
<pre>
'prefix' => env('DB_PREFIX', ''),
</pre>

### fortify.php
<pre>
'username' => env('AUTH_USERNAME', 'email'),
'prefix' => env('AUTH_PREFIX', ''),
'domain' => env('AUTH_DOMAIN', null),
</pre>

## Providers/FortifyServiceProvider.php
<pre>
use Illuminate\Support\Facades\Validator;

class FortifyServiceProvider extends ServiceProvider {
	public function boot() {
		# ...
		Fortify::authenticateUsing(function (Request $request) {
			return $this->login($request);
		});
	}

	public function login(Request $request) {
		if ($request->has(['username', 'password'])) {
			# Validation -------------------------------------------------------------#
			$rules = [
				'username' => 'required|string|min:3',
				'password' => 'required'
			];

			$validate = Validator::make($request->only(['username', 'password']), $rules);
			if ($validate->fails()) {
				// return back()->withErrors($validate->errors());
			}
			#-------------------------------------------------------------------------#

			if (auth()->attempt([$this->username() => $request->username, 'password' => $request->password], $request->remember)) {
				# Check if user is active
				if (!auth()->user()->active) {
					auth()->logout();
					// return session()->put('errors', ['Your account is not active!']);
					// return back()->withErrors("Your account is not active!");
				}

				# login successful!
				return auth()->user();
			}
		}
	}

	public function username() {
		if (is_numeric(request()->username)) {
			return 'phone';
		} elseif (filter_var(request()->username, FILTER_VALIDATE_EMAIL)) {
			return 'email';
		} else {
			return 'username';
		}
	}
}
</pre>

## CreateNewUser.php
<pre>
# ...
use Spatie\Permission\Models\Role;

class CreateNewUser implements CreatesNewUsers {
	# ...
	public function create(array $input) {
		$validated = Validator::make($input, [
			'first_name' => ['bail', 'required', 'string', 'min:3', 'max:32'],
			'middle_name' => ['bail', 'nullable', 'string', 'min:3', 'max:32'],
			'last_name' => ['bail', 'required', 'string', 'min:3', 'max:32'],
			'date_of_birth' => ['bail', 'nullable', 'date'],
			'gender' => ['bail', 'required', 'string', 'in:female,male'],
			'username' => ['bail', 'required', 'string', 'min:3', 'max:16', 'unique:users'],
			'phone' => ['bail', 'required', 'min:8', 'max:14', 'unique:users'],
			'email' => ['bail', 'nullable', 'email', 'max:255', 'unique:users'],
			'password' => $this->passwordRules(),
			'terms' => Jetstream::hasTermsAndPrivacyPolicyFeature() ? ['required', 'accepted'] : '',
		])->validate();

		$validated["password"] = Hash::make($validated["password"]);

		return DB::transaction(function () use ($validated) {
			return tap(User::create($validated), function (User $user) {
				$this->createTeam($user);

				$role	= Role::findOrCreate('User');
				$user->assignRole([$role->id]);
			});
		});
	}
}
</pre>

## HandleInertiaRequests.php
<pre>
use Illuminate\Support\Str;
use Inertia\Inertia;

class HandleInertiaRequests extends Middleware {
	# ...
	protected $rootView = 'guest';

	/**
	 * Sets the root template that's loaded on the first page visit.
	 *
	 * @see https://inertiajs.com/server-side-setup#root-template
	 * @param Request $request
	 * @return string
	 */
	public function rootView(Request $request) {
		$app = Str::contains($request->url(), [env('APP_OFFICE', '')]);
		if ($app) {
			return $this->rootView = 'app';
		}

		return parent::rootView($request);
	}

	/**
	 * Defines the props that are shared by default.
	 *
	 * @see https://inertiajs.com/shared-data
	 * @param  \Illuminate\Http\Request  $request
	 * @return array
	 */
	public function share(Request $request): array {
		# Default values
		$app['name'] = config('app.name');
		$app['url'] = config('app.url');
		$app['asset'] = config('app.asset_url');

		# Merge with existing values
		$app = array_merge($app, Inertia::getShared('app', []));

		if ($request->user()) {
			$user['user'] = $request->user()->only(['id']);
			$user['user']['permissions'] =  $request->user()->getAllPermissions()->pluck('name');
		}

		return array_merge(parent::share($request), $user ?? [], [
			'app' => $app,
		]);
	}

}

</pre>