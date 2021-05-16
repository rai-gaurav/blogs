In [previous](https://dev.to/raigaurav/creating-rest-apis-with-perl-mojolicious-and-openapi-1bng) article we have created few REST API's.
Now lets try to deploy it to production. It will be a long article so brace your self.
For production deployment we will be using few things -
1. [Apache2 Server](https://httpd.apache.org/)
2. [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/)
3. [Docker](https://www.docker.com/)

[Mojolicious](https://mojolicious.org/) already comes with [hypnotoad](https://docs.mojolicious.org/hypnotoad) which is a production grade server. You can use it and you will be ready in no time. This way is already mentioned in [Deploying a Mojolicious Application using Hypnotoad and Apache](https://perlmaven.com/deploying-a-mojolicious-application). If you are interested you can read it. When I first deployed my mojo app to production I followed that approach (thank you perlmaven :smiley:).

There are several other ways which we will see. But before that I want to thank a unknown person. While exploring more about this deployment process and what to choose I stumble upon this link - https://codeday.me/es/qa/20190709/1032129.html
This link is not working as of today(Don't try to run it on 'http' as it is redirecting to some malicious site). I am not sure who is the owner of this website. But this website contains a lot of useful information regarding Perl. Even though it was in Japanese language (AFAIK), which you can translate easily, I found a great amount of knowledge here. Its a shame its not working anymore. This is also a motivation for me to write this article as I don't want that knowledge to get lost on the internet. I hope someday it will come up and people can see lot of interesting article on it. The thing which I am going to write next is inspired from one of the article mentioned there(the uWSGI part). Right now even I am not sure whether that is the correct link or not, but that is the only one I have. So a big thanks to that unknown person.

I encourage you to look at this [stackoverflow answer](https://stackoverflow.com/questions/12127566/an-explanation-of-the-nginx-starman-dancer-web-stack/12134555#12134555) to understand the basic concepts. It talks about Nginx, PSGI, Plack, Starman and Dancer. It has one of the best explanation and it should be added in hall of fame if possible. :grin:

So lets get started.

# Why uWSGI ?

According to uWSGI project - 
> The uWSGI project aims at developing a full stack for building hosting services.
Versatility, performance, low-resource usage and reliability are the strengths of the project.
The "WSGI" part in the name is a tribute to the namesake Python standard, as it has been the first developed plugin for the project.

Please note that uWSGI(highly-performant WSGI server implementation) is not any language specific. I am taking an example for Perl language, but it is almost similar to other language (Python, Ruby,PHP etc.). I will provide the option in between for other languages too.

Sample Web App Architecture :

**Perl**
```
Code <-> Web framwork (Mojolicious, Dancer, Catalyst etc.) <-> PSGI <-> uWSGI <-> Web Server(apache, nginx) <-> Clients
```
**Python**
```
Code <-> Web framwork (Django, Flask etc.) <-> WSGI <-> uWSGI <-> Web Server(apache, nginx) <-> Clients
```
**Ruby**
```
Code <-> Web framwork (Rails etc.) <-> RACK <-> uWSGI <-> Web Server(apache, nginx) <-> Clients
```
Please keep in mind that PSGI/WSGI/RACK is not Yet Another web application framework. PSGI/WSGI/RACK is a specification to decouple web server environments from web application framework code. Nor is PSGI a web application API. Web application developers (end users) will not run their web applications directly using the PSGI interface, but instead are encouraged to use frameworks that support PSGI.

uWSGI is toolkit that contains PSGI/WSGI/RACK middleware, helpers and adapters to web servers. In other words, they are the implementation of PSGI/WSGI/RACK specification.

There are several ways to setup the particular architecture mentioned above.

1. Using [Gunicorn](https://gunicorn.org/) or 'Green Unicorn' (inspired from Ruby 'Unicorn') for **Python**
       OR
   Using [Plack](https://plackperl.org/)(inspired from Ruby 'Rack') and [Starman](https://metacpan.org/pod/Starman) / [Starlet](https://metacpan.org/pod/Starlet) / [Gazelle](https://metacpan.org/pod/Gazelle) for **Perl**.

2. Using ([WSGI](https://wsgi.readthedocs.io/en/latest/) + uWSGI) for **Python** 
       OR
   ([PSGI](https://metacpan.org/pod/PSGI) + uWSGI) for **Perl**
      OR
   ([RACK](https://github.com/rack/rack) + uWSGI) for **Ruby**.

All these WSGI/PSGI/RACK are plugin provided by uWSGI which extend across almost all languages.

So the question is which option is best or which has more advantage over other - I will try to explain with help of Perl but I hope it is true across other language as everyone is inspired from each other.

1. The PSGI 'protocol' (like WSGI) is essentially a calling convention for a subroutine. A request enters the application as a subroutine call with a hash as an argument. The application responds through the return value of the subroutine: an arrayref that contains an HTTP status code, HTTP headers and body. There is more than that, but those are the essential elements.

2. What this means is that a process can only implement PSGI if the process contains a Perl interpreter. To achieve this, the process can be implemented in Perl or implemented in a language like C that can be loaded by the `libperl.so` shared library. Similarly, a process can only implement WSGI if it contains a Python interpreter.

3. In reality the PSGI application is within the `Starman` process. So there are really only two parts (although both parts are multi-process containers).

4. When we say that "nginx has uWSGI directly integrated", this does not mean that a WGSI application runs within the Nginx process. It means that the WSGI application runs in a separate uwsgi process and Nginx communicates with that process through a TCP socket using the uWSGI protocol. This is essentially the same model as Nginx with Starman behind, but with the distinction that the socket connection to Starman will use the HTTP protocol:
```
    .----------------------.          .-----------.
    |       Starman        |          |   Nginx   |
    |                      |   HTTP   |      /    |   HTTP
    | .------------------. |<---------|   Apache  |<-------(internet)
    | | PSGI Application | |          |           |
    | '------------------' |          |           |
    '----------------------'          '-----------'
```
The HTTP protocol has higher overhead than the uWSGI protocol(remember [OSI Model](https://en.wikipedia.org/wiki/OSI_model) - HTTP at `Application Layer`(7) while TCP at `Transport Layer`(4)), so you can get better performance by running an application server that speaks the uWSGI socket protocol and can load libperl.so to implement the PSGI interface. uWSGI can do that :
```
    .----------------------.           .----------.
    |        uWSGI         |           |  Nginx   |
    |                      |   uWSGI   |     /    |   HTTP
    | .------------------. |<----------|  Apache  |<-------(internet)
    | | PSGI Application | |           |          |
    | '------------------' |           |          |
    '----------------------'           '----------'
```
Hence it is encouraged to use uWSGI over any language specific implementation.

All few thing to note here is that-

1. uWSGI implementation is available in almost all language (no more mod_perl or mod_python (language specific))
2. It can be implemented across CGI script also even mason too.
3. Applicable across different Web server. So if tomorrow you want nginx instead of Apache, its 5 min of work. Even some has out of box support for it (e.g. Nginx).
4. Scalability
5. Speed

# How to use uWSGI ?
* First install 'uWSGI'. It is available as package in several OS/distributions. So at most you have to do 
```shell
sudo apt-get install uwsgi
```
* Install the plugin specific to language you are using. Each language has a plugin associated with it -
```
    uwsgi-plugin-psgi    -> Perl
    uwsgi-plugin-python3 -> Python3
    uwsgi-plugin-python  -> Python2.7
    uwsgi-plugin-ruby    -> Ruby
```
You can install these plugin using apt-get(which I prefer).
  OR
```shell
curl http://uwsgi.it/install | bash -s psgi /tmp/uwsgi
```
* Web server(Apache, Nginx) specific changes needed

For **Apache** - `mod_proxy_uwsgi`. More info [here](https://uwsgi-docs.readthedocs.io/en/latest/Apache.html)
```shell
sudo apt-get install libapache2-mod-proxy-uwsgi
```
**Nginx** includes uwsgi protocol support out of the box. More info [here](https://uwsgi-docs.readthedocs.io/en/latest/Nginx.html)

# How to Run
### Running on terminal (without web server)-

**Python**
```shell
uwsgi --http-socket :8080 --wsgi-file <Application Script>
```
**Perl**
```shell
uwsgi_psgi --http-socket :8080 --psgi script/my_app
```
  OR
```shell
uwsgi --plugins http,psgi --http :8080 --http-modifier1 5 --psgi script/my_app
```

Please note that 'http-modifier' tag in option.
1. uWSGI supports various languages and platform. When the server receives a request it has to know where to ‘route’ it.
2. Each uWSGI plugin has an assigned number (the modifier), the perl/psgi one has the 5. So –http-modifier1 5 means “route to the psgi plugin”.
3. ruby/rack has 7.
4. lua has 6.

### If using socket (through Web Server)-
**Python**
```shell
uwsgi --socket 127.0.0.1:8080 -w wsgi
```
**Perl**
```shell
uwsgi_psgi --socket :8080 --protocol=http --psgi script/my_app
```
There are various command line parameters. Have a look at [Options](https://uwsgi-docs.readthedocs.io/en/latest/Options.html) to understand them and use them as per your need. 
Since it support a large range of options, we will be using the config file instead of command line params as it will be easy to read and maintain.

Whatever we have read till now is generic introductions of uWSGI.
Now lets try to use it for our mojo app.

# Creating Apache configuration
You can use any web server you like. I am just using Apache(I have a soft corner for it :sweat_smile:).
Create a file in `etc\apache2.conf`
```shell
<VirtualHost *:80>
    ServerAdmin grai@gmail.com
    ServerName  mojo-react-app.com

    RewriteEngine on
    # This checks to make sure the connection is not already HTTPS
    RewriteCond %{HTTPS} off [OR]
    RewriteCond %{HTTP:X-Forwarded-Proto} !https

    # Redirect http (port 80) to https (port 443)
    RewriteRule ^/?(.*) https://%{SERVER_NAME}/$1 [NC,R=301,L]

</VirtualHost>

<VirtualHost *:443>
    ServerAdmin grai@gmail.com
    ServerName  mojo-react-app.com

    SSLEngine on
    SSLProxyEngine on
    SSLProxyVerify none 
    SSLProxyCheckPeerCN off
    SSLProxyCheckPeerName off
    SSLProxyCheckPeerExpire off
    SSLCertificateFile /etc/ssl/certs/server.crt
    SSLCertificateKeyFile /etc/ssl/private/server.key

    DocumentRoot /var/www/html
    <Proxy *>
        Order deny,allow
        Allow from all
    </Proxy>
    ProxyRequests Off
    ProxyPreserveHost On
    ProxyPass / uwsgi://127.0.0.1:6363/ keepalive=On
    ProxyPassReverse / uwsgi://127.0.0.1:6363/
    RequestHeader set X-Forwarded-Ssl on
    RequestHeader set X-Forwarded-Proto "https"

    ErrorLog ${APACHE_LOG_DIR}/mojo-react-app-error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/mojo-react-app-access.log combined
</VirtualHost>
```
We have two host listening on `80`(http) and `443`(https). While the first one is doing nothing except redirecting to 443. It means if we try to access out website from browser as plain `http` it will redirect the connection to `https` and will force us to go through that path. We will look into it in real time. But its a good addition to a website. With just few lines of code we are supporting both http and https.
I really love that when a web site owner does this (redirect).

The configuration is pretty standard. You can find more details [here](https://httpd.apache.org/docs/2.4/ssl/ssl_howto.html) and [here](https://httpd.apache.org/docs/2.4/vhosts/examples.html) about what each one of these means.
One thing I would like to point out is `ProxyPass` and `ProxyPassReverse`. Here we just doing the forwarding. `6363` is the port where our uWSGI app is listening. We are forwarding the incoming request to it which will forward it to back-end server. More details - [Reverse Proxy](https://httpd.apache.org/docs/current/howto/reverse_proxy.html).
`SSLCertificateFile` and `SSLCertificateKeyFile` contains the path to CA signed certificate file on your system.
Also, we have created a separate error and access log for our application(which you should do in case you have multiple application running). All the incoming request hitting Apache and error if any will be logged to those logs.

# Creating uWSGI configuration
As I mentioned before we will be using config file for uWSGI instead of command line, so lets create that file. I am creating a `ini` format file but other formats are also acceptable(xml, json, yaml)
```shell
[uwsgi]
project = MojoReactApp
chdir = /home/mojo_react_app
# spawn the specified number of workers/processes
workers = 4
# enable master process
master = true

[uwsgi_psgi]
ini = :uwsgi
# Currently the module lacks the ability to set modifiers, though this will be fixed soon.
# An alternative is to set the plugin you want to use as the first one (0)
plugins = 0:psgi
psgi = script/mojo_react_app
# set uwsgi protocol modifier1 (perl/psgi is 5)
http-socket-modifier1 = 5
# do not catch $SIG{__DIE__}
perl-no-die-catch = true

[development]
# This will load the uwsgi section below
ini = :uwsgi_psgi
# set environment variable
env = PLACK_ENV=development
# bind to the specified UNIX/TCP socket using uwsgi protocol
socket = 127.0.0.1:6363
logto = log/uwsgi_development.log

[staging]
# This will load the uwsgi section below
ini = :uwsgi_psgi
# set environment variable
# staging is similar to production, there is no separate DB for staging
env = PLACK_ENV=production
# bind to the specified UNIX/TCP socket using uwsgi protocol
socket = 127.0.0.1:6363
logto = log/uwsgi_staging.log

[production]
# This will load the uwsgi section below
ini = :uwsgi_psgi
# set environment variable
env = PLACK_ENV=production
# bind to the specified UNIX/TCP socket using uwsgi protocol
socket = 127.0.0.1:6363
logto = log/uwsgi_production.log
```
Please have a look at [Configuration](https://uwsgi-docs.readthedocs.io/en/latest/Configuration.html) for more details about each items. Check the [ini-files](https://uwsgi-docs.readthedocs.io/en/latest/Configuration.html#ini-files) section for creating the ini config.

> By default, uWSGI uses the [uwsgi] section, but you can specify another section name while loading the INI file with the syntax `filename:section`, that is:

    uwsgi --ini myconf.ini:app1

> Alternatively, you can load another section from the same file by omitting the filename and specifying just the section name.

* In our `[uwsgi]` section we have created the config which we want globally available.
* `[uwsgi_psgi]` section contain items specific to psgi plugin. If you are using some other plugin (e.g. ruby) you can update it for that. I have added the comment on each line to understand it better.
* I have created 3 more section - `[development]`, `[staging]` and `[production]`. This is something which I generally do for all my project. In all 3 I have loaded the [uwsgi_psgi] section and in [uwsgi_psgi] I have loaded [uwsgi] meaning all those items are available in these 3 section.
* You can add the params which you think is specific to particular environment(e.g. PLACK_ENV).
* `socket` contains the address where the request coming from Apache will be forwarded (remember `ProxyPass`). This is the address where your mojolicious app will be running.
* `logto` contains the filename where the logs will be generated.

The reason I have created 3 sections - dev, stag and prod because it make everything so smooth. In just few min you have your specific environment ready. We will look into it more and what I meant by this statement.

# Creating the signed certificate
For now I am creating a [self-signed certificate](https://en.wikipedia.org/wiki/Self-signed_certificate)
 but for actual production use you should get a CA signed certificate. If your website is going to be in a public space accessible across world, better to get a proper certificate.
A self-signed certificate is ok till your usage is limited (intranet site etc.).

There are plenty of place where you can learn how to create it. One of them is - [How To Create a Self-Signed SSL Certificate](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04)
Also, you should not be committing this certificate in your repo(git, svn etc). I have just committed it for demo.
This is private to you, use it with utmost care.

# Creating the startup script
Lets create a wrapper script which will start our app.
Inside `script\start_mojo_react_app.sh`
```shell
#!/bin/bash
# This script can be run by docker or directly by provide the parameter.
# Move it outside and run -
# ./start_mojo_react_app.sh -m "development" or
# ./start_mojo_react_app.sh -m "production"

# Restart Web server (here Apache)
service apache2 restart

# start Mojo React App
while getopts "m:" opt; do
    case ${opt} in
        m)
            mode="${OPTARG}"
            echo "Running in '$mode' mode"
            if [ "$mode" == "development" ]; then
                # Since we are using uwsgi, we have commented the morbo
                # exec morbo -l "https://*:6363" script/mojo_react_app
                exec uwsgi --ini etc/uwsgi.conf:$mode
            elif [[ "$mode" = "production" || "$mode" = "staging" ]]; then
                # exec hypnotoad -f script/mojo_react_app
                exec uwsgi --ini etc/uwsgi.conf:$mode
            else 
                echo "Wrong mode provided. Accepted value - development, staging or production" 1>&2
            fi
            ;;
        : )
            echo "Invalid option: $OPTARG requires an argument" 1>&2
    esac
done
```
* In my case I have merged the `staging` and `production` environment. But you can segregate them based on your need.
* I have commented the [morbo](https://docs.mojolicious.org/morbo)(development server) and [hypnotoad](https://docs.mojolicious.org/hypnotoad)(HTTP and WebSocket production server) line. The reason being we are using uWSGI. In case you don't want to use uWSGI you can uncomment those and remove uWSGI line. Also don't forget to add the hypnotoad specific config and update the `ProxyPass` and `ProxyPassReverse` in `apache2.conf` :smirk: 
```
    ProxyPass / http://127.0.0.1:6363/ keepalive=On
    ProxyPassReverse / http://127.0.0.1:6363/
```


# Creating the Dockerfile
Now lets wrap our application with all the configuration file created above.
Create a Dockerfile in `mojo_react_app\Dockerfile`
```docker
# build environment
FROM ubuntu:18.04

# By default will run in 'dev' mode
ENV mode=development

# Needed dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    gnupg \
    curl \
    vim \
    less \
    openssl=1.1.1-1ubuntu2.1~18.04.9 \
    libssl-dev \
    zlib1g-dev \
    apache2 \
    uwsgi=2.0.15-10.2ubuntu2.1 \
    uwsgi-plugin-psgi=2.0.15-10.2ubuntu2.1 \
    libapache2-mod-proxy-uwsgi=2.0.15-10.2ubuntu2.1

# Needed dependencies specific to project
RUN curl -L https://cpanmin.us | perl - -M https://cpan.metacpan.org -n Mojolicious@9.17
RUN curl -L https://cpanmin.us | perl - -M https://cpan.metacpan.org -n Mojolicious::Plugin::OpenAPI@4.03
RUN curl -L https://cpanmin.us | perl - -M https://cpan.metacpan.org -n Mojolicious::Plugin::SwaggerUI@0.0.4
RUN curl -L https://cpanmin.us | perl - -M https://cpan.metacpan.org -n IO::Socket::SSL@2.070
RUN curl -L https://cpanmin.us | perl - -M https://cpan.metacpan.org -n JSON::XS@4.03

# Apache specific configuration
RUN \
    a2enmod headers && \
    a2enmod proxy && \
    a2enmod proxy_http && \
    a2enmod proxy_uwsgi && \
    a2enmod rewrite && \
    a2enmod ssl

# Disable the default apache home page on port 80
RUN a2dissite 000-default.conf

# Copy your codebase from local to inside container
COPY . /home/mojo_react_app
RUN mkdir -p /home/mojo_react_app/log/

# These will be used while setting virtual host in apache.conf
ENV APACHE_LOG_DIR /var/log/apache2
ENV APACHE_LOCK_DIR /var/lock/apache2
ENV APACHE_PID_FILE /var/run/apache2.pid

# Copy Apache config file
COPY etc/apache2.conf /etc/apache2/sites-available/mojo_react_app.conf
RUN ln -s /etc/apache2/sites-available/mojo_react_app.conf /etc/apache2/sites-enabled/mojo_react_app.conf

WORKDIR /home/mojo_react_app

# Expose both http and https port
EXPOSE 80 443

COPY script/start_mojo_react_app.sh /start_mojo_react_app.sh
RUN chmod a+x /start_mojo_react_app.sh

ENTRYPOINT /start_mojo_react_app.sh -m "$mode"
```
I have added comment for better understanding.
* We are using the ubuntu(18.04) image. You can use the recent one (20.04) but still some plugins are missing for this version hence I have used 18.04. You can also use the official [Perl](https://hub.docker.com/_/perl) docker image instead if you want (e.g. perl:5.32).

* By default we will running in `development` mode. You can override it by providing different value at runtime (e.g. staging or production). This is the one which will cause the the different section to pick in uwsgi.conf.

* Next we are installing some tools(e.g. vim). In case you don't want it you can remove those lines. Few notable plugins/tools are - `openssl`(for https), `apache2`, `uwsgi`, `uwsgi-plugin-psgi`(for perl), `libapache2-mod-proxy-uwsgi`(for apache2).
I have used some version which are specific to ubuntu(18.04), you can update that as per your image or not use at all.

* Next I have installed some dependencies specific to our Mojolicious app. I am using [cpanminus](https://metacpan.org/pod/App::cpanminus) for this. I have added the version number also to get that specific version. There are again multiple way to achieve this. One of them is using `cpanm`(with [cpanfile](https://metacpan.org/pod/distribution/Module-CPANfile/lib/cpanfile.pod)) from command line as mentioned [here](https://metacpan.org/pod/distribution/App-cpanminus/bin/cpanm) and [here](https://mvp.kablamo.org/dependencies/cpanm/). Another is using [Carton](https://metacpan.org/pod/Carton) which is dependency manager for Perl(similar to Bundler in Ruby).
Since our scope is limited I have used it like that. In big project you may want to follow one of above mentioned approach.

* Next is some Apache specific configuration. We are enabling some modules in Apache. More info at [a2enmod](https://manpages.ubuntu.com/manpages/trusty/man8/a2enmod.8.html).
We are enabling the ability of handling HTTP proxy requests, handling the uWSGI protocol, ssl etc. Also we disable the default Apache homepage which is available on port 80.

* After that we are copying our `mojo_react_app` code base and placing it at `/home/mojo_react_app` inside container.

* We are setting some environment variables which we are using in `apache.conf` (remember `${APACHE_LOG_DIR}`).

* Next we copied our virtual host config file and copied it inside container(site-available). Also we created a symbolic link for it in `/etc/apache2/sites-enabled/`.

* Now we make the change our working dir to `/home/mojo_react_app` inside container where all our code base is.

* We have exposed the 80 and 443 port to outside world.

* We copied the startup script to our current working dir(which is /home/mojo_react_app).

* Finally we are running that startup script with `mode` as param (by default - development).

All the docker keywords and there meaning is already available at - [Dockerfile reference](https://docs.docker.com/engine/reference/builder/). Check it out for more info.

With that we are ready to start our application.

# Running the application
### Build the dockerfile
```shell
docker build --pull=false --no-cache=false -t mojo_react_app:development .
```
### Create the container
```shell
docker create --name mojo_react_app_development -t \
       -p 0.0.0.0:80:80 -p 0.0.0.0:443:443 \
       --env mode=development docker create --name 
```
### Copy the signed certificates
We will copy the self signed certificate inside the created container. This certificate will be available somewhere on your prod machine(private). For now I am copying it from the project dir.
```shell
docker cp mojo_react_app/apache-certificate/apache_certificate_development.crt mojo_react_app_development:/etc/ssl/certs/server.crt

docker cp mojo_react_app/mojo_react_app/apache-certificate/apache_certificate_development.key mojo_react_app_development:/etc/ssl/private/server.key
```
Remember the destination path is the one which we used in `apache2.conf` for virtual host configuration.

### Start the container
```shell
docker start mojo_react_app_development
```
### Show running containers
Just to check whether our container is running or not
```shell
docker ps | grep mojo_react_app_development
```
You can change the `development` in name and `mode` to `staging` or `production` for different environment.

Lets login to container and see what is going on there.
```shell
docker exec -it mojo_react_app_development bash
```
Lets go to apache log dir to see the logs there
![apache_logs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hh318tqq7dpe93hknwi0.PNG)
We can see our application access and error are getting generated here. Whatever the request apache received that will be logged here.

Lets go to our project dir and see the uWSGI logs
![uwsgi_log](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0pcck347vwn94g9ijm0o.PNG)
All the mojolicious app log will be logged here.

Lets tail this log and see whats inside
![tail_uwsgi](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uo1hyy3ijq3nap9u9mm2.PNG)
Hmm, all the 4 works which we have configured in [uwsgi] section inside `uwsgi.conf` is ready to rock.

Just open the browser and hit the http://localhost
![redirect_301](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2zgu96v3srnrvvu0uhpw.PNG)
* Just look at the *Network* tab in browser. Even though you hit `http` it got redirected to `https`. The status code 301 says so.
The url itself is now https://localhost at the top.
* Also you can see the server is *Apache(2.4.29)* running on *Ubuntu* and not *Mojolicious(Perl)* since we are using the reverse proxy architecture.

Lets go ahead and click to open the API page.
![api](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pk7qe0ikn5qhoiuh9h5w.PNG)
Voila!!!. So far so good. It is similar to what we have seen previously except on `https`.

Since we are already tailing the uWSGi log, lets see whats happens there -
![api_logs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uaoolwc5i9mwpy9g8ppv.PNG)
We got a `GET` request on `/api` and we render the template based on our internal logic. All the debug message will go away when you run it in production mode.

Go ahead and try to do the `GET` request on the endpoints. You will be able to do it without any issue.

A high level diagram of architecture we followed is -
![architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/56y2t5k7cjbvulies9u3.PNG) 

# What next
Well even though we are using docker, we have to run several command manually. We have to take care of different environment while creating and starting the image. Not to mention the copy of Apache certificate. What if we can automate it more.
What if we say - `make dev` and all the thing got taken cared of.
Similarly for `make stag` and `make prod`.
This thing will come handy when you will do the automated deployment using Jenkins or some other ways.
I can see the sparkle in your eyes.
![One piece](https://data.whicdn.com/images/186559937/original.gif)
We will look into that in our next section.

# Bonus
Our good folks at Mojolicious already thought of containers and clouds. Have a look at [Containers](https://docs.mojolicious.org/Mojolicious/Guides/Cookbook#Containers) for more info.
You can generate `Makefile` and `Dockerfile` for your app using just 2 simple command. Inside your project dir -
```shell
mojo generate makefile
./script/mojo_react_app generate dockerfile
```

# Honorable Mention -
https://stackoverflow.com/questions/12127566/an-explanation-of-the-nginx-starman-dancer-web-stack/12134555#12134555
https://uwsgi-docs.readthedocs.io/en/latest/index.html
https://codeday.me/es/qa/20190709/1032129.html
https://metacpan.org/pod/PSGI
https://perlmaven.com/deploying-a-mojolicious-application

And a lot of different articles read over the years, the source of which I don't remember. :sob:

The above code is also available at [github](https://github.com/rai-gaurav/mojo_react_app/tree/main/with_jsx/server/mojo_react_app)

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Mojolicious logo taken from [here](https://github.com/mojolicious/mojo/blob/master/lib/Mojolicious/resources/public/mojo/logo.png)
OpenAPI logo taken from [here](https://www.openapis.org/news/blogs/2016/07/you-can-get-involved-creating-openapi-specification-and-heres-how/attachment/openapi_pantone)
uWSGI logo taken from [here](https://www.seekpng.com/ipng/u2t4i1i1y3e6t4q8_official-uwsgi-logo-uwsgi-logo/)
Docker logo taken from [here](https://www.docker.com/company/newsroom/media-resources)
Apache logo taken from [here](https://httpd.apache.org/)
Ubuntu logo taken from [here](https://design.ubuntu.com/brand/ubuntu-logo/)
