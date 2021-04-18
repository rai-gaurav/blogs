In my [previous](https://dev.to/raigaurav/data-visualization-creating-charts-from-perl-using-plotly-js-chart-plotly-3m48) article, I talked about [Chart::Plotly](https://metacpan.org/pod/Chart::Plotly). Today we will looking at creating the similar chart using another javascript library [amCharts](https://www.amcharts.com/).

I have the opportunity to work on both [v3](https://github.com/amcharts/amcharts3) and [v4](https://github.com/amcharts/amcharts4) of amCharts. v3 is currently in maintenance mode. v4 is rewritten in typescript. One good thing about the library is there are lot of documentation and examples available on there website. Also you can use it in plain Javascript or integrate them into various application frameworks - React, Angular2+, Ember, Vue.js etc.
Also you don't need to be javascript expert to use it. It is highly configurable. You can use any syntax for configuration - TypeScript/ES6, JavaScript or JSON. For more details have a look at there excellent [documentation](https://www.amcharts.com/docs/v4/).

Without further adieu lets get started.
#Creating the data config

We will use the exact same example as in [previous](https://dev.to/raigaurav/data-visualization-creating-charts-using-perl-chart-clicker-1hm) article and try to create a multi line chart. But this time we will tweak the data format a little bit.

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
The reason we are using this format is because amCharts use array of objects to create chart where each object in the array represents a single data point. More info [here](https://www.amcharts.com/docs/v4/concepts/data/).
We can use any data format but ultimately we have to convert it as array of object before creating chart which doesn't make sense (especially if you are doing it at time of page loading). So why not to create the data in the format which we can use easily.

# Creating the mojo app
We we will be using the [Mojolicious](https://mojolicious.org/) framework for server side. You can install it using single command as mentioned on website -
```shell
$ curl -L https://cpanmin.us | perl - -M https://cpan.metacpan.org -n Mojolicious
```
It also have excellent [documentation](https://docs.mojolicious.org/). Have a look at it to learn more.
The version I am using for this article is 9.14.
We will go ahead and create an app from command line.
```shell
$  mojo generate app MojoApp
```
This command will generate a example application with proper directory structure for a MVC application. Easy peasy
```
📦mojo_app
 ┣ 📂lib
 ┃ ┣ 📂MojoApp
 ┃ ┃ ┗ 📂Controller
 ┃ ┃ ┃ ┗ 📜Example.pm
 ┃ ┗ 📜MojoApp.pm
 ┣ 📂public
 ┃ ┗ 📜index.html
 ┣ 📂script
 ┃ ┗ 📜mojo_app
 ┣ 📂t
 ┃ ┗ 📜basic.t
 ┣ 📂templates
 ┃ ┣ 📂example
 ┃ ┃ ┗ 📜welcome.html.ep
 ┃ ┗ 📂layouts
 ┃ ┃ ┗ 📜default.html.ep
 ┗ 📜mojo_app.yml
```
Now go inside the dir and try to run this app.
```shell
$ morbo ./script/mojo_app
Web application available at http://127.0.0.1:3000
```
Open the browser and hit http://localhost:3000/ and you can see the welcome page.
If you open and look into `MojoApp.pm` you can see - `get` request on `/`(home page) is redirected to `example` controller (Example.pm) and function `welcome` is called inside that controller to fulfill the request. You can also see the template `example/welcome.html.ep` is rendered inside that function which you are seeing when you hit the `http://localhost:3000/`

We will be adding/modifying some parts of this dir structure to suit our need.
1. We will be creating a 'mojo_app/etc/' dir to put our 'input_data.json' created previously.
2. We will be renaming the default controller `example` to something meaningful
3. Also we will be modifying the `layouts\default.html.ep` template.
4. And we will be adding amCharts javascript library in template.

Update MojoApp.pm with the following changes in `startup`-
```perl
    # Normal route to controller
    $r->get('/')->to('charts#create_multi_line_chart');
```
Create new or rename Example.pm to Charts.pm in `Controller` and update it with - 
```perl
package MojoApp::Controller::Charts;
use Mojo::Base 'Mojolicious::Controller', -signatures;
use Mojo::JSON qw(decode_json encode_json);

sub read_json_file ($self, $json_file) {

    open(my $in, '<', $json_file) or $self->app->log->error("Unable to open file $json_file : $!");
    my $json_text = do { local $/ = undef; <$in>; };
    close($in) or $self->app->log->error("Unable to close file : $!");

    my $config_data = decode_json($json_text);
    return $config_data;
}

sub create_multi_line_chart ($self) {
    my $data_in_json = $self->read_json_file( "etc/input_data.json");

    $self->render(template => 'charts/multi_line_chart', chart_data => encode_json($data_in_json));
}

1;
```
Here we are just reading the input json file and rendering the template with the chart data. Please note that `create_multi_line_chart` will be called at every load of page. Here I am reading the file every time. You can optimize it by reading it once at the start or caching it in case your input data doesn't change that often.
The JSON file is just an example. You can get this data from a database also.
Since we are talking about MVC framwork, why not move this data logic to `Model`.
Create `lib\MojoApp\Model\Data.pm` and update it with
```perl
package MojoApp::Model::Data;

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

sub get_data ($self) {
    my $data_in_json = $self->_read_json_file("etc/input_data.json");

    return $data_in_json;
}

1;
```
Again, you can connect to DB and generate this data. For simplicity I am just getting the data from JSON file. (This data is actually generated from CouchDB :P).
Lets update our `startup` in MojoApp.pm
```perl
use MojoApp::Model::Data;

sub startup ($self) {

...
    # Helper to lazy initialize and store our model object
    $self->helper(
        model => sub ($c) {
            state $data = MojoApp::Model::Data->new();
            return $data;
        }
    );
...

}
```
Lets remove the extra thing from controller Charts.pm and use this helper.
```perl
package MojoApp::Controller::Charts;
use Mojo::Base 'Mojolicious::Controller', -signatures;
use Mojo::JSON qw(encode_json);

sub create_multi_line_chart ($self) {
    my $data_in_json = $self->model->get_data();

    $self->render(template => 'charts/multi_line_chart', chart_data => encode_json($data_in_json));
}

1;
```
We updated the controller to use the model for data and render the template.
Now lets go to `template` section and update/create a folder name `charts` in which we will be creating template `multi_line_chart.html.ep`.
Also lets update the `default.html.ep` template a little bit.
```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title><%= title %></title>

        %= content 'head'
    </head>
    <body>
        <div>
            %= content
        </div>
        %= content 'end'
    </body>
</html>
```
This is our layout template and we will be using it at our every page throughout the website. There are different placeholders where we will genrating data for different pages. For more details have a look at [Mojolicious::Guides::Rendering](https://docs.mojolicious.org/Mojolicious/Guides/Rendering) and [Mojo::Template](https://docs.mojolicious.org/Mojo/Template)
In multi_line_chart.html.ep
```perl
% layout 'default';
% title 'Charts';

% content_for 'head' => begin
    <link rel="stylesheet" type="text/css" href="css/charts.css">
% end

<div id="chartdiv"></div>

% content_for 'end' => begin
    %= javascript "https://cdn.amcharts.com/lib/4/core.js"
    %= javascript "https://cdn.amcharts.com/lib/4/charts.js"
    %= javascript "https://cdn.amcharts.com/lib/4/themes/animated.js"

    %= javascript "js/multi_line_chart.js"

    %= javascript begin
        createMultiLineChart(<%== $chart_data %>);
    % end
% end
```
In simple language, we are saying here - use the `default.html.ep` template, update the title of the page to 'Charts', append the `head` section with the css for this page, in the page body create a 'div' with 'id' `chartdiv` and in the end of the body add the mentioned javascripts file.
The `$chart_data` which we are using in javascript, gets passed from server side while rendering the template in `create_multi_line_chart` method. It is encoded in JSON for which we are decoding on client side.
The top 3 javascript included are amCharts library.
Now lets create `charts.css` and `multi_line_chart.js` which we are referencing here. These will be automatically served from 'public' dir.
In `public/css/charts.css`
```css
#chartdiv {
    width: 850px;
    height: 550px;
}
```
Its very small css where we just setting the dimensions of the chart.
In `public/js/multi_line_chart.js`
```javascript
function createSeries(chart, axis, field, name) {
    // Create series
    var series = chart.series.push(new am4charts.LineSeries());
    series.dataFields.dateX = "Date";
    series.dataFields.valueY = field;
    series.strokeWidth = 2;
    series.xAxis = axis;
    series.name = name;
    series.tooltipText = "{name}: [bold]{valueY}[/]";

    var bullet = series.bullets.push(new am4charts.CircleBullet());

    return series;
}

function createMultiLineChart(chartData) {
    // Themes begin
    am4core.useTheme(am4themes_animated);

    var chart = am4core.create("chartdiv", am4charts.XYChart);

    // Increase contrast by taking every second color
    chart.colors.step = 2;
    // Add title to chart
    var title = chart.titles.create();
    title.text = chartData["title"];

    // Add data to chart
    chart.data = chartData["data"];

    // Create axes
    var dateAxis = chart.xAxes.push(new am4charts.DateAxis());
    dateAxis.title.text = chartData["label"]["domainAxis"];

    var valueAxis = chart.yAxes.push(new am4charts.ValueAxis());
    valueAxis.title.text = chartData["label"]["rangeAxis"];

    //var single_data_item = chartData["data"][0];
    var series1 = createSeries(chart, dateAxis, "Toyota", "Toyota");
    var series2 = createSeries(chart, dateAxis, "Ford", "Ford");
    var series3 = createSeries(chart, dateAxis, "Honda", "Honda");
    var series4 = createSeries(chart, dateAxis, "Renault", "Renault");

    // Add legend
    chart.legend = new am4charts.Legend();

    // Add cursor
    chart.cursor = new am4charts.XYCursor();
    chart.cursor.xAxis = dateAxis;

    // Add scrollbar
    chart.scrollbarX = new am4core.Scrollbar();

    // Add export menu
    chart.exporting.menu = new am4core.ExportMenu();
}
```
I have added the comments for the description. You can look at [reference](https://www.amcharts.com/docs/v4/reference/lineseries/) and [xy-chart](https://www.amcharts.com/docs/v4/chart-types/xy-chart/) for more details.
The function `createMultiLineChart` created here is the one which we are calling in `multi_line_chart.html.ep`.

Save it and refresh the home page. 
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/00r5jifchifb7p20yrld.PNG)
I have tried to use mostly the default configuration. The screenshot above is not doing the justice to the actual dynamic chart. For that you have to run and see it for yourself.

Now lets try to modify the `public/js/multi_line_chart.js` with some more configuration. As I mentioned before it is highly configurable and its difficult to cover each and every thing so I will try to cover whatever I can.
```javascript
function createSeries(chart, axis, field, name) {
    // Create series
    var series = chart.series.push(new am4charts.LineSeries());
    series.dataFields.dateX = "Date";
    series.dataFields.valueY = field;
    //series.dataFields.categoryX = "Date";
    series.strokeWidth = 2;
    series.xAxis = axis;
    series.name = name;
    series.tooltipText = "{name}: [bold]{valueY}[/]";
    //series.fillOpacity = 0.8;

    // For curvey lines
    series.tensionX = 0.8;
    series.tensionY = 1;

    // Multiple bullet options - circle, triangle, rectangle etc.
    var bullet = series.bullets.push(new am4charts.CircleBullet());
    bullet.fill = new am4core.InterfaceColorSet().getFor("background");
    bullet.fillOpacity = 1;
    bullet.strokeWidth = 2;
    bullet.circle.radius = 4;

    return series;
}

function createMultiLineChart(chartData) {
    // Themes begin
    am4core.useTheme(am4themes_animated);

    var chart = am4core.create("chartdiv", am4charts.XYChart);

    // Increase contrast by taking every second color
    chart.colors.step = 3;
    //chart.hiddenState.properties.opacity = 0; // this creates initial fade-in

    // Add title to chart
    var title = chart.titles.create();
    title.text = chartData["title"];
    title.fontSize = 25;
    title.marginBottom = 15;

    chart.data = chartData["data"];

    // Create axes - for normal Axis
    // var categoryAxis = chart.xAxes.push(new am4charts.CategoryAxis());
    // categoryAxis.dataFields.category = "Date";
    // categoryAxis.renderer.grid.template.location = 0;

    // Create axes - for Date Axis
    var dateAxis = chart.xAxes.push(new am4charts.DateAxis());
    //dateAxis.dataFields.category = "Date";
    dateAxis.renderer.grid.template.location = 0;
    dateAxis.renderer.minGridDistance = 50;
    dateAxis.title.text = chartData["label"]["domainAxis"];

    var valueAxis = chart.yAxes.push(new am4charts.ValueAxis());
    //valueAxis.renderer.line.strokeOpacity = 1;
    //valueAxis.renderer.line.strokeWidth = 2;
    valueAxis.title.text = chartData["label"]["rangeAxis"];

    var series1 = createSeries(chart, dateAxis, "Toyota", "Toyota");
    var series2 = createSeries(chart, dateAxis, "Ford", "Ford");
    var series3 = createSeries(chart, dateAxis, "Honda", "Honda");
    var series4 = createSeries(chart, dateAxis, "Renault", "Renault");

    // Add legend
    chart.legend = new am4charts.Legend();

    // Add cursor
    chart.cursor = new am4charts.XYCursor();
    chart.cursor.xAxis = dateAxis;

    // Add scrollbar
    chart.scrollbarX = new am4core.Scrollbar();

    // Add export menu
    chart.exporting.menu = new am4core.ExportMenu();
}
```

Now we will try to see the output again -
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fi7og88vjcasg4ishx45.PNG)
Somewhat better than the previous one. The three dots on the top right corner gives to more options to interact like - downloading the image as png or svg, getting the data in JSON or CSV format, printing the chart etc.
Also there are certain plugins available which you can use to enhance the experience. More details at [Plugins](https://www.amcharts.com/docs/v4/concepts/plugins/).

As I mentioned there are lot of config options and I haven't 
covered all of them. But I will try to cover it in my next installment where I will create the same chart in React.js using Typescript/ES6. Also the above js file can be modified a little bit to make it generalized for any type of multi line chart(especially the 'createSeries' call). I will leave that as an exercise.

The above example is available at [github](https://github.com/rai-gaurav/Charts/tree/main/amCharts).

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Mojolicious logo taken from [here](https://github.com/mojolicious/mojo/blob/master/lib/Mojolicious/resources/public/mojo/logo.png)
amCharts logo taken form [here](https://www.amcharts.com/about/press-kit/amcharts_light_transparent/)
