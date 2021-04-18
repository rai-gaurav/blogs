There are various module available on [metacpan](https://metacpan.org) which have help you with the automating the Firefox browser.
1. [WWW::Mechanize::Firefox](https://metacpan.org/pod/WWW::Mechanize::Firefox) - automate through the Mozrepl plugin. However,  it is retired from the Mozilla platform in November 2017. The last known compatible version is Firefox 54. (Take a look at [WWW::Mechanize::Chrome](https://metacpan.org/pod/WWW::Mechanize::Chrome) if you are looking for Chrome (or Chromium) browser. The chrome version is relevent unlike firefox).
2. [Firefox::Marionette](https://metacpan.org/pod/Firefox::Marionette) - Automate the Firefox browser with the Marionette protocol
3. [Selenium::Firefox](https://metacpan.org/pod/Selenium::Firefox) - It is part of [Selenium::Remote::Driver](https://metacpan.org/release/Selenium-Remote-Driver) and support multiple browsers.

There are others also but some of them are long deprecated and not recommended.

We will be using Firefox::Marionette for automation. To know more about Marionette protocol have a look at its [documentation](https://firefox-source-docs.mozilla.org/testing/marionette/Intro.html)

We have already created a login page during our last [exercise](https://dev.to/raigaurav/using-google-recaptcha-v3-with-perl-5506). We would be using the same example (except reCAPTCHA) for our automation.

Lets take a quick look at it.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/14221h20h6l8r5bnnyr0.PNG)
We will be filling the username and password and clicking on the 'Login' button through automated process.
Lets have a quick look at the HTML body of this page-
```html
<body>
    <h1>Login</h1>
    <form action="/login" method="post">
        <label>username:</label> <input name="username" type="text">
        <br><br>
        <label>password:</label> <input name="password" type="password">
        <br><br>
        <input id="submit" type="submit" value="Log in">
    </form>
</body>
```

Lets start with writing some code
```perl
#!/usr/bin/env perl

use strict;
use warnings;
use Firefox::Marionette;

sub main {
    my $url           = "http://localhost:3000/";
    my $firefox       = Firefox::Marionette->new(visible => 1);
    my $window_handle = $firefox->new_window(type => 'window', focus => 1, private => 1);
    $firefox->switch_to_window($window_handle);
    $firefox->go($url);
}

main();
```
Lets go through each line in `main` function. 
* We have our server running locally on 3000 port hence that url. You can update it to your own url.
* In next line we are creating the object. We have passed `visible` as `1` so that we can see the live browser otherwise it will start in headless mode. There are various [parameter](https://metacpan.org/pod/Firefox::Marionette#new) available. You can use them as per your requirements.
* Next we are saying to open a Firefox windows with different parameters. Here, we are saying open a new window in private mode and put it in foreground. Have a look at [new_window](https://metacpan.org/pod/Firefox::Marionette#new_window) for more detail.
* [switch_to_window](https://metacpan.org/pod/Firefox::Marionette#switch_to_window) switch the focus to the said window
* [go($url)](https://metacpan.org/pod/Firefox::Marionette#go) instruct the Firefox to go to the given URL

After hitting the url, we will get the login page. Now we have fill the fields with proper value ans submit the form.
If you look at the HTML we can see there are 3 `input` fields which can distinguished based on there `name` and `id` value. E.g.
* For username, find a field with `name` eq `username` and fill it
* For password, find a field with `name` eq `password` and fill it
* For login button, find by `id` having value `submit` and click it.

Lets complete the code written previously.
```perl
sub main {
    my $url           = "http://localhost:3000/";
    my $firefox       = Firefox::Marionette->new(visible => 1);
    my $window_handle = $firefox->new_window(type => 'window', focus => 1, private => 1);
    $firefox->switch_to_window($window_handle);
    $firefox->go($url);

    # For username
    my $element = $firefox->find_name('username');
    # Remove if there is something already filled there
    $firefox->clear($element);
    # Fill it with username i.e. admin
    $firefox->type($element, "admin");

    # For password
    $element = $firefox->find_name('password');
    # Remove if there is something already filled there
    $firefox->clear($element);
    # Fill it with password i.e. admin
    $firefox->type($element, "admin");

    # This is just for your eyes so that you can see what is going on
    sleep 5;

    # Find login button by the given id and click it
    $firefox->find_id('submit')->click();

    sleep 10;
}
```
I have added the comment for clarity. Lets look it in action.
Save the file and run it.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/fcy4lt1b2ks9lnc4bple.PNG)
After auto button click it will redirect to home page

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/pz0nctgxnvutu0xmt8pe.PNG)

I have also created a gif that will give more insight on the live action.

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/vm65x3isfl77mta5652a.gif)

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Firefox logo taken from [here](https://www.mozilla.org/en-US/firefox/new/)
