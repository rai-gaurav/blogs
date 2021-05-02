As promised in [previous](https://dev.to/raigaurav/data-visualization-using-amcharts-in-react-js-with-mojo-29mh) article, we will be using react.js with jsx. For that we should have our server ready.
Today we will be looking at creating the RESTful API's with [OpenAPI](https://swagger.io/specification/).
Again we will using [Mojolicious](https://mojolicious.org/) for that. Some other prerequisite which we will be using.
1. [Mojolicious::Plugin::OpenAPI](https://metacpan.org/release/Mojolicious-Plugin-OpenAPI) - OpenAPI / Swagger plugin for Mojolicious 
2. [Mojolicious::Plugin::SwaggerUI](https://metacpan.org/release/Mojolicious-Plugin-SwaggerUI) - Swagger UI plugin for Mojolicious

We will be using the same example mentioned previously.
So without further adieu lets get started.

# Creating the data config 
For multi-line chart -
```json
{
    "title": "Number of automobiles sold per day by manufacturer",
    "label": {
        "domainAxis": "Date",
        "rangeAxis": "Numbers of automobiles sold"
    },
    "data": [
        {
            "Date": "2020-04-15",
            "Honda": 10,
            "Toyota": 20,
            "Ford": 6,
            "Renault": 16
        },
        {
            "Date": "2020-04-16",
            "Honda": 3,
            "Toyota": 15,
            "Ford": 19,
            "Renault": 10
        },
        {
            "Date": "2020-04-17",
            "Honda": 5,
            "Toyota": 8,
            "Ford": 12,
            "Renault": 6
        },
        {
            "Date": "2020-04-18",
            "Honda": 9,
            "Toyota": 10,
            "Ford": 4,
            "Renault": 12
        }
    ]
}
```
We will be creating one more chart in this example - stacked clustered column chart. We will look into it in more details when we will create the client side. But for now, lets see the input data which is similar to previous one.
```json
{
    "title": "Revenues by region",
    "label": {
        "domainAxis": "Date",
        "rangeAxis": "Expenditure"
    },
    "data": [
        {
            "Date": "2020-04-15",
            "Europe": 2.5,
            "America": 2.5,
            "Asia": 2.1,
            "Australia": 1.2,
            "Antartica": 0.2,
            "Africa": 0.1
        },
        {
            "year": "2004",
            "Europe": 2.6,
            "America": 2.7,
            "Asia": 2.2,
            "Australia": 1.3,
            "Antartica": 0.3,
            "Africa": 0.1
        },
        {
            "year": "2005",
            "Europe": 2.8,
            "America": 2.9,
            "Asia": 2.4,
            "Australia": 1.4,
            "Antartica": 0.3,
            "Africa": 0.1
        }
    ]
}
```

# Creating the mojo app
The version I am using for this article is 9.14.
```shell
$  mojo generate app MojoReactApp
```
Running the app -
```shell
$ morbo ./script/mojo_app
Web application available at http://127.0.0.1:3000
```
Now we will modify this app to suit our need.

# Writing the specification
For API creation we will follow the [OpenAPI specification](https://swagger.io/specification/). We will be using 3.0 version for this. 
```json
{
    "openapi": "3.0.2",
    "info": {
        "version": "1.0",
        "title": "Mojo React App API",
        "description": "This is a sample server for a mojolicious app.",
        "contact": {
            "name": "Gaurav Rai",
            "url": "https://github.com/rai-gaurav"
        }
    },
    "servers": [
        {
            "url": "/api/v1",
            "description": "Version one api"
        }
    ],
    "paths": {
        "/multi-line-chart": {
            "get": {
                "summary": "Get multi line chart data",
                "tags": ["Chart Data"],
                "operationId": "getMultiLineChartData",
                "x-mojo-name": "get_multi_line_chart_data",
                "x-mojo-to": {
                    "controller": "LineCharts",
                    "action": "get_multi_line_chart"
                },
                "responses": {
                    "200": {
                        "description": "Multi Line Chart Response",
                        "content": {
                            "application/json": {
                                "schema": {
                                    "type": "object",
                                    "properties": {
                                        "chart_data": {
                                            "type": "object",
                                            "items": {
                                                "type": "object"
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        },
        "/stacked-column-chart": {
            "get": {
                "summary": "Get stacked column chart data",
                "tags": ["Chart Data"],
                "operationId": "getStackedColumnChartData",
                "x-mojo-name": "get_stacked_column_chart_data",
                "x-mojo-to": {
                    "controller": "ColumnCharts",
                    "action": "get_stacked_column_chart"
                },
                "responses": {
                    "200": {
                        "description": "Stacked Column Chart Response",
                        "content": {
                            "application/json": {
                                "schema": {
                                    "type": "object",
                                    "properties": {
                                        "chart_data": {
                                            "type": "object",
                                            "items": {
                                                "type": "object"
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```
The above specification can also be written in .yaml format also. For now I am using json one. Save it as `public/api.json`.

* Inside `server` section we are using `/api/v1` as `url`. You can provide full url or relative one. If relative, it is resolved against the server where the given OpenAPI definition file is hosted. More info [here](https://swagger.io/docs/specification/api-host-and-base-path/)
* There are total two API endpoint `/multi-line-chart` and `/stacked-column-chart` for our 2 charts. But support only `GET` request.
* I encourage you to checkout the [specification](https://swagger.io/specification/) for more details on each and every key.
* Two keys which are specific to Mojolicious are - `x-mojo-name` and `x-mojo-to`. `x-mojo-to` contains the name of controller and action to perform in case that particular endpoint is requested. Checkout [Mojolicious::Plugin::OpenAPI::Guides::OpenAPIv3](https://metacpan.org/pod/distribution/Mojolicious-Plugin-OpenAPI/lib/Mojolicious/Plugin/OpenAPI/Guides/OpenAPIv3.pod) for more details.

# Using the plugin in Mojolicious App
Lets add the OpenAPI and SwaggerUi plugin in our application.
Inside MojoReactApp.pm
```perl
package MojoReactApp;
use Mojo::Base 'Mojolicious', -signatures;
use MojoReactApp::Model::Data;

# This method will run once at server start
sub startup ($self) {

    # Load configuration from config file
    my $config = $self->plugin('NotYAMLConfig');

    # Configure the application
    $self->secrets($config->{secrets});

    # Load the "api.json" specification
    # Can also be written as -
    # "OpenAPI" => {spec => $self->static->file("api.json")->path}

    $self->plugin(
        "OpenAPI" => {
            url => $self->home->rel_file("public/api.json")
        }
    );

    $self->plugin(
        SwaggerUI => {
            route => $self->routes()->any('api'),
            url => "/api/v1",
            title => "My Mojolicious App"
        }
    );

    # Helper to lazy initialize and store our model object
    $self->helper(
        model => sub ($c) {
            state $data = MojoReactApp::Model::Data->new();
            return $data;
        }
    );

    # Router
    my $r = $self->routes;

    # Normal route to controller
    $r->get('/')->to(template => 'home/welcome');
}

1;
```
Here I have just used the OpenAPI and SwaggerUI plugin.
We are using the `url` to load the specification which have created before.
SwaggerUI is just a fancy thing I am using. There is no such limitation to used it for creating the API.
* `route` - The ui will be available on http://{hostname}:{port}/api
* `url` - Url for the specification. In `api.json` inside `server` section we have written the url as `/api/v1` meaning our specification is available under that path. Hence we have given the same path here.
* Please have a look at the [options](https://metacpan.org/pod/Mojolicious::Plugin::SwaggerUI#OPTIONS) for the meaning of each parameters.
* If case you noticed, we have created just one route `/` which will just show the home page. Since we are crating Rest API's and all the other endpoint are already taken care in `api.json` we don't have to add it here.
* Our directory structure is also simpler.

📦mojo_react_app
 ┣ 📂etc
 ┃ ┣ 📜input_multi_line_chart_data.json
 ┃ ┗ 📜input_stacked_clustered_column_chart.json
 ┣ 📂lib
 ┃ ┣ 📂MojoReactApp
 ┃ ┃ ┣ 📂Controller
 ┃ ┃ ┃ ┣ 📜ColumnCharts.pm
 ┃ ┃ ┃ ┗ 📜LineCharts.pm
 ┃ ┃ ┗ 📂Model
 ┃ ┃ ┃ ┗ 📜Data.pm
 ┃ ┗ 📜MojoReactApp.pm
 ┣ 📂public
 ┃ ┗ 📜api.json
 ┣ 📂script
 ┃ ┗ 📜mojo_react_app
 ┣ 📂t
 ┃ ┗ 📜basic.t
 ┣ 📂templates
 ┃ ┣ 📂home
 ┃ ┃ ┗ 📜welcome.html.ep
 ┃ ┗ 📂layouts
 ┃ ┃ ┗ 📜default.html.ep
 ┗ 📜mojo_react_app.yml

* The `public` section only contains `api.json`. There is no need of css and js as we are creating only API. Even you can remove the whole `template` section also. I have just added it for `/` endpoint. The `welcome` template is just some minor adjustment to default one.
```html
% layout 'default';
% title 'Welcome';
<h2>Hello there</h2>
<p>
  This is the home page.
  For API <%= link_to 'click here' => '/api' %>.
</p>
```
* Since we are decided to created 2 charts, we have 2 json file in `etc` dir.
* Each one has there own controller.
* Our Model is almost similar to [previous](https://dev.to/raigaurav/data-visualization-using-amcharts-with-perl-and-mojo-38ff) article with minor modification.
```perl
package MojoReactApp::Model::Data;

use strict;
use warnings;
use experimental qw(signatures);
use Mojo::JSON qw(decode_json);

sub new ($class) {
    my $self = {};
    bless $self, $class;
    return $self;
}

sub _read_json_file ($self, $json_file) {
    open(my $in, '<', $json_file) or $self->app->log->error("Unable to open file $json_file : $!");
    my $json_text = do { local $/ = undef; <$in>; };
    close($in) or $self->app->log->error("Unable to close file : $!");

    my $config_data = decode_json($json_text);
    return $config_data;
}

sub get_line_data ($self) {
    my $data_in_json = $self->_read_json_file("etc/input_multi_line_chart_data.json");

    return $data_in_json;
}

sub get_column_data ($self) {
    my $data_in_json = $self->_read_json_file("etc/input_stacked_clustered_column_chart.json");

    return $data_in_json;
}

1;
```

# Controller

Lets create our 2 controller.
Inside `lib\MojoReactApp\Controller\ColumnCharts.pm`

```perl
package MojoReactApp::Controller::ColumnCharts;
use Mojo::Base 'Mojolicious::Controller', -signatures;
use Mojo::JSON qw(encode_json);

sub get_stacked_column_chart ($self) {

    # Do not continue on invalid input and render a default 400 error document.
    my $app = $self->openapi->valid_input or return;

    my $data_in_json = $app->model->get_column_data();

    # $output will be validated by the OpenAPI spec before rendered
    my $output = {chart_data => $data_in_json};
    $app->render(openapi => $output);
}

1;
```
Simple and easy!!. Here we are validating the input in `$self->openapi->valid_input`. After that we are getting the data from model and just returning that as the response inside the hash.

Similarly for `lib\MojoReactApp\Controller\LineCharts.pm`

```perl
package MojoReactApp::Controller::LineCharts;
use Mojo::Base 'Mojolicious::Controller', -signatures;
use Mojo::JSON qw(encode_json);

sub get_multi_line_chart ($self) {

    # Do not continue on invalid input and render a default 400 error document.
    my $app = $self->openapi->valid_input or return;

    my $data_in_json = $app->model->get_line_data();

    # $output will be validated by the OpenAPI spec before rendered
    my $output = {chart_data => $data_in_json};
    $app->render(openapi => $output);
}

1;
```
Exact similar to previous one, except we are calling different model function here.

Lets run it and see the output.
Hitting 'http://localhost:3000/'
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ml42u2oxsvtvybh96bjd.PNG)

You can see your home page which get generated by the `welcome` template. Click on the link or hit 'http://localhost:3000/api' in browser, you can see the shiny Swagger UI.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4e03ax2lqabz5k8nv8sj.PNG)
We can also see the 2 endpoint which you have created.
We can see all the things which we have written in `api.json`
This swagger ui is ultimately using that specification and generating this page.
Hit 'http://localhost:3000/api/v1' and you can see the specifications also.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jkjhgzvj4tkhsj95iohw.PNG)

Now lets try to do a GET request on one of the endpoint. You can use the SwaggerUI for this or a normal curl request or directly hit the endpoint from browser or use some 3rd party tool like [Postman](https://www.postman.com/) , all will work.
SwaggerUI will just give you some documentation about usage and the expected response.

Let hits the `/multi-line-chart` endpoint. In SwaggerUi page, click on 'Try it out' and 'execute' it.
You can see the response in JSON format.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/90y76p77qjrbnmzlgpsd.PNG)
We can see there is equivalent curl request already prepared which we can also use to get the same response. 
Also in 'Requested URL' we can see ultimately it is calling the '/api/v1/multi-line-chart' which you can hit from browser and see the exact result.

With this our API development is done. Next we will be seeing how to use this API in React.js and create those charts in jsx.
Also, I have used `morbo` which is good for development. But for production we need to make certain changes. I will talk about those (Docker, Makefile, Apache2/Nginx, uWSGI/Plack and hypnotoad) in a different section.

The above example is available at [github](https://github.com/rai-gaurav/mojo_react_app/tree/main/with_jsx/server/mojo_react_app).

There is also a good tutorial available on Mojolicious blog - [A RESTful API with OpenAPI](https://mojolicious.io/blog/2017/12/22/day-22-how-to-build-a-public-rest-api/)

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Mojolicious logo taken from [here](https://github.com/mojolicious/mojo/blob/master/lib/Mojolicious/resources/public/mojo/logo.png)
OpenAPI logo taken from [here](https://www.openapis.org/news/blogs/2016/07/you-can-get-involved-creating-openapi-specification-and-heres-how/attachment/openapi_pantone)
