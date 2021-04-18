In my previous [article](https://dev.to/raigaurav/data-visualization-using-amcharts-with-perl-and-mojo-38ff), I talked about using [amCharts](https://www.amcharts.com/) library with Perl [Mojolicious](https://mojolicious.org/). Today we will looking at creating the similar chart with [React.js](https://reactjs.org/) instead of plain JavaScript. I will keep it short since we already talked about it previously and will be reusing most of the code.

There are 2 ways we can use the react.js -
1. Without JSX (using `<script>` tag)
2. With JSX

JSX stand for JavaScript XML. It allow you to easily write HTML in react.
For now we will take the baby step and start without JSX.

# Creating the data config
We will use the exact same example as in [previous](https://dev.to/raigaurav/data-visualization-using-amcharts-with-perl-and-mojo-38ff) article and try to create a multi line chart.
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

# Creating the mojo app 
The version I am using for this article is 9.14.
```shell
$  mojo generate app MojoReactApp
```
This command will generate a example application with proper directory structure for a MVC application and mentioned previously.

Now go inside the dir and try to run this app.
```shell
$ morbo ./script/mojo_app
Web application available at http://127.0.0.1:3000
```
Open the browser and hit http://localhost:3000/ and you can see the welcome page.

The rest of step is exact similar to mention in 'Creating the mojo app' section in [previous](https://dev.to/raigaurav/data-visualization-using-amcharts-with-perl-and-mojo-38ff) article. So I will not board you by repeating it again. We will directly see the react part.

# Adding React.js to app

We will update the `default.html.ep` to include the react.js
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
        <script src="https://unpkg.com/react@17/umd/react.production.min.js" crossorigin></script>
        <script src="https://unpkg.com/react-dom@17/umd/react-dom.production.min.js" crossorigin></script>

        %= content 'end'
    </body>
</html>
```
We are using the `production` minified version. You can also use the `development` version also for debugging purpose.
I have added react.js on layout template as we will be using it all our web pages.

In `multi_line_chart.html.ep`
```html
% layout 'default';
% title 'Charts';

% content_for 'head' => begin
    <link rel="stylesheet" type="text/css" href="css/charts.css">
% end

<div id="root"></div>

% content_for 'end' => begin
    %= javascript "https://cdn.amcharts.com/lib/4/core.js"
    %= javascript "https://cdn.amcharts.com/lib/4/charts.js"
    %= javascript "https://cdn.amcharts.com/lib/4/themes/animated.js"

    %= javascript "js/multi_line_chart.js"

    %= javascript begin 
        var domContainer = document.getElementById("root");
        createMultiLineChart(domContainer, <%== $chart_data %>);
    % end
% end
```
We are geeting the `$chart_data` form `create_multi_line_chart` in `lib\MojoReactApp\Controller\Charts.pm` when the template get rendered.
Lets update the `public/js/multi_line_chart.js` to make it a React Component.
```javascript
"use strict";

// React without JSX

const e = React.createElement;

class MultiLineChart extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
            chartId: this.props.chartId,
            chartData: this.props.data,
        };
    }

    createSeries = (chart, axis, field, name) => {
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
    };

    createChart = (chart) => {
        // Increase contrast by taking evey fouth color
        chart.colors.step = 4;
        //chart.hiddenState.properties.opacity = 0;             // this creates initial fade-in

        // Add title to chart
        var title = chart.titles.create();
        title.text = this.state.chartData["title"];
        title.fontSize = 25;
        title.marginBottom = 15;

        chart.data = this.state.chartData["data"];

        // Create axes - for normal Axis
        // var categoryAxis = chart.xAxes.push(new am4charts.CategoryAxis());
        // categoryAxis.dataFields.category = "Date";
        // categoryAxis.renderer.grid.template.location = 0;

        // Create axes - for Date Axis
        var dateAxis = chart.xAxes.push(new am4charts.DateAxis());
        //dateAxis.dataFields.category = "Date";
        dateAxis.renderer.grid.template.location = 0;
        dateAxis.renderer.minGridDistance = 50;
        dateAxis.title.text = this.state.chartData["label"]["domainAxis"];

        var valueAxis = chart.yAxes.push(new am4charts.ValueAxis());
        //valueAxis.renderer.line.strokeOpacity = 1;
        //valueAxis.renderer.line.strokeWidth = 2;
        valueAxis.title.text = this.state.chartData["label"]["rangeAxis"];

        //var single_data_item = this.state.chartData["data"][0];
        var series1 = this.createSeries(chart, dateAxis, "Toyota", "Toyota");
        var series2 = this.createSeries(chart, dateAxis, "Ford", "Ford");
        var series3 = this.createSeries(chart, dateAxis, "Honda", "Honda");
        var series4 = this.createSeries(chart, dateAxis, "Renault", "Renault");

        // Add legend
        chart.legend = new am4charts.Legend();

        // Add cursor
        chart.cursor = new am4charts.XYCursor();
        chart.cursor.xAxis = dateAxis;

        // Add scrollbar
        chart.scrollbarX = new am4core.Scrollbar();

        // Add export menu
        chart.exporting.menu = new am4core.ExportMenu();
    };

    componentDidMount() {
        am4core.useTheme(am4themes_animated);
        const chart = am4core.create(this.state.chartId, am4charts.XYChart);
        this.createChart(chart);
        this.chart = chart;
    }

    componentWillUnmount() {
        if (this.chart) {
            this.chart.dispose();
        }
    }

    render() {
        return e("div", { id: this.state.chartId }, null);
    }
}

