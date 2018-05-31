<header role="banner">
            <a href="/">Brayniverse <span class="sr-only">Home</span></a>
        </header>
        <main id="main">
            <h1>Monitoring Failed Login Attempts With Laravel</h1>
            <p>Published on <time datetime="2017-01-18">08 January, 2017</time></p>
            <p><a href="https://laravel.com/docs/5.3/authentication#login-throttling" target="_blank" rel="noopener">Laravel's login throttling</a> is a valuable combatant against dictionary attacks.</p>
            <blockquote>
                <p>the user will not be able to login for one minute if they fail to provide the correct credentials after several attempts.</p>
            </blockquote>
            <p>Login throttling hinders and impedes potential dictionary attacks, however, it doesn't prevent them entirely, an attacker could wait for one minute and try again. In order to identify malicious activity and take action to stop it, you can record failed login attempts; this doesn't completely thwart hackers either, but it can be the groundwork for some further security tools like:</p>
            <ul>
                <li>Letting the user know somebody tried logging into their account.</li>
                <li>Blocking the IP address until you can investigate.</li>
                <li>Locking the users account until you can verify ownership.</li>
            </ul>
            <p>Laravel <a href="https://github.com/laravel/framework/pull/13761" target="_blank" rel="noopener">introduced</a> the <code>Illuminate\Auth\Events\Failed</code> event in 5.2, which is fired when the guest provides invalid credentials when logging in.</p>
            <p><code>Failed</code> has two public properties, <code>$user</code> and <code>$credentials</code>. If the provided email address exists in the database, the <code>$user</code> property is an instance of the <code>User</code> model associated with that address, otherwise, the <code>$user</code> property is <code>null</code>. You can guess what <code>$credentials</code> is.</p>
            <p>Whenever the <code>Failed</code> event is fired an event listener will record the failed attempt. Then we will have a log of all failed login attempts that we can use to improve security.</p>
            <p>To begin with let's make a new <code>LoginAttempt</code> model and a migration file to go with it.</p>
            <pre>
                <code>php artisan make:model FailedLoginAttempt --migration</code>
            </pre>
            <p>In the migration file add the following.</p>
            <pre>
                <code>
                    Schema::create('failed_login_attempts', function (Blueprint $table) {<br />
                    &#9;$table-&gt;increments('id');<br />
                    &#9;$table-&gt;unsignedInteger('user_id')-&gt;nullable();<br />
                    &#9;$table-&gt;string('email_address');<br />
                    &#9;$table-&gt;ipAddress('ip_address');<br />
                    &#9;$table-&gt;timestamps();<br />
                    <br />
                    &#9;$table-&gt;foreign('user_id')-&gt;references('id')-&gt;on('users');<br />
                    });
                </code>
            </pre>
            <h2 id="user_id"><code>user_id</code></h2>
            <p>The user's ID helps us to identify an attack on a particular account.</p>
            <h2 id="email_address"><code>email_address</code></h2>
            <p>The email address allows us to determine whether the guest has misspelt their email, which could explain the failed login attempts.</p>
            <h2 id="ip_address"><code>ip_address</code></h2>
            <p>We should not assume that an intruder will consecutively target the same email address. Therefore, we also record the IP address as well to identify when the same source is attempting several addresses.</p>
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
            <p>Now create a <code>RecordFailedLoginAttempt</code> event listener and register it in the <code>EventServiceProvider</code>.</p>
            <pre>
                <code>
                    php artisan make:listener RecordFailedLoginAttempt --event=Illuminate\\Auth\\Events\\Failed
                </code>
            </pre>
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
            <p>Finally, to test that the feature is working.</p>
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
        </main>