[Brayniverse <span class="sr-only">Home</span>](/)
# Monitoring Failed Login Attempts With Laravel

Published on <time datetime="2017-01-18">08 January, 2017</time>

<a href="https://laravel.com/docs/5.3/authentication#login-throttling" target="_blank" rel="noopener">Laravel's login throttling</a> is a valuable combatant against dictionary attacks.
> the user will not be able to login for one minute if they fail to provide the correct credentials after several attempts.

Login throttling hinders and impedes potential dictionary attacks, however, it doesn't prevent them entirely, an attacker could wait for one minute and try again. In order to identify malicious activity and take action to stop it, you can record failed login attempts; this doesn't completely thwart hackers either, but it can be the groundwork for some further security tools like:

- Letting the user know somebody tried logging into their account.
- Blocking the IP address until you can investigate.
- Locking the users account until you can verify ownership.

Laravel <a href="https://github.com/laravel/framework/pull/13761" target="_blank" rel="noopener">introduced</a> the <code>Illuminate\Auth\Events\Failed</code> event in 5.2, which is fired when the guest provides invalid credentials when logging in.

`Failed` has two public properties, `$user` and `$credentials`. If the provided email address exists in the database, the `$user` property is an instance of the `User` model associated with that address, otherwise, the `$user` property is `null`. You can guess what `$credentials` is.

Whenever the `Failed` event is fired an event listener will record the failed attempt. Then we will have a log of all failed login attempts that we can use to improve security.

To begin with let's make a new <code>LoginAttempt</code> model and a migration file to go with it.

```php artisan make:model FailedLoginAttempt --migration```

In the migration file add the following.

```php
Schema::create('failed_login_attempts', function (Blueprint $table) {
    $table->incremenets('id');
    $table->unsignedInteger('user_id')->nullable();
    $table->string('email_address');
    $table->ipAddress('ip_address');
    $table->timestamps();
    
    $table->foreign('user_id')->references('id')->on('users');
});
```

## `user_id`

The user's ID helps us to identify an attack on a particular account.

## `email_address`

The email address allows us to determine whether the guest has misspelt their email, which could explain the failed login attempts.

## `ip_address`

We should not assume that an intruder will consecutively target the same email address. Therefore, we also record the IP address as well to identify when the same source is attempting several addresses.
<pre>
    <code>
        class FailedLoginAttempt extends Model<br />
        {<br />
        &#9;protected $fillable = [<br />
        &#9;&#9;'user_id', 'email', 'ip_address',<br />
        &#9;];<br />
        <br />
        &#9;public static function record($user = null, $email, $ip)<br />
        &#9;{<br />
        &#9;&#9;return static::create([<br />
        &#9;&#9;&#9;'user_id' =&gt; is_null($user) ? null : $user-&gt;id,<br />
        &#9;&#9;&#9;'email_address' =&gt; $email,<br />
        &#9;&#9;&#9;'ip_address' =&gt; $ip<br />
        &#9;&#9;]);<br />
        &#9;}<br />
        }
    </code>
</pre>

Now create a `RecordFailedLoginAttempt` event listener and register it in the `EventServiceProvider`.

```php artisan make:listener RecordFailedLoginAttempt --event=Illuminate\\Auth\\Events\\Failed```

<pre>
    <code>
        class RecordFailedLoginAttempt<br />
        {<br />
        &#9;public function handle(Failed $event)<br />
        &#9;{<br />
        &#9;&#9;FailedLoginAttempt::record(<br />
        &#9;&#9;&#9;$event-&gt;user,<br />
        &#9;&#9;&#9;$event-&gt;credentails['email'],<br />
        &#9;&#9;&#9;request()-&gt;ip()<br />
        &#9;&#9;);<br />
        &#9;}<br />
        }
    </code>
</pre>
<pre>
    <code>
        class EventServiceProvider extends ServiceProvider<br />
        {<br />
        &#9;...<br />
        &#9;protected $listen = [<br />
        &#9;&#9;\\Illuminate\\Auth\\Events\\Failed::class =&gt; [<br />
        &#9;&#9;&#9;\\App\\Listeners\\RecordFailedLoginAttempt::class,<br />
        &#9;&#9;]<br />
        &#9;];<br />
        <br />
        &#9;...<br />
        }
    </code>
</pre>

Finally, to test that the feature is working.

<pre>
    <code>
        class FailedLoginTest extends TestCase<br />
        {<br />
        &#9;use DatabaseTransactions;<br />
        &#9;/** @test */<br />
        &#9;public function failed_login_attempts_are_recorded()<br />
        &#9;{<br />
        &#9;&#9;$this-&gt;visit('/login')<br />
        &#9;&#9;&#9;-&gt;type('john@doe.com', 'email')<br />
        &#9;&#9;&#9;-&gt;type('incorrect password', 'password')<br />
        &#9;&#9;&#9;-&gt;press('Log In');<br />
        <br />
        &#9;&#9;$this-&gt;seeInDatabase('failed_login_attempts', [<br />
        &#9;&#9;&#9;'user_id' =&gt; null,<br />
        &#9;&#9;&#9;'email_address' =&gt; 'john@doe.com',<br />
        &#9;&#9;]);<br />
        &#9;}<br />
        <br />
        &#9;/** @test */<br />
        &#9;public function existing_user_is_recorded()<br />
        &#9;{<br />
        &#9;&#9;$user = factory(User::class)-&gt;create();<br />
        <br />
        &#9;&#9;$this-&gt;visit('/login')<br />
        &#9;&#9;&#9;-&gt;type($user-&gt;email, 'email')<br />
        &#9;&#9;&#9;-&gt;type('incorrect password', 'password')<br />
        &#9;&#9;&#9;-&gt;press('Log In');<br />
        <br />
        &#9;&#9;$this-&gt;seeInDatabase('failed_login_attempts', [<br />
        &#9;&#9;&#9;'user_id' =&gt; $user-&gt;id,<br />
        &#9;&#9;&#9;'email_address' =&gt; $user-&gt;email,<br />
        &#9;&#9;]);<br />
        &#9;}<br />
        }
    </code>
</pre>
