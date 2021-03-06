# Guard: Joyful Authentication

My favorite new feature for Symfony 2.8 is Guard. It makes creating custom and crazy
authentication systems really really easy. I'm a bit biased: Guard was my creation,
inspired by a lot of people and projects.

## The Weirdest Login Form Ever

Later, I'll do some in-depth screencasts about Guard, but I want to give you a taste
of what's possible. In this project, I have a tradition login system:

[[[ code('67f98fbfec') ]]]

The `/login` controller renders a login template and this template builds a form
that POSTs right back to that same `loginAction`:

[[[ code('06b3bd593d') ]]]

It has a username field, a password field and then a couple of extra fields. One of
them is "the answer to life the universe and everything", which of course must be
set to 42. The user also needs to check this "agreement" checkbox.

So how can we make a login system that checks *four* fields instead of just the usual
username and password? Hello Guard.

Before coding, start the built-in web server, which is now `bin/console server:run`:

```bash
bin/console server:run
```

Try out `/login`.

## Creating the Authenticator Class

To use guard, we need a new class. I'll create a new directory called `Security`,
but that's not important. Call the class `WeirdFormAuthenticator`.Next, make
this class implement `GuardAuthenticatorInterface` or extend the slightly easier
`AbstractGuardAuthenticator`:

[[[ code('37e8e4c775') ]]]

In PhpStorm, I'll open the [generate menu](http://knpuniversity.com/screencast/phpstorm/doctrine)
and select "Implement Methods". Select *all* of them, including `start()`, which is
hiding at the bottom. Also move the `start()` method to the bottom of the class:
it'll fit more thematically down there:

[[[ code('e248d645b5') ]]]

Your job is simple: fill in each of these methods. 

## getCredentials()

Once we hook it up, the `getCredentials()` method will be called on the authenticator
on *every single request*. This is your opportunity to collect the username, password,
API token or whatever "credentials" the user is sending in the request.

For this system, we don't need to bother looking for the credentials unless the user
is submitting the login form. Add an if statement: if `$request->getPathInfo()` - that's
the current, cleaned URL - `!= /login` or `!$request->isMethod('POST')`, then return
`null`:

[[[ code('4cfff2117f') ]]]

When you return `null` from `getCredentials()`, no other methods will be called on
the authenticator.

Below that, return an array that contains any credential information we need. Add
a `username` key set to `$request->request->get('_username')`: that's the name of
the field in the form:

[[[ code('9cff45d30f') ]]]

Repeat that for password and `answer`. In the form, answer's
name is `the_answer` and the last is named `terms`. Finish the array by fetching
`the_answer` and `terms`:

[[[ code('e60348b4ab') ]]]

The keys on the right are obviously the names of the submitted form fields. The keys
on the left can be anything: you'll see how to use them soon.

## getUser()

If `getCredentials()` returns a non-null value, then `getUser()` is called next.
I have a really simple `User` entity that has basically just one field: `username`:

[[[ code('3553a068d4') ]]]

I'm not even storing a password. Whatever your situation, just make sure you have
a `User` class that implements `UserInterface` *and* a "user provider" for it that's
configured in `security.yml`:

[[[ code('b1ad015509') ]]]

See the `$credentials` variable? That's equal to whatever you returned from `getCredentials()`.
Set a new `$username` variable by grabbing the `username` key from it:

[[[ code('bfc54673a9') ]]]

Let's get crazy and say: "Hey, if you have a username that starts with an `@` symbol,
that's not okay and you can't login". To fail authentication, return null:

[[[ code('08bcb04c02') ]]]

Now, query for the User! We need the entity manager, so add a `private $em` property
and generate the constructor. Type-hint it with `EntityManager`:

[[[ code('e7adb0e11e') ]]]

Back in `getUser()`, this is easy: return `$this->em->getRepository('AppBundle:User')->findOneBy()`
with `username` equals `$username`:

[[[ code('2cb18dc889') ]]]

The job of `getUser()` is to return a User object or null to fail authentication.
It's perfect.

## checkCredentials()

If `getUser()` *does* return a User, `checkCredentials()` is called. This is where
you check if the password is valid or anything else to determine that the authentication
request is legitimate.

And surprise! The `$credentials` argument is once again what you returned from `getCredentials()`.

Let's check a few things. First, the user doesn't have a password stored in the database.
In my system, the password for everybody is the same: `symfony3`. If it doesn't equal
`symfony3`, return `null` and authentication will fail:

[[[ code('7e25b199e0') ]]]

