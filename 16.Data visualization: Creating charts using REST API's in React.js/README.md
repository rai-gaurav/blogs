In our [previous](https://dev.to/raigaurav/creating-rest-apis-with-perl-mojolicious-and-openapi-1bng) post of the series we have created the REST API's in [Mojolicious](https://mojolicious.org/)

The swagger ui is available on https://localhost/api (port:443) for development environment. If you look at the swagger ui mentioned [here] (https://dev.to/raigaurav/getting-ready-for-production-jio), we can see we have 2 API's endpoint 
1. `/api/v1/multi-line-chart`
2. `/api/v1/stacked-column-chart`

We will be querying these two endpoint in our react app.
Also I expect you to have a little understating in React.js.
So, lets get started.

# Install react and other dependencies
There are many place where you can get info about how to install react, so I will not go through the whole process in details.
1. Install Node.js from [official site](https://nodejs.org/en/)
2. Install create-react-app
```
npx create-react-app react-app
```
It will take some time. After installation is successful, you will see `react-app` dir created. Go inside it and run
```
npm start
```
It will open your default browser and you can see your home page on http://localhost:3000/.

We have to install some more dependencies.
We will add these to `package.json`. Here is the snapshot of mine.
```json
...
    "dependencies": {
        "mdbreact": "^4.27",
        "@amcharts/amcharts4": "^4.10.18",
        "react": "^16.13.1",
        "react-dom": "^16.13.1",
        "react-scripts": "^3.3.1"
    },
...
```
* We will be using [Material Design for Bootstrap](https://mdbootstrap.com/docs/) and there [MDBReact](https://mdbootstrap.com/docs/react/) for our react app. For now we will be using v4 as it is stable but they have v5 also release few month back.
* For charting we will be suing [AMcharts v4](https://www.amcharts.com/)
* The react version we are using is 16.13.1. The current version is 17.0.2. If you are writing any thing from scratch better to go ahead with newer version. My objective here is to show the chart usage and scope is very limited, hence using this version. Also you will creating function component instead of class component in newer version and a lot of complexity can be avoided.

After updating `package.json`, run
```
npm install
```
It will install all the dependencies in `node_modules`

Also our back-end server is running on https://localhost, we will add this to package.json so that we don't have to add the whole path in `fetch`.
```
{
...
   "proxy": "https://localhost",
...
}
```

# Modifying the application
We will be creating few trivial things which every website has - header, footer, body, different pages etc.
Before that we will be removing/modifying few items. If you look at the your dir structure various files and dir is already created by you.
* `index.html` is the entry point. Let update `index.js` which is actually doing all the work to
```js
import React from "react";
import ReactDOM from "react-dom";

import "@fortawesome/fontawesome-free/css/all.min.css";
import "bootstrap-css-only/css/bootstrap.min.css";
import "mdbreact/dist/css/mdb.css";

import ReactApp from "./ReactApp";

ReactDOM.render(
    <React.StrictMode>
        <ReactApp />
    </React.StrictMode>,
    document.getElementById("root")
);
``` 
Here I have imported `mdb` and other dependencies. I have also renamed the `App.js` to `ReactApp.js` and included that.

## Creating header
We will creating a component in `react-app\src\components\layouts\Header.jsx`. We will be using [Bootstrap Navbar](https://mdbootstrap.com/docs/react/navigation/navbar/) for it where we ill be creating navigation for different pages.
```js
import React, { Component } from "react";
import {
    MDBNavbar,
    MDBNavbarBrand,
    MDBNavbarNav,
    MDBNavbarToggler,
    MDBCollapse,
    MDBNavItem,
    MDBNavLink,
} from "mdbreact";
import { withRouter } from "react-router";

class Header extends Component {
    constructor(props) {
        super(props);
        this.state = {
            collapse: false,
        };
        this.onClick = this.onClick.bind(this);
    }

    onClick() {
        this.setState({
            collapse: !this.state.collapse,
        });
    }

    render() {
        return (
            <React.Fragment>
                <header>
                    <MDBNavbar color="default-color" dark expand="md" scrolling fixed="top">
                        <MDBNavbarBrand href="/">
                            <strong>Mojo React App</strong>
                        </MDBNavbarBrand>
                        <MDBNavbarToggler onClick={this.onClick} />
                        <MDBCollapse isOpen={this.state.collapse} navbar>
                            <MDBNavbarNav left>
                                <MDBNavItem active={this.props.location.pathname === "/"}>
                                    <MDBNavLink to="/">Home</MDBNavLink>
                                </MDBNavItem>
                                <MDBNavItem active={this.props.location.pathname === "/chart1"}>
                                    <MDBNavLink to="/chart1">LineChart</MDBNavLink>
                                </MDBNavItem>
                                <MDBNavItem active={this.props.location.pathname === "/chart2"}>
                                    <MDBNavLink to="/chart2">ColumnChart</MDBNavLink>
                                </MDBNavItem>
                            </MDBNavbarNav>
                        </MDBCollapse>
                    </MDBNavbar>
                </header>
            </React.Fragment>
        );
    }
}

export default withRouter(Header);

```
We will be changing the tab highlight based on the `this.props.location.pathname` value which will be passed from parent component.
This will create a header similar to
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aurs9n280sbry1fb16e5.JPG)

## Creating footer
Create a component in `react-app\src\components\layouts\Footer.jsx`. We will be using [Bootstrap footer](https://mdbootstrap.com/docs/react/navigation/footer/) and modifying it as per our need.
```js
import React, { Component } from "react";
import { MDBContainer, MDBFooter } from "mdbreact";

class Footer extends Component {
    render() {
        return (
            <MDBFooter color="default-color" className="font-small pt-4 mt-4">
                <div className="text-center py-3">
                    <MDBContainer fluid className="text-center">
                        <a href="/">Home</a> | <a href="/chart1">LineChart</a>| <a href="/chart2">ColumnChart</a>
                    </MDBContainer>
                </div>
                <div className="footer-copyright text-center py-3">
                    <MDBContainer fluid>
                        &copy; {new Date().getFullYear()} Copyright:{" "}
                        <a href="https://www.mdbootstrap.com"> MDBootstrap.com </a>
                    </MDBContainer>
                </div>
            </MDBFooter>
        );
    }
}

export default Footer;

```

## Creating Home page.
Lets create a small home landing page. Inside `react-app\src\components\Home.jsx`
```js
import React, { Component } from "react";

class Home extends Component {
    render() {
        return (
            <React.Fragment>
                <h2>This is home page</h2>
                <h5>Welcome to Mojolicious React application</h5>
            </React.Fragment>
        );
    }
}

export default Home;

```
Simple. Also lets update our `ReactApp.js`(renamed from App.js) and `ReactApp.css`(renamed from App.css) to inculded the newly created header and footer.
```js
import React, { Component } from "react";
import { BrowserRouter, Route, Switch } from "react-router-dom";
import "./ReactApp.css";

import Header from "./components/layouts/Header";
import Footer from "./components/layouts/Footer";
import Home from "./components/Home";
import { MDBContainer } from "mdbreact";

class ReactApp extends Component {
    render() {
        return (
            <React.Fragment>
                <BrowserRouter>
                    <Header location={this.props.location} />
                    <main className="site-content">
                        <MDBContainer className="text-center my-5">
                            <Switch>
                                <Route exact path="/" component={Home} />
                                {/* <Route exact path="/chart1" component={Chart1} />
                                <Route exact path="/chart2" component={Chart2} /> */}
                            </Switch>
                        </MDBContainer>
                    </main>
                    <Footer />
                </BrowserRouter>
            </React.Fragment>
        );
    }
}

export default ReactApp;

```
* I have commented the charting components as we haven't created those now.
* We have imported the `Header` and `Footer` components and on request of `/` we are rendering the `Home`component.
* There are certain keywords here which has special meaning in react(e.g. `Switch` etc.). I encourage you to look at official react document to understand them.
* If you look, closely we have created our web page skeleton her. Inside `BrowserRouter` tag you can see - `Header` at top, `main` content in the middle and `Footer` at bottom.

In `ReactApp.css`
```css
.site-content {
    padding-top: 25px;
}
```
Let run this and see it in action.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1twjga39brfk37tlwvfx.JPG)

So far so good.

## Creating charts
Now lets try to create the chart components and uncomment those lines in `ReactApp.js`
We will be creating 2 charts for 2 API endpoints on 2 separate pages.

### Chart1
I am using this name but better to use some meaningful name here.
Inside `react-app\src\components\Chart1.jsx`
```js
import React, { Component } from "react";
import LineChart from "./Charts/LineChart";

class Chart1 extends Component {
    constructor(props) {
        super();
        this.state = {
            error: null,
            isLoaded: false,
            chartData: [],
        };
    }
    getChartData = () => {
        fetch("/api/v1/multi-line-chart")
            .then((response) => response.json())
            .then(
                (result) => {
                    this.setState({
                        isLoaded: true,
                        chartData: result.chart_data,
                    });
                },
                (error) => {
                    this.setState({
                        isLoaded: true,
                        error,
                    });
                }
            );
    };

    componentDidMount() {
        this.getChartData();
    }
    render() {
        if (this.state.error) {
            return <div>Error: {this.state.error.message}</div>;
        } else if (!this.state.isLoaded) {
            return (
                <div className="spinner-border" role="status">
                    <span className="sr-only">Loading...</span>
                </div>
            );
        } else {
            return (
                <React.Fragment>
                    <LineChart
                        chartId="chart1"
                        data={this.state.chartData.data}
                        axisNames={{
                            xAxis: [this.state.chartData.label.domainAxis],
                            yAxis: [this.state.chartData.label.rangeAxis],
                        }}
                        lineForXAxis="Date"
                        linesForFirstAxis={["Ford", "Honda", "Renault", "Toyota"]}
                        chartTitle={this.state.chartData.title}
                    />
                </React.Fragment>
            );
        }
    }
}

export default Chart1;

```
* The above code is similar to what available on [AJAX and APIs](https://reactjs.org/docs/faq-ajax.html) section on react doc.
* We are querying our API endpoint `/api/v1/multi-line-chart` which will return the JSON response which we will be passing to `LineChart` component for creating multi-line chart.
* During the process of request and getting the response we will be using a `Loading` spinner.
* In case of any error in response the same will be available on UI.
* The one thing which is of interest is `LineChart` component. I have created this component some time back and the objective of this article is to showcase its power. This component is created in such a way you can create a **single line chart**, **a multi-line chart** or a **multi-axle chart**. You can also create **percentage chart**. It doesn't matter whether your x-axis is Date axis or not it will work for both. Just pass the parameter in props, and will be create the chart based on it it on fly. We will look into it. The `LineChart` component provide you with the layer of abstraction and it can act as a base component for all your Line charts. 

### Chart2
Inside `react-app\src\components\Chart2.jsx`
```js
import React, { Component } from "react";
import StackedClusteredColumnChart from "./Charts/StackedClusteredColumnChart";

class Chart2 extends Component {
    constructor(props) {
        super();
        this.state = {
            error: null,
            isLoaded: false,
            chartData: [],
        };
    }
    getChartData = () => {
        fetch("/api/v1/stacked-column-chart")
            .then((response) => response.json())
            .then(
                (result) => {
                    this.setState({
                        isLoaded: true,
                        chartData: result.chart_data,
                    });
                },
                (error) => {
                    this.setState({
                        isLoaded: true,
                        error,
                    });
                }
            );
    };

    componentDidMount() {
        this.getChartData();
    }
    render() {
        if (this.state.error) {
            return <div>Error: {this.state.error.message}</div>;
        } else if (!this.state.isLoaded) {
            return (
                <div className="spinner-border" role="status">
                    <span className="sr-only">Loading...</span>
                </div>
            );
        } else {
            return (
                <React.Fragment>
                    <StackedClusteredColumnChart
                        chartId="chart2"
                        data={this.state.chartData.data}
                        axisNames={{
                            xAxis: [this.state.chartData.label.domainAxis],
                            yAxis: [this.state.chartData.label.rangeAxis],
                        }}
                        columnForXAxis="Year"
                        columnsForYAxis={["Africa", "America", "Antartica", "Asia", "Australia", "Europe"]}
                        chartTitle={this.state.chartData.title}
                    />
                </React.Fragment>
            );
        }
    }
}

export default Chart2;

```
* We are querying our API endpoint `/api/v1/stacked-column-chart` which will return the JSON response which we will be passing to `StackedClusteredColumnChart` component for creating column chart.
* Again this is similar to `LineChart` component and powerful too.
Just pass the proper params in props and it will do all the work for you.

Before creating the line and column chart component let update the `ReactApp.css` for loading spinner and chart css
```css
.site-content {
    padding-top: 25px;
}

.chart-display {
    width: 1000px;
    height: 500px;
}

.loader {
    border: 16px solid #f3f3f3;
    border-top: 16px solid #3498db;
    border-radius: 50%;
    width: 120px;
    height: 120px;
    animation: spin 2s linear infinite;
}

@keyframes spin {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}

```

### Creating LineChart.jsx
This is quite a big component.
Amcharts comes with lot of good example and documentation. I encourage you to look at the [series](https://www.amcharts.com/docs/v4/concepts/series/) doc and [multi-axes](https://www.amcharts.com/demos/multiple-value-axes/) example to understand more. I have modifies those default configurations and used it as per my need. Each of these are covered in there documentation. I have also added comments in between for understanding.

Inside `react-app\src\components\Charts\LineChart.jsx`
```js
import React, { Component } from "react";
import * as am4core from "@amcharts/amcharts4/core";
import * as am4charts from "@amcharts/amcharts4/charts";
import am4themes_animated from "@amcharts/amcharts4/themes/animated";

class LineChart extends Component {
    constructor(props) {
        super(props);
        this.state = {
            chartId: this.props.chartId,
            chartdata: this.props.data,
            axisNames: this.props.axisNames,
            lineForXAxis: this.props.lineForXAxis,
            linesForFirstAxis: this.props.linesForFirstAxis,
            linesForSecondAxis: this.props.linesForSecondAxis
                ? this.props.linesForSecondAxis
                : null,
            legendNames: this.props.legendNames
                ? this.props.legendNames
                : this.props.linesForFirstAxis.concat(this.props.linesForSecondAxis),
            isPercentageChart: this.props.isPercentageChart ? true : false,
            isDateAxis: this.props.isDateAxis ? true : false,
        };
    }

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

    createDateAxis = (chart, xAxisName) => {
        let dateAxis = chart.xAxes.push(new am4charts.DateAxis());
        dateAxis.title.text = xAxisName;
        dateAxis.baseInterval.timeUnit = "minute";
        dateAxis.baseInterval.count = 1;
        let axisTooltip = dateAxis.tooltip;
        axisTooltip.background.strokeWidth = 0;
        axisTooltip.background.cornerRadius = 3;
        axisTooltip.background.pointerLength = 0;
        axisTooltip.dy = 5;
        dateAxis.tooltipDateFormat = "MMM dd HH:mm:ss";
        dateAxis.cursorTooltipEnabled = true;
        //dateAxis.renderer.minGridDistance = 50;
        //dateAxis.renderer.grid.template.disabled = true;
        dateAxis.renderer.line.strokeOpacity = 1;
        dateAxis.renderer.line.strokeWidth = 2;
        dateAxis.skipEmptyPeriods = true;
        return dateAxis;
    };
    createCategoryAxis = (chart, xAxisName) => {
        let categoryAxis = chart.xAxes.push(new am4charts.CategoryAxis());
        categoryAxis.dataFields.category = this.state.lineForXAxis;
        categoryAxis.title.text = xAxisName;

        categoryAxis.renderer.grid.template.location = 0;
        categoryAxis.renderer.minGridDistance = 20;
        categoryAxis.renderer.cellStartLocation = 0.1;
        categoryAxis.renderer.cellEndLocation = 0.9;
        return categoryAxis;
    };
    createValueAxisRange = (valueAxis, value, color, guideLabel) => {
        let axisRange = valueAxis.axisRanges.create();
        axisRange.value = value;
        axisRange.grid.stroke = am4core.color(color);
        axisRange.grid.strokeOpacity = 0.7;
        // https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute/stroke-dasharray
        axisRange.grid.strokeDasharray = "4 5";
        axisRange.grid.opacity = 0.8;
        axisRange.grid.strokeWidth = 2;
        axisRange.label.inside = true;
        axisRange.label.text = guideLabel;
        axisRange.label.fill = axisRange.grid.stroke;
        axisRange.label.verticalCenter = "bottom";
        axisRange.label.horizontalCenter = "middle";
        return axisRange;
    };
    createValueAxis = (chart, yAxisName, opposite) => {
        let valueAxis = chart.yAxes.push(new am4charts.ValueAxis());
        valueAxis.title.text = yAxisName;
        valueAxis.min = 0;
        valueAxis.ghostLabel.disabled = true;
        valueAxis.extraMax = 0.1;
        valueAxis.numberFormatter = new am4core.NumberFormatter();
        valueAxis.numberFormatter.numberFormat = "# a";
        if (typeof opposite !== "undefined") {
            valueAxis.renderer.opposite = opposite;
        }
        if (this.state.linesForSecondAxis) {
            valueAxis.renderer.grid.template.disabled = true;
        }
        valueAxis.renderer.line.strokeOpacity = 1;
        valueAxis.renderer.line.strokeWidth = 2;
        valueAxis.renderer.ticks.template.disabled = false;
        valueAxis.renderer.ticks.template.strokeOpacity = 1;
        valueAxis.renderer.ticks.template.strokeWidth = 2;
        return valueAxis;
    };

    createAxis = (chart, xAxisName, yAxisName) => {
        // Create x-axes
        let xAxis;
        if (this.state.isDateAxis) {
            xAxis = this.createDateAxis(chart, xAxisName);
        } else {
            xAxis = this.createCategoryAxis(chart, xAxisName);
        }
        // Create y-axes
        let valueAxis = this.createValueAxis(chart, yAxisName);
        if (this.state.isPercentageChart) {
            // This is to create horizontal 'red' (on 80%) and 'green'(on 100%) lines
            this.createValueAxisRange(valueAxis, 80, "#ff0000", "Threshold");
            this.createValueAxisRange(valueAxis, 100, "#00b33c", "Goal");
        }
        return [xAxis, valueAxis];
    };

    createTrendLine = (chart, value, name, yAxisId, bulletType, fillOpacity) => {
        let series = chart.series.push(new am4charts.LineSeries());
        series.name = name;
        series.dataFields.valueY = value;
        if (this.state.isDateAxis) {
            series.dataFields.dateX = this.state.lineForXAxis;
        } else {
            series.dataFields.categoryX = this.state.lineForXAxis;
        }
        series.strokeWidth = 2;
        series.strokeOpacity = 0.8;
        series.tensionX = 0.7;
        series.yAxis = yAxisId;
        series.fillOpacity = fillOpacity;
        if (this.state.isPercentageChart) {
            series.tooltipText = "{name}: [bold]{valueY}%[/]";
        } else {
            series.tooltipText = "{name}: [bold]{valueY}[/]";
        }
        series.tooltip.background.cornerRadius = 13;
        series.tooltip.background.fillOpacity = 0.8;
        series.tooltip.exportable = false;
        series.minBulletDistance = 15;
        // Enable the number in the legend on hovering over the graph
        if (this.state.isPercentageChart) {
            series.legendSettings.itemValueText = "[bold]{valueY}%[/]";
            series.legendSettings.valueText =
                "(Avg: [bold]{valueY.average.formatNumber('#.##')}%[/])";
        } else {
            series.legendSettings.itemValueText = "[bold]{valueY}[/]";
        }
        // Add a drop shadow filter on columns
        //let shadow = series.filters.push(new am4core.DropShadowFilter());
        //shadow.dx = 10;
        //shadow.dy = 10;
        //shadow.blur = 5;
        let bullet;
        let hoverState;
        switch (bulletType) {
            case "rectangle":
                bullet = series.bullets.push(new am4charts.Bullet());
                let square = bullet.createChild(am4core.Rectangle);
                square.strokeWidth = 1;
                square.width = 7;
                square.height = 7;
                square.stroke = am4core.color("#fff");
                square.horizontalCenter = "middle";
                square.verticalCenter = "middle";
                hoverState = square.states.create("hover");
                hoverState.properties.scale = 1.7;
                break;
            case "triangledown":
            case "triangleup":
                bullet = series.bullets.push(new am4charts.Bullet());
                let triangle = bullet.createChild(am4core.Triangle);
                triangle.strokeWidth = 1;
                triangle.width = 7;
                triangle.height = 7;
                if (bulletType === "triangleup") {
                    triangle.direction = "top";
                } else {
                    triangle.direction = "bottom";
                }
                triangle.stroke = am4core.color("#fff");
                triangle.horizontalCenter = "middle";
                triangle.verticalCenter = "middle";
                hoverState = triangle.states.create("hover");
                hoverState.properties.scale = 1.7;
                break;
            case "circle":
            case "hollowcircle":
                bullet = series.bullets.push(new am4charts.CircleBullet());
                bullet.strokeWidth = 1;
                bullet.circle.radius = 3.5;
                bullet.fillOpacity = 1;
                if (bulletType === "circle") {
                    bullet.stroke = am4core.color("#fff");
                    bullet.circle.fill = series.stroke;
                } else {
                    bullet.stroke = series.stroke;
                    bullet.circle.fill = am4core.color("#fff");
                }
                hoverState = bullet.states.create("hover");
                hoverState.properties.scale = 1.7;
                break;
            default:
                break;
        }
        this.addEvents(series);
        return series;
    };
    addEvents = (series) => {
        // Enable interactions on series segments
        let segment = series.segments.template;
        segment.interactionsEnabled = true;

        // Create hover state
        let hoverState = segment.states.create("hover");
        hoverState.properties.strokeWidth = 4;
        hoverState.properties.strokeOpacity = 1;
    };
    createLegend = (chart) => {
        chart.legend = new am4charts.Legend();
        chart.legend.maxWidth = 400;
        chart.legend.markers.template.width = 40;
        chart.legend.markers.template.height = 10;
        // Use this to change the color of the legend label
        //chart.legend.markers.template.disabled = true;
        //chart.legend.labels.template.text = "[bold {color}]{name}[/]";
        chart.legend.itemContainers.template.paddingTop = 2;
        chart.legend.itemContainers.template.paddingBottom = 2;
        chart.legend.labels.template.maxWidth = 130;
        chart.legend.labels.template.truncate = true;
        chart.legend.itemContainers.template.tooltipText = "{name}";
        chart.legend.numberFormatter = new am4core.NumberFormatter();
        chart.legend.numberFormatter.numberFormat = "#.## a";
        chart.legend.itemContainers.template.events.on("over", (ev) => {
            let lineSeries = ev.target.dataItem.dataContext.segments.template;
            lineSeries.strokeOpacity = 1;
            lineSeries.strokeWidth = 4;
        });
        chart.legend.itemContainers.template.events.on("out", function (ev) {
            let lineSeries = ev.target.dataItem.dataContext.segments.template;
            lineSeries.strokeOpacity = 0.8;
            lineSeries.strokeWidth = 2;
        });
        chart.legend.valueLabels.template.adapter.add("textOutput", function (text, target) {
            if (text === "(Avg: [bold]%[/])" || text === "(Total: [bold][/])") {
                return "N/A";
            } else if (text === "[bold]%[/]" || text === "[bold][/]") {
                return "";
            }
            return text;
        });
    };

    createExportMenu = (chart, title) => {
        chart.exporting.menu = new am4core.ExportMenu();
        chart.exporting.menu.verticalAlign = "bottom";
        chart.exporting.filePrefix = title + " LineChart";
    };

    createCursor = (chart) => {
        chart.cursor = new am4charts.XYCursor();
    };

    createScrollBar = (chart, series) => {
        chart.scrollbarX = new am4core.Scrollbar();
        chart.scrollbarX.thumb.background.fill = am4core.color("#66c9ff");
        chart.scrollbarX.startGrip.background.fill = am4core.color("#0095e6");
        chart.scrollbarX.endGrip.background.fill = am4core.color("#0095e6");
        chart.scrollbarX.stroke = am4core.color("#66c9ff");
        chart.scrollbarX.height = "20";
        chart.scrollbarX.exportable = false;
        // Add simple vertical scrollbar
        // chart.scrollbarY = new am4core.Scrollbar();
        // chart.scrollbarY.thumb.background.fill = am4core.color("#66c9ff");
        // chart.scrollbarY.startGrip.background.fill = am4core.color("#0095e6");
        // chart.scrollbarY.endGrip.background.fill = am4core.color("#0095e6");
        // chart.scrollbarY.stroke = am4core.color("#66c9ff");
        // chart.scrollbarY.width = "20";
        // chart.scrollbarY.exportable = false;
    };

    addChartTitle = (chart, titleText) => {
        let title = chart.titles.create();
        title.text = titleText;
        title.fontSize = 25;
        title.marginBottom = 30;
    };

    createChart = (chart) => {
        chart.data = this.state.chartdata;
        chart.colors.step = 4;
        // This will change the background color of chart
        //chart.background.fill = "#fff";
        //chart.background.opacity = 0.5;
        this.createLegend(chart);

        this.createCursor(chart);

        // Use this to change bullet type in lines if needed
        //let bulletsType = ["circle", "triangleup", "triangledown", "hollowcircle", "rectangle"];
        let axis = this.createAxis(
            chart,
            this.state.axisNames.xAxis[0],
            this.state.axisNames.yAxis[0]
        );
        for (let i = 0; i < this.state.linesForFirstAxis.length; i++) {
            //if (typeof bulletsType[i] !== "undefined") {
            this.createTrendLine(
                chart,
                this.state.linesForFirstAxis[i],
                this.state.legendNames[i],
                axis[1],
                "circle"
            );
            //} else {
            //    this.createTrendLine(chart, this.state.linesForFirstAxis[i], axis[1]);
            //}
        }

        if (this.state.linesForSecondAxis) {
            let yAxis = this.createValueAxis(chart, this.state.axisNames.yAxis[1], "true");
            for (let i = 0; i < this.state.linesForSecondAxis.length; i++) {
                let series;
                let fillOpacity = 0.2;
                //if (typeof bulletsType[this.state.linesForSecondAxis.length - i] !== "undefined") {
                series = this.createTrendLine(
                    chart,
                    this.state.linesForSecondAxis[i],
                    this.state.legendNames[this.state.linesForFirstAxis.length + i],
                    yAxis,
                    "circle",
                    fillOpacity
                );
                //} else {
                //    series = this.createTrendLine(chart, this.state.linesForSecondAxis[i], yAxis);
                //}
                if (this.state.linesForSecondAxis.length === 1) {
                    yAxis.renderer.line.stroke = series.stroke;
                    yAxis.renderer.ticks.template.stroke = series.stroke;
                }
            }
        }
        this.createScrollBar(chart);
        if (this.props.chartTitle) {
            this.addChartTitle(chart, this.props.chartTitle);
            this.createExportMenu(chart, this.props.chartTitle);
        } else {
            this.createExportMenu(chart, "");
        }
    };

    componentDidUpdate(prevProps) {
        if (this.chart !== null) {
            if (JSON.stringify(prevProps.data) !== JSON.stringify(this.props.data)) {
                this.chart.data = this.props.data;
            }
        }
    }

    render() {
        return (
            <div>
                <div id={this.state.chartId} className="chart-display" />
            </div>
        );
    }
}

export default LineChart;
```

## Creating StackedClusteredColumnChart.jsx
Again please have a look at amcharts doc and demo for more understanding. For starter you can look at this [example] (https://www.amcharts.com/demos/stacked-clustered-column-chart/)
Inside `react-app\src\components\Charts\StackedClusteredColumnChart.jsx`
```js
import React, { Component } from "react";
import * as am4core from "@amcharts/amcharts4/core";
import * as am4charts from "@amcharts/amcharts4/charts";
import am4themes_animated from "@amcharts/amcharts4/themes/animated";

class StackedClusteredColumnChart extends Component {
    constructor(props) {
        super(props);
        this.state = {
            chartId: this.props.chartId,
            chartdata: this.props.data,
            axisNames: this.props.axisNames,
            columnForXAxis: this.props.columnForXAxis,
            columnsForYAxis: this.props.columnsForYAxis,
            legendNames: this.props.legendNames
                ? this.props.legendNames
                : this.props.columnsForYAxis,
            showDummyData: this.props.showDummyData ? true : false,
            isPercentageChart: this.props.isPercentageChart ? true : false,
            isDateAxis: this.props.isDateAxis ? true : false,
        };
    }
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
    getLinearGradientModifier = () => {
        // Adding greadient to create a round bar effect
        let fillModifier = new am4core.LinearGradientModifier();
        fillModifier.brightnesses = [0, 1, 1, 0];
        fillModifier.offsets = [0, 0.45, 0.55, 1];
        fillModifier.gradient.rotation = 0;
        return fillModifier;
    };
    getLinearGradient = (color1, color2) => {
        let gradient = new am4core.LinearGradient();
        gradient.addColor(color1);
        if (typeof color2 !== "undefined") {
            gradient.addColor(color2);
        } else {
            gradient.addColor("#66c9ff");
            gradient.addColor(color1);
        }
        gradient.rotation = 90;
        return gradient;
    };
    createLegend = (chart) => {
        chart.legend = new am4charts.Legend();
        chart.legend.maxWidth = 400;
        chart.legend.markers.template.width = 20;
        chart.legend.markers.template.height = 20;
        chart.legend.itemContainers.template.paddingRight = 2;
        chart.legend.itemContainers.template.paddingLeft = 2;
        chart.legend.labels.template.maxWidth = 100;
        chart.legend.labels.template.truncate = true;
        chart.legend.valueLabels.template.align = "left";
        chart.legend.valueLabels.template.textAlign = "end";
        chart.legend.itemContainers.template.tooltipText = "{name}";

        chart.legend.itemContainers.template.events.on("over", (ev) => {
            let seriesColumn = ev.target.dataItem.dataContext.columns.template;
            seriesColumn.fillOpacity = 1;
        });
        chart.legend.itemContainers.template.events.on("out", function (ev) {
            let seriesColumn = ev.target.dataItem.dataContext.columns.template;
            seriesColumn.fillOpacity = 0.7;
        });
        chart.legend.valueLabels.template.adapter.add("textOutput", function (text, target) {
            if (text === "(Avg: [bold]%[/])" || text === "(Total: [bold][/])") {
                return "N/A";
            } else if (text === "[bold]%[/]" || text === "[bold][/]") {
                return "";
            }
            return text;
        });
    };
    createScrollBar = (chart) => {
        chart.scrollbarX = new am4core.Scrollbar();
        chart.scrollbarX.background.fillOpacity = 0.7;

        let gradient = this.getLinearGradient("#0095e6");
        chart.scrollbarX.thumb.background.fill = gradient;
        chart.scrollbarX.thumb.background.fillOpacity = 0.7;
        chart.scrollbarX.startGrip.background.fill = am4core.color("#0095e6");
        chart.scrollbarX.endGrip.background.fill = am4core.color("#0095e6");
        chart.scrollbarX.stroke = am4core.color("#66c9ff");
        chart.scrollbarX.height = "20";
        chart.scrollbarX.exportable = false;
    };
    createExportMenu = (chart, title) => {
        chart.exporting.menu = new am4core.ExportMenu();
        chart.exporting.menu.verticalAlign = "bottom";
        chart.exporting.filePrefix = title + " StackedColumnChart";
    };
    createCursor = (chart) => {
        chart.cursor = new am4charts.XYCursor();
    };
    createDateAxis = (chart, xAxisName) => {
        let dateAxis = chart.xAxes.push(new am4charts.DateAxis());
        dateAxis.title.text = xAxisName;
        dateAxis.cursorTooltipEnabled = true;
        dateAxis.renderer.minGridDistance = 30;
        dateAxis.renderer.cellStartLocation = 0.1;
        dateAxis.renderer.cellEndLocation = 0.9;
        dateAxis.skipEmptyPeriods = true;
        dateAxis.renderer.grid.template.location = 0;
        dateAxis.renderer.axisFills.template.disabled = false;
        dateAxis.renderer.axisFills.template.fill = am4core.color("#b3b3b3");
        dateAxis.renderer.axisFills.template.fillOpacity = 0.2;
        return dateAxis;
    };
    createCategoryAxis = (chart, xAxisName) => {
        let categoryAxis = chart.xAxes.push(new am4charts.CategoryAxis());
        categoryAxis.dataFields.category = this.state.columnForXAxis;

        categoryAxis.title.text = xAxisName;
        categoryAxis.renderer.grid.template.location = 0;
        categoryAxis.renderer.minGridDistance = 20;
        categoryAxis.renderer.cellStartLocation = 0.1;
        categoryAxis.renderer.cellEndLocation = 0.9;
        categoryAxis.renderer.axisFills.template.disabled = false;
        categoryAxis.renderer.axisFills.template.fillOpacity = 0.2;
        categoryAxis.renderer.axisFills.template.fill = am4core.color("#b3b3b3");
        return categoryAxis;
    };
    createValueAxis = (chart, yAxisName) => {
        let valueAxis = chart.yAxes.push(new am4charts.ValueAxis());
        valueAxis.title.text = yAxisName;
        valueAxis.min = 0;
        valueAxis.ghostLabel.disabled = true;
        valueAxis.extraMax = 0.1;
        valueAxis.renderer.line.strokeOpacity = 1;
        valueAxis.renderer.line.strokeWidth = 2;
        valueAxis.renderer.ticks.template.disabled = false;
        valueAxis.renderer.ticks.template.strokeOpacity = 1;
        valueAxis.renderer.ticks.template.strokeWidth = 2;
        return valueAxis;
    };
    createValueAxisRange = (valueAxis, value, color, guideLabel) => {
        let axisRange = valueAxis.axisRanges.create();
        axisRange.value = value;
        axisRange.grid.stroke = am4core.color(color);
        axisRange.grid.strokeOpacity = 0.7;
        // https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute/stroke-dasharray
        axisRange.grid.strokeDasharray = "4 5";
        axisRange.grid.opacity = 0.8;
        axisRange.grid.strokeWidth = 2;
        axisRange.label.inside = true;
        axisRange.label.text = guideLabel;
        axisRange.label.fill = axisRange.grid.stroke;
        axisRange.label.verticalCenter = "bottom";
        axisRange.label.horizontalCenter = "middle";
        return axisRange;
    };

    createAxis = (chart, xAxisName, yAxisName) => {
        // Create x-axes
        let xAxis;
        if (this.state.isDateAxis) {
            xAxis = this.createDateAxis(chart, xAxisName);
        } else {
            xAxis = this.createCategoryAxis(chart, xAxisName);
        }
        // Create y-axes
        let valueAxis = this.createValueAxis(chart, yAxisName);
        if (this.state.isPercentageChart) {
            // This is to create horizontal 'red' (on 80%) and 'green'(on 100%) lines
            this.createValueAxisRange(valueAxis, 80, "#ff0000", "Threshold");
            this.createValueAxisRange(valueAxis, 100, "#00b33c", "Goal");
        }
        return [xAxis, valueAxis];
    };

    createSeries = (chart, field, name, stacked, showDummyData) => {
        // For normal coloums
        let series = chart.series.push(new am4charts.ColumnSeries());
        // For 3D coloums
        //let series = chart.series.push(new am4charts.ColumnSeries3D());
        series.name = name;
        series.dataFields.valueY = field;
        if (this.state.isDateAxis) {
            series.dataFields.dateX = this.state.columnForXAxis;
        } else {
            series.dataFields.categoryX = this.state.columnForXAxis;
        }
        if (showDummyData && !this.state.isPercentageChart) {
            series.columns.template.propertyFields.dummyData = field + "_breakdown";
            series.columns.template.tooltipText =
                "[bold]{name} #{categoryX}\n[bold]Total:[/] {valueY}\n[#00cc44 bold]Pass:[/] {dummyData.pass}\n[#ff0000 bold]Fail:[/] {dummyData.fail}\n[#ff471a bold]Error:[/] {dummyData.error}\n[#ff9900 bold]Terminated:[/] {dummyData.terminated}[/]";
        } else if (this.state.isPercentageChart) {
            series.columns.template.tooltipText = "{name}: [bold]{valueY}%[/]";
        } else {
            series.columns.template.tooltipText = "{name}: [bold]{valueY}[/]";
        }
        series.strokeWidth = 2;
        series.tooltip.background.fillOpacity = 0.9;
        series.tooltip.exportable = false;
        series.stacked = stacked;
        series.columns.template.width = am4core.percent(90);
        series.columns.template.fillOpacity = 0.7;
        series.tooltip.getFillFromObject = false;
        series.tooltip.background.fill = am4core.color("#ffffff");
        series.tooltip.background.stroke = chart.colors.getIndex(
            chart.colors.currentStep - chart.colors.step
        );
        series.tooltip.background.strokeWidth = 2;
        series.tooltip.label.fill = am4core.color("#000000");

        let fillModifier = this.getLinearGradientModifier();
        series.columns.template.fillModifier = fillModifier;
        if (this.state.isPercentageChart) {
            series.legendSettings.itemValueText = "[bold]{valueY}%[/]";
            series.legendSettings.valueText =
                "(Avg: [bold]{valueY.average.formatNumber('#.##')}%[/])";
        } else {
            series.legendSettings.itemValueText = "[bold]{valueY}[/]";
            series.legendSettings.valueText = "(Total: [bold]{valueY.sum.formatNumber('#.')}[/])";
        }
        series.cursorTooltipEnabled = false;
        this.addEvents(series);
    };

    addChartTitle = (chart, titleText) => {
        let title = chart.titles.create();
        title.text = titleText;
        title.fontSize = 25;
        title.marginBottom = 30;
    };

    addEvents = (series) => {
        let hoverState = series.columns.template.states.create("hover");
        hoverState.properties.fillOpacity = 1;
    };

    preZoomChart = (chart, xAxis) => {
        chart.events.on("ready", (a) => {
            // different zoom methods can be used - zoomToIndexes, zoomToDates, zoomToValues
            if (this.state.isDateAxis) {
                xAxis.start = 0.4;
                xAxis.end = 1;
            } else {
                xAxis.zoomToIndexes(chart.data.length - 9, chart.data.length, false, true, true);
            }
        });
    };

    createChart = (chart) => {
        chart.data = this.state.chartdata;
        chart.colors.step = 3;
        if (this.props.isDateAxis) {
            chart.dateFormatter.inputDateFormat = "yyyy-MM-ddThh";
        }
        this.createLegend(chart);
        this.createCursor(chart);
        // Fow now its single axis hence '0'
        let axis = this.createAxis(
            chart,
            this.state.axisNames.xAxis[0],
            this.state.axisNames.yAxis[0]
        );

        this.createScrollBar(chart);
        if (this.props.chartTitle) {
            this.addChartTitle(chart, this.props.chartTitle);
            this.createExportMenu(chart, this.props.chartTitle);
        } else {
            this.createExportMenu(chart, "");
        }
        for (let i = 0; i < this.state.columnsForYAxis.length; i++) {
            this.createSeries(
                chart,
                this.state.columnsForYAxis[i],
                this.state.legendNames[i],
                false,
                this.state.showDummyData
            );
        }

        // Prezoom only one we have some big dataset (equal or more than 10 points on xaxis)
        if (chart.data.length > 9) {
            this.preZoomChart(chart, axis[0]);
        }
        // Extending the axisFills to axis labels
        chart.plotContainer.adapter.add("pixelHeight", function (value, target) {
            return value + 40;
        });
    };
    render() {
        return (
            <div>
                <div id={this.state.chartId} className="chart-display" />
            </div>
        );
    }
}

export default StackedClusteredColumnChart;

```
I have tried to create a proper function name so that it will be easy for you to understand what I am doing in chart. Also, I have added comments in between for your understanding.

Lets run in and see it in action.
Hit the 'LineChart' on Navbar.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uvj4frvha3ikmglhixok.JPG)
Similary for ColumnChart
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n0vip49763bgqvxkf1x3.JPG)

Let's see the action in real time.

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r2m7rtmtqdkozgpejr9i.gif)

# Takeaway
There are certain places where I haven't explained that much. The article is getting too big and there are numerous other places where you get that info. Also my aim was to show the usage of amcharts library in react.js. We have already done the similar thing many times in [past](https://dev.to/raigaurav/data-visualization-using-amcharts-in-react-js-with-mojo-29mh) (if you are following my article). The only difference right now is jsx.
`LineChart` and `StackedClusteredColumnChart` components are the 2 key takeaway. You can use them as independent components in your code or modify it as per your need.

# Conclusion
With this we are done with our series.
In the past few months I have gone through different charting libraries and ways to use them. I have created different article based on that.
* [Data visualization: Creating charts using Perl (Chart::Clicker)](https://dev.to/raigaurav/data-visualization-creating-charts-using-perl-chart-clicker-1hm) 
* [Data visualization: Creating charts from perl using plotly.js (Chart::Plotly)](https://dev.to/raigaurav/data-visualization-creating-charts-from-perl-using-plotly-js-chart-plotly-3m48)
* [Data visualization: Using amCharts with Perl and Mojo](https://dev.to/raigaurav/data-visualization-using-amcharts-with-perl-and-mojo-38ff) 
* [Data visualization: Using amCharts in React.js with Mojo(without jsx)](https://dev.to/raigaurav/data-visualization-using-amcharts-with-perl-and-mojo-38ff) 

and the current one ofcourse.

I hope it will be helpful for you in future. All these libraries are quite poweful and you can use any one of them to create elegenet charts. 

The above example is also available at [github](https://github.com/rai-gaurav/mojo_react_app/tree/main/with_jsx).

# References
* [Perl](https://www.perl.org/)
* [Mojolicious](https://mojolicious.org/)
* [React](https://reactjs.org/)
* [Amcharts](https://www.amcharts.com/)
* [MDB](https://mdbootstrap.com/)

Amcharts logo taken from [here](https://www.amcharts.com/about/press-kit/amcharts_light_transparent/)
React logo taken from [here](https://reactjs.org/)
