In the previous article [Using google reCaptcha with Perl ](https://dev.to/raigaurav/using-google-recaptcha-with-perl-49hd) we talked about reCAPTCHA v2. This time we will take the same example and try to use reCAPTCHA v3 instead.
So lets get started

## 1. Sign up for an API key pair
We have to register our website at - https://www.google.com/recaptcha/admin/. Provide it with the following parameters

* Label - Some name for this
* reCAPTCHA type - select v3.
* Domains - Your domain name (e.g. myname.com). For local host write "localhost". You can add multiple domain also.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/x308yxgb1mu6k2m253nx.PNG)

After submit you will get "Site Key" and "Secret Key".
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/ahh53g220ypzq3ex1zhs.PNG)

We will be using "Site key" on client side and "Secret key" on server side.

## 2. Adding the widget on client side
We will be using the same example as we have done for v2. For now, I will be putting it on Login page but one advantage of v3 is it can be used on any page and will not interfere with the user. I encourage you to check out the reCAPTCHA v3 [documentation](https://developers.google.com/recaptcha/docs/v3) for more info. 
For using the reCAPTCHA in your web page we have to include the js file.
```html
<head>
    <script src="https://www.google.com/recaptcha/api.js?onload=onloadCallback&render=<Your Site Key>"></script></script>
</head>
```
We will be adding below code in the web page. You can add these anywhere in your web page.
```html
    <input type="hidden" id="g-recaptcha-response" name="g-recaptcha-response">
    <input type="hidden" name="action" value="validate_captcha">
```
As mentioned before, v3 doesn't interfere with user, instead it will give a score between 0.0 to 1.0 depending upon the user(Just like probability). 1.0 is very likely a good interaction, 0.0 is very likely a bot. Its up to the site owner to decide what they want to do when they encounter the bot. In v3 the decision making responsibility is with the site owner. Google reCAPTCHA v3 will not block anyone irrespective of user.

Since the decision is up to site owner, you don't want to bind your action on a button click (where a user click on the button then you decide whether it is a bot or not). Instead you will be checking starting from page load and as soon as you came to know it is a bot you can take appropriate action (kicking him out of the page etc.). We will be using little bit of JavaScript for this check.

Again, we will be using [Mojo::Template](https://docs.mojolicious.org/Mojo/Template) and [Mojolicious::Plugin::TagHelpers](https://docs.mojolicious.org/Mojolicious/Plugin/TagHelpers) to generate client side code.
```perl
<html>
    <head>
        <script src="https://www.google.com/recaptcha/api.js?onload=onloadCallback&render=<Your Site Key>"></script>
    </head>
    <body>
        %= t h1 => 'Login'
        % if (flash('error')) {
            <h2 style="color:red"><%= flash('error') %></h2>
        % }
        %= form_for login => (method => 'post') => begin
            <label>username:</label> <%= text_field 'username' %>
        <br /><br />
            <label>password:</label> <%= password_field 'password' %>
        <br /><br />
            <input type="hidden" id="g-recaptcha-response" name="g-recaptcha-response">
            <input type="hidden" name="action" value="validate_captcha">
        %= submit_button 'Log in', id => 'submit'
        %= end
        <script>
            function onloadCallback() {
                grecaptcha.ready(function() {
                    grecaptcha.execute('<Your Site Key>', {action:'validate_captcha'})
                    .then(function(token) {
                        document.getElementById('g-recaptcha-response').value = token;
                        // Create an endpoint on your server to validate the token and return the score
                        fetch('/recaptchav3-verify', {
                            method: 'POST',
                            headers: {
                                'Content-Type': 'application/json',
                            },
                            body: JSON.stringify({'token': token})
                        })
                        .then(response => response.json())
                        .then(data => {
                            if (data.error === true) {
                                alert(data.description + " Bot found.");
                            }
                            else {
                                console.log('reCaptcha verification : success');
                            }
                        })
                        .catch((error) => {
                            console.error('Error:', error);
                        });
                    });
                });
            }
        </script>
    </body>
</html>
```
In summary, we are just creating a normal login form. One noted difference with the v2 version is - here we are calling endpoint `recaptchav3-verify` on page load only and based on json response, if it is normal user we will just write in console otherwise alert will appear saying its bot.

## 3. Creating Server side and validating
We will just update the code we have written for v2 
```perl
#!/usr/bin/env perl

use strict;
use warnings;
use Mojolicious::Lite;
use Mojo::UserAgent;

sub is_valid_captcha {
    my ($c) = @_;

    # https://docs.mojolicious.org/Mojo/Message#json
    my $post_params = $c->req->json;
    my $token       = $post_params->{token};
    my $captcha_url = 'https://www.google.com/recaptcha/api/siteverify';
    my $response
        = $c->ua->post(
        $captcha_url => form => {response => $token, secret => $ENV{'CAPTCHA_V3_SECRET_KEY'}})
        ->result;
    if ($response->is_success()) {
        my $out = $response->json;

        # reCAPTCHA v3 returns a score -> 1.0 is very likely a good interaction, 0.0 is very likely a bot
        if ($out->{success} && $out->{score} > 0.5) {
            return 1;
        }
        else {
            # Its a bot
            return 0;
        }
    }
    else {
        # Connection to reCAPTCHA failed
        return 0;
    }
}

# Check for authentication
helper auth => sub {
    my $c = shift;

    if (($c->param('username') eq 'admin') && ($c->param('password') eq 'admin')) {
        return 1;
    }
    else {
        return 0;
    }
};

helper ua => sub {
    my $ua = Mojo::UserAgent->new;
    $ua->transactor->name('Mozilla/5.0 (Windows NT 6.1; WOW64; rv:77.0) Gecko/20190101 Firefox/77.0');
    return $ua;
};

# Different Routes
get '/' => sub { 
    my $c = shift;
    $c->render;
} => 'index';

post '/login' => sub {
    my $c = shift;
    if ($c->auth) {
        $c->session(auth => 1);
        $c->flash(username => $c->param('username'));
        return $c->redirect_to('home');
    }
    $c->flash('error' => 'Wrong login/password');
    $c->redirect_to('index');
} => 'login';

post '/recaptchav3-verify' => sub {
    my $c = shift;
    if (is_valid_captcha($c)) {
        return $c->render(
            json => {
                      error => Mojo::JSON->false
                    }
        );
    }
    else {
        return $c->render(
            json => {
                      error => Mojo::JSON->true,
                      description => 'Captcha verification failed.'
                    }
        );
    }
};

get '/logout' => sub {
    my $c = shift;
    delete $c->session->{auth};
    $c->redirect_to('index');
} => 'logout';

under sub {
    my $c = shift;
    return 1 if ($c->session('auth') // '') eq '1';

    $c->render('denied');
};

get '/home' => sub {
    my $c = shift;
    $c->render();
} => 'home';

app->start;

__DATA__

@@ index.html.ep
<html>
    <head>
        <script src="https://www.google.com/recaptcha/api.js?onload=onloadCallback&render=<Your Site Key>"></script>
    </head>
    <body>
        ... as in Step 2
    </body>
</html>

@@ home.html.ep
%= t h1 => 'Home Page'
<h2 style="color:green">Hello <%= flash('username') %></h2>
<a href="<%= url_for('logout') %>">Logout</a>

@@ denied.html.ep
%= t h2 => 'Access Denied'
<a href="<%= url_for('index') %>">Login</a>
```
Save the file and run below command on your terminal -
```shell
morbo recaptcha_v2_verification.pl
```
One noticeable difference is addition of `recaptchav3-verify` route. We are just verifying the captcha by reading the post params and doing a post request on google api and checking the response for success. The response is a JSON object - 
```shell
{
  "success": true|false,      // whether this request was a valid reCAPTCHA token for your site
  "score": number             // the score for this request (0.0 - 1.0)
  "action": string            // the action name for this request (important to verify)
  "challenge_ts": timestamp,  // timestamp of the challenge load (ISO format yyyy-MM-dd'T'HH:mm:ssZZ)
  "hostname": string,         // the hostname of the site where the reCAPTCHA was solved
  "error-codes": [...]        // optional
}
```
We are checking the score. If it is less than **0.5** we are considering it as bot. We are using the threshold as 0.5 here but you are free to choose depending upon whether you want to be strict or lenient.
From v3 [doc](https://developers.google.com/recaptcha/docs/v3)
>
reCAPTCHA learns by seeing real traffic on your site. For this reason, scores in a staging environment or soon after implementing may differ from production. As reCAPTCHA v3 doesn't ever interrupt the user flow, you can first run reCAPTCHA without taking action and then decide on thresholds by looking at your traffic in the admin console

After that we are returning a json response to client side. We are using [Mojo::JSON](https://docs.mojolicious.org/Mojo/JSON) to convert the value into boolean `true` or `false`.
Go to browser and hit "http://localhost:3000". You can see your login page with reCAPTCHA widget on the bottom right.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/3e9fhyh3oxyuw8hymim2.PNG)
For a successful verification you can see the `home` page
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/y9rfady44d9l630purc7.PNG)
In case of bot access you can see an alert
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/74ymswm1d0l4nmj169k4.PNG)
Here I am just alerting when there is a bot, but you can choose what action you wan to do.
There is high chance where you will not see the bot alert initially even though a bot will access your page. But some frequent access in some time, v3 will learn and start alerting about it. 
You can also look at your [admin console](https://www.google.com/recaptcha/admin) over a time and look at overall traffic and suspicious requests.
Here is one such example - 
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/ayvidzouwoac9vxp48d6.PNG)
The complete example is also available at [Github](https://github.com/rai-gaurav/Data-Structures-and-Algorithms-Perl/blob/master/web_programming/recaptcha_verification/recaptcha_v3_verification.pl)
I hope this is helpful to you.

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Mojolicious logo taken from [here](https://github.com/mojolicious/mojo/blob/master/lib/Mojolicious/resources/public/mojo/logo.png)
reCAPTCHA logo taken from [here](https://www.google.com/recaptcha/about/)
