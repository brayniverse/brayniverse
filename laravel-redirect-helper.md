# Laravel Validators Package

Published on <time datetime="2016-08-29T15:42:00.000Z">29 August, 2016</time>

Recently, I created a package for Laravel called <a href="https://github.com/brayniverse/laravel-redirect-helper" target="_blank" rel="noopener">Route Redirect Helper</a>, which adds a `Route::redirect()` macro to the router so you don't have to create a closure for each simple redirect in your application.
After using my helper for a bit I noticed that I could simplify something else I do in most projects, which is create a `PagesController` to handle all _static_ pages like FAQs, Terms, License, etc.

To reduce the number of overly-simple controllers I have in my applications, I created a new package for Laravel called <a href="https://github.com/brayniverse/laravel-route-view-helper" target="_blank" rel="noopener">Route View Helper</a>, which allows me to turn the former into the latter:

## The old way

```php
Route::get('/terms', function () {
  return view('pages.terms');
});
```

## The new way

```php
Route::view('/terms', 'pages.terms');
```

The package uses an internal controller so you can still cache your routes.