function createMultiLineChart(domContainer, chartData) {
    ReactDOM.render(
        e(MultiLineChart, { chartId: "chartdiv", data: chartData }, null),
        domContainer
    );
}
```
We are calling the `createMultiLineChart` function from our template with parameters. The main point to know here is the state and lifecycle functions - `componentDidMount` and `componentWillUnmount`.
Since there are already plenty of documentation available, I encourage you to look into it. One of the place to learn the concept is react official docs- [State and Lifecycle](https://reactjs.org/docs/state-and-lifecycle.html)

If you look closely the rest of function definition is not much changes from the [previous](https://dev.to/raigaurav/data-visualization-using-amcharts-with-perl-and-mojo-38ff) used javascript. We just wrapped it in react.

The final directory structure is -
```
📦mojo_react_app
 ┣ 📂etc
 ┃ ┗ 📜input_data.json
 ┣ 📂lib
 ┃ ┣ 📂MojoReactApp
 ┃ ┃ ┣ 📂Controller
 ┃ ┃ ┃ ┗ 📜Charts.pm
 ┃ ┃ ┗ 📂Model
 ┃ ┃ ┃ ┗ 📜Data.pm
 ┃ ┗ 📜MojoReactApp.pm
 ┣ 📂public
 ┃ ┣ 📂css
 ┃ ┃ ┗ 📜charts.css
 ┃ ┗ 📂js
 ┃ ┃ ┗ 📜multi_line_chart.js
 ┣ 📂script
 ┃ ┗ 📜mojo_react_app
 ┣ 📂t
 ┃ ┗ 📜basic.t
 ┣ 📂templates
 ┃ ┣ 📂charts
 ┃ ┃ ┗ 📜multi_line_chart.html.ep
 ┃ ┗ 📂layouts
 ┃ ┃ ┗ 📜default.html.ep
 ┣ 📜mojo_react_app.yml
 ┗ 📜README.md
```
Save it and try to hit 'http://localhost:3000' again. From the user perspective side nothing has changed, you will see the same output as before.
![fi7og88vjcasg4ishx45](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c37jpdr8v8b6wnh0zzmh.PNG)

As I mentioned before we will starting with baby step.
The usage of the above mentioned way is very limited. You can use the react.js without jsx when your scope is small - where you have to make a website of few pages because here we are not using the full power of react.
To use the react.js with full potential and unleash its power you have to use jsx. We will be looking in to it our next article.

The above example is available at [github](https://github.com/rai-gaurav/mojo_react_app/tree/main/without_jsx).

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Mojolicious logo taken from [here](https://github.com/mojolicious/mojo/blob/master/lib/Mojolicious/resources/public/mojo/logo.png)
React logo taken from [here](https://reactjs.org/)
amCharts logo taken form [here](https://www.amcharts.com/about/press-kit/amcharts_light_transparent/)
