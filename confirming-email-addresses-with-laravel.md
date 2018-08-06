# Confirming email addresses with Laravel
Published on <time datetime="2016-06-22T15:42:00.000Z">22 June, 2016</time>

I recently embarked on a new project which heavily relies on transactional emails; and when running a service that sends emails, it's a good idea to confirm the owner of the email address consents to receiving emails from you. Otherwise, you run the risk of unintentionally spamming people - as a user may provide a fake email address or one they do not own. Either way, it's a good idea to confirm ownership before sending any more emails.

Sending email confirmations in a Laravel application can be a simple process. In this post, I will demonstrate _one_ way of doing it.

## The workflow

1. Create a unique URL and email it to the user.</li>
2. Once the user visits this URL, destroy it so it cannot be used again.</li>
3. Be content in knowing the user owns the email address.</li>

## The unique URL
We are going to create an endpoint where part of the URL is a randomly generated token. Take the following for example:

`/email/confirm/EOpoetKBHYCMc6MHlFUFo0MA2G9trY`

The gibberish at the end is our confirmation token. We need a way to store this token and associate it with the user, so we are going to add a `confirmation_token` column to the `users` table. When this column is `null` we must assume the user has confirmed their email.

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('confirmation_token')->nullable()->unique();
    $table->index('confirmation_token');
});
```

<div class="note">
    <p><em>Don't forget to add the <code>confirmation_token</code> to the user's <code>$hidden</code> attributes, because you don't want to unintentionally display it.</em></p>
</div>

```php
protected $hidden = [
    'password', 'remember_token', 'confirmation_token',
];
```

<h2 id="token">Generating the token</h2>
<p>You can create the token however you please. A couple of options are; use <a href="https://github.com/ramsey/uuid" target="_blank" rel="noopener">the ramsey/uuid package</a>, or, the path I have opting for, Laravel's <code>str_random</code> helper, which does a good enough job for what we need.</p>
<h2 id="sending-the-email">Sending the email</h2>
<p>Now we need to send the confirmation email to the user after they register. I'm going to contain this logic within a Job because there are a few situations where we may need to send the email.</p>
<ol>
    <li>After registration</li>
    <li>When asking to resend the confirmation email</li>
    <li>After changing an email address</li>
</ol>

```php
class SendEmailConfirmation extends Job implements ShouldQueue
{
    use InteractsWithQueue, SerializesModels;

    /**
     * The User model.
     *
     * @var User
     */
    public $user;

    /**
     * Create a new job instance.
     *
     * @param User $user
    */
    public function __construct(User $user)
    {
        $this->user = $user;
    }

    /**
     * Execute the job.
     *
     * @param \Illuminate\Contracts\Mail\Mailer $mailer
    */
    public function handle(Mailer $mailer)
    {
        $user = $this->user;
        $token = str_random(30);
        $user->update(['confirmation_token' => $token]);

        $mailer->send('emails.email_confirmation', compact('user', 'token'), function ($message) use ($user) {
            $message->to($user->email, $user->name);
            $message->from('support@app.com', 'Support');
            $message->subject('Please confirm your email');
        });
    }
}
```

<h2 id="handling-the-confirmation">Handling the confirmation</h2>
<p>As I said before, we need to create an endpoint. All we need to do is find the user by their confirmation token, and if they exist, confirm their email.</p>

```php
Route::get('email/confrm/{$token}', function (Request $request, $token) {
    $user = User::where('confirmation_token', $token)->first();
    
    if ($user) {
        $user->update(['confirmation_token' => null]);
    }
    
    return view('auth.email.confirmed');
});
```

<h2 id="tying-it-all-up">Tying it all up</h2>
<p>We're almost done, all we have left to do is dispatch the <code>SendEmailConfirmation</code> job.</p>
<h3 id="after-registration">After registration</h3>

```php
class AuthController extends Controller
{
    ...
    
    protected function create(array $data)
    {
        $user = User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => bcrypt($data['password']),
        ]);
        
        dispatch(new SendEmailConfirmation($user));
        
        return $user;
    }
    
    ...
}
```

<h3 id="resending-confirmation-email">Resending confirmation email</h3>

```php
Route::group(['middleware' => 'auth'], function () {
    Route::get('email/confirm', function (Request $request) {
        $user = $request->user();
        
        if ($user->confirmedEmailAddress()) {
            return redirect('/');
        }
        
        return view('auth.email.email', compact('user'));
    });
    
    Route::post('email/confirm', function (Request $request) {
        dispatch(new SendEmailConfirmation($request->user());
        
        return view('auth.email.sent');
    });
});
```