Second, if the `answer` does not equal 42, return `null`. And finally, if the `terms`
weren't checked, you guessed it, return `null`. Authentication will fail unless `checkCredentials()`
exactly returns `true`. So return that at the bottom:

[[[ code('1aa4542af8') ]]]

## Handling Authentication Success and Failure

That's it! The last few methods handle what happens on authentication failure and
success. Both should return a `Response` object, or null to do nothing and let the
request continue.

### onAuthenticationFailure()

The `$exception` passed to `onAuthenticationFailure` holds details about *what* went
wrong:

[[[ code('a3f7494b36') ]]]

Was the user not found? Were the credentials wrong? Store this in the session
so we can show it to the user. For the key, use the constant: `Security::AUTHENTICATION_ERROR`:

[[[ code('b1a368d2cd') ]]]

This is *exactly* what the normal form login system does: it adds an error to this
same key on the session. 

In the controller, the `security.authentication.utils` service reads this key from the session
when you call `getLastAuthenticationError()`:

[[[ code('f190a111f8') ]]]

Other than storing the error, what we *really* want to do when authentication fails
is redirect back to the login page. To do this, add the `Router` as an argument to
the constructor and use that to set a new property. I'll do that with a
[shortcut](http://knpuniversity.com/screencast/phpstorm/service-shortcuts#generating-constructor-properties):

[[[ code('9b2abd6d4e') ]]]

Now it's simple: `$url = $this->router->generate()` and pass it `login` - that's the
route name to my login page. Then, return a `new RedirectResponse`:

[[[ code('c2b4569933') ]]]

### onAuthenticationSuccess()

When authentication works, keep it simple and redirect back to the homepage. In a
real app, you'll want to redirect them back to the previous page:

[[[ code('74e2a6429d') ]]]

There's a base class called `AbstractFormLoginAuthenticator` that can help with this.

### start() - Inviting the User to Authentication

The `start()` method is called when the user tries to access a page that requires
login as an anonymous user. For this situation, redirect them to the login page:

[[[ code('2609556690') ]]]

### supportsRememberMe()

Finally, if you want to be able to support remember me functionality, return true
from `supportsRememberMe()`:

[[[ code('5bead0c372') ]]]

You'll still need to configure the `remember_me` key in the firewall.

## Configuring the Authenticator

That's it! Now we need to tell Symfony about the authentication with two steps. The
first shouldn't surprise you: register this as a service. In `services.yml` create
a new service - how about `weird_authenticator`. The class is `WeirdFormLoginAuthenticator`
and there are two arguments: the entity manager and the router:

[[[ code('3c9f54bf74') ]]]

Finally, hook this up in security.yml. Under your firewall, add a `guard` key and
an `authenticators` key below that. Add one authenticator - `weird_authenticator` -
the service name:

[[[ code('0afece19a6') ]]]

Now, `getCredentials()` will be called on every request. If it returns something
other than `null`, `getUser()` will be called. If this returns a `User`, then
`checkCredentials()` will be called. And if this returns `true`, authentication
will pass.

## Test the Login

Try it out! With a blank form, it says `Username could not be found`. It *is* looking
for the credentials on the request, but it fails on the `getUser()` method. I do
have one user in the database: `weaverryan`.  Put `symfony3` for the password but
set the answer to 50. This fails with "Invalid Credential". Finally, fill in everything
correctly. And it works! The web debug toolbar congratulates us: logged in as weaverryan.

## Customize the Error Messages

We saw two different error messages, depending on whether authentication failed in
`getUser()` or `checkCredentials()`. But you can *completely* control these messages
by throwing a `CustomUserMessageAuthenticationException()`. I know, it has a long name,
but it's job is clear: pass it the message you want want to show the user, like:

[[[ code('0ca2c8d66a') ]]]

Below if the `answer` is wrong, do the same thing: `throw new CustomUserMessageAuthenticationException()`
with:

[[[ code('9376281293') ]]]

And in case the terms checkbox isn't checked, throw a new `CustomUserMessageAuthenticationException`
and threaten them to agree to our terms!

[[[ code('687e1802b9') ]]]

Try it out with `@weaverryan`. There's the first message! Try 50 for the answer:
there's the second message. And if you fill in everything right but have no checkbox,
you see the final message.

That's Guard authentication: create one class and do whatever the heck that you want.
I showed a login form, but it's even easier to use for API authentication. Basically,
authentication is not hard anymore. Invent whatever insane system you want and give
the user exactly the message you want. If you want to return `JSON` on authentication
failure and success instead of redirecting, awesome, do it. 
