As usual there are multiple ways to create beautiful charts in Perl.
1. [Chart::Clicker](https://metacpan.org/pod/Chart::Clicker)
2. [GD::Graph](https://metacpan.org/pod/GD::Graph)
3. [Chart::Plotly](https://metacpan.org/pod/Chart::Plotly)
4. [Chart::GGPlot](https://metacpan.org/pod/Chart::GGPlot) - Experimental 

We will be using Chart::Clicker in this post.

But let me get few thing clear before that. Both **Chart::Clicker** and **GD::Graph** hasn't been updated in last 4+ years. Especially for GD::Graph there are multiple issue opened and not sure about current status.

**Chart::Plotly** is something which I am more interested in right now. It is based on [plotly.js](https://plotly.com/javascript/).
I also encourage you to checkout [Dash](https://github.com/plotly/dash) which is built on top of Plotly.js, React and Flask for building ML & data science web apps.
There is something equivalent in Perl world also from same author(Chart::Plotly) - [Dash](https://metacpan.org/pod/Dash) which will be using [Mojolicious](https://metacpan.org/pod/Mojolicious) or [Dancer2](https://metacpan.org/pod/Dancer2). But right now it is in experimental stage and still under active development. I hope it will be ready for production soon because I can't wait to try my hands on it.

I have used Chart::Clicker 2-3 years back and it was good for my need at that time. It is based on [Cairo](https://www.cairographics.org/). The reason I used it instead of Chart::Plotly is because it was very new. But its 2021 and there are lots of beautiful charting library already available (specially on front-end side) which is not just easy to use but also look good to eyes. Few notable ones are -
1. [Chart.js](https://www.chartjs.org/)
2. [D3.js](https://d3js.org/)
3. [Amcharts](https://www.amcharts.com/)
4. [Highcharts](https://www.highcharts.com/)

There are many more (a lot) but that is not the focus of today. Out of these 
* **Chart.js** is lightweight and elegant when you have some basic and simple requirement. When your requirement gets more complex or too specific it will be little difficult to achieve and you have to write some extra codes to achieve it.
* **D3.js** is the beast which is difficult to tame but can win you any battle.(I spent some time on taming it but still not confident - maybe I am not a good Javascript developer till now.)
* I can personally vouch for **Amcharts**. After spending time on above two library, Amcharts is the sweet spot. It has everything you need and out of the box. Plenty of good examples on there website and highly configurable to your need - especially there [v4](https://www.amcharts.com/v4/) which is written in Typescript.

That's been said lets start with our example. There is a [talk](http://onemogin.com/assets/talks/Data-Viz-w-ChartClicker.pdf) given by author sometime back. You can also take a look at it for more info.

## Basic chart elements
Lets look at basic chart terminologies. From [canvasJS docs](https://canvasjs.com/docs/charts/basics-of-creating-html5-chart/chart-elements/)
![chart_terminology](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/39kmv6fwe4h5jkmwm6i9.PNG)
 
## Creating the data config
We will be creating multi-line chart for today. To make it easy to use and highly configurable we will be creating a input data json file
```json
{
    "title": "Number of automobiles sold per day by manufacturer",
    "domainAxis": {
        "data": ["2020-04-15", "2020-04-16", "2020-04-17", "2020-04-18"],
        "label": "Date"
    },
    "rangeAxis": {
        "lines": {
            "line1": {
                "data": [10,3,5,9],
                "legendName": "Honda"
            },
            "line2": {
                "data": [20,15,8,10],
                "legendName": "Toyota"
            },
            "line3": {
                "data": [6,19,12,4],
                "legendName": "Ford"
            },
            "line4": {
                "data": [16,10,6,12],
                "legendName": "Renault"
            }
        },
        "label": "Numbers of automobiles sold"
    }
}
```
The names are self explanatory. `domainAxis` is the x-axis which is a **Date**. `rangeAxis` is y-axis which has 3 lines for now.

## Directory structure
Our directory structure will be simple. *input* for input data, *output* for generated chart, *lib* for perl module.

 ┣ 📂input
 ┃ ┗ 📜input_data.json
 ┣ 📂lib
 ┃ ┗ 📜CreateCharts.pm
 ┣ 📂output
 ┣ 📜multi_line_chart.pl
 ┗ 📜README.md

## Creating the module
While working on creating the multi-line chart, I spend a quite amount of time understanding it. The documentation is good but still I think it can be improved. Why like this and how this is happening is unanswered at some places. One good thing is there are lot of [example](https://github.com/gphat/chart-clicker-examples/) present. So, you can look at it and found your way out.

So, lets create our module.
```perl
package CreateCharts;
use strict;
use warnings;

use Chart::Clicker;
use Chart::Clicker::Context;
use Chart::Clicker::Data::DataSet;
use Chart::Clicker::Data::Marker;
use Chart::Clicker::Data::Series;
use Chart::Clicker::Axis::DateTime;

use Geometry::Primitive::Rectangle;
use Geometry::Primitive::Circle;

use Graphics::Color::RGB;

use DateTime;

sub new {
    my ($class, @arguments) = @_;
    my $self = {@arguments};
    bless $self, $class;
    return $self;
}

sub _generate_specific_colors {
    my ($self) = @_;

    # build the color allocator
    # Add more colors in case your line is more than 6
    my $ca = Chart::Clicker::Drawing::ColorAllocator->new;

    my $red    = Graphics::Color::RGB->new({red => .75, green => 0,   blue => 0,   alpha => .8});
    my $green  = Graphics::Color::RGB->new({red => 0,   green => .75, blue => 0,   alpha => .8});
    my $blue   = Graphics::Color::RGB->new({red => 0,   green => 0,   blue => .75, alpha => .8});
    my $orange = Graphics::Color::RGB->new(red => .88, green => .48, blue => .09, alpha => .8);
    my $aqua   = Graphics::Color::RGB->new(red => 0, green => 1, blue => 1, alpha => .8);
    my $fuchsia = Graphics::Color::RGB->new(red => 1, green => 0, blue => 1, alpha => .8);

    $ca->add_to_colors($green);
    $ca->add_to_colors($red);
    $ca->add_to_colors($blue);
    $ca->add_to_colors($orange);
    $ca->add_to_colors($aqua);
    $ca->add_to_colors($fuchsia);

    return $ca;
}

sub _genrate_random_colors {

    # let Chart::Clicker autmatically pick complementing colors for you
    # https://metacpan.org/pod/Chart::Clicker::Drawing::ColorAllocator#AUTOMATIC-COLOR-ALLOCATION
    my $ca = Chart::Clicker::Drawing::ColorAllocator->new({
            seed_hue => 0,    #red
    });
    return $ca;
}

sub _add_series {
    my ($self, $x_axis, $y_axis) = @_;
    my $ds = Chart::Clicker::Data::DataSet->new;
    foreach my $axis (keys %{$y_axis}) {
        $ds->add_to_series(
            Chart::Clicker::Data::Series->new(
                keys   => $x_axis,
                values => $y_axis->{$axis}->{data},
                name   => $y_axis->{$axis}->{legendName},
            )
        );
    }
    return $ds;
}

sub _add_title {
    my ($self, $cc, $title) = @_;
    $cc->title->font->family('Helvetica');

    $cc->title->text($title);
    $cc->title->font->size(20);
    $cc->title->padding->bottom(10);
}

sub _style_legend {
    my ($self, $cc) = @_;
    $cc->legend->font->size(20);
    $cc->legend->font->family('Helvetica');
}

sub _add_background {
    my ($self, $cc) = @_;

    # https://metacpan.org/pod/Graphics::Color::RGB
    my $titan_white = Graphics::Color::RGB->new(red => .98, green => .98, blue => 1, alpha => 1);
    my $white       = Graphics::Color::RGB->new(red => 1,   green => 1,   blue => 1, alpha => 1);

    $cc->plot->grid->visible(1);
    $cc->background_color($white);
    $cc->plot->grid->background_color($titan_white);
    $cc->border->width(0);
}

sub _add_label {
    my ($self, $def, $x_label, $y_label) = @_;
    $def->domain_axis->label($x_label);
    $def->range_axis->label($y_label);
    $def->domain_axis->label_font->size(20);
    $def->range_axis->label_font->size(20);
}

sub _add_shapes_to_lines {
    my ($self, $defctx) = @_;

    # https://metacpan.org/pod/Chart::Clicker::Renderer::Line#shape
    $defctx->renderer->shape(Geometry::Primitive::Circle->new({radius => 6,}));

    # https://metacpan.org/pod/Chart::Clicker::Renderer::Line#shape_brush
    $defctx->renderer->shape_brush(
        Graphics::Primitive::Brush->new(
            width => 2,
            color => Graphics::Color::RGB->new(red => 1, green => 1, blue => 1)
        )
    );

    $defctx->renderer->brush->width(2);
}

sub generate_chart {
    my ($self, $chart_loc, $summary_info) = @_;

    my $cc = Chart::Clicker->new(width => 800, height => 600, format => 'png');

    my $x_axis = $summary_info->{domainAxis};
    my $y_axis = $summary_info->{rangeAxis};

    my (@epoch_datetime);

    for my $datetime (@{$x_axis->{data}}) {

        # https://github.com/gphat/chart-clicker/blob/master/example/date-axis.pl
        # Need to convert date time string to epoch time
        my ($y, $m, $d) = split(/-/, $datetime);
        my $epoch = DateTime->new(year => $y, month => $m, day => $d)->epoch;
        push @epoch_datetime, $epoch;
    }

    my $ds = $self->_add_series(\@epoch_datetime, $y_axis->{lines});
    $cc->add_to_datasets($ds);

    # To generate random colors and let Chart::Clicker autmatically pick color
    # my $ca = $self->_genrate_random_colors();

    # To generate some specific colors for lines ise this function
    my $ca = $self->_generate_specific_colors();
    $cc->color_allocator($ca);

    $self->_add_title($cc, $summary_info->{title});
    $self->_style_legend($cc);
    $self->_add_background($cc);

    my $defctx = $cc->get_context('default');

    # For range axis
    $defctx->range_axis->range(Chart::Clicker::Data::Range->new(lower => 0));
    $defctx->range_axis->format('%d');

    # https://metacpan.org/pod/Chart::Clicker::Axis#fudge_amount
    # $defctx->range_axis->fudge_amount(0.02);

    # For domain axis
    $defctx->domain_axis(
        Chart::Clicker::Axis::DateTime->new(
            format => "%Y-%m-%d",
            ticks  => scalar @{$x_axis->{data}},
            position         => 'bottom',
            tick_label_angle => 0.78539816,                       # 45 deg in radians
            orientation      => 'vertical',
            tick_font        => Graphics::Primitive::Font->new({family => 'Helvetica', slant => 'normal'})
        )
    );

    $self->_add_label($defctx, $x_axis->{label}, $y_axis->{label});
    $self->_add_shapes_to_lines($defctx);

    $cc->write_output($chart_loc);
}

1;
```
I tried to use proper names for subroutines, so it is easy to understand. Comments are also added.
* `_generate_specific_colors` - generates specific set of color for your lines. You can update it to suit your need.
* `_generate_random_colors` - allow the colors to be chosen randomly by the module itself.
* `_add_series` - add the series/ lines to your chart. The loop inside will run for each line(total 3) and add it to dataset.
* `_add_title` - add title to your chart which you have provided in input json file. There is different format options available which you can update to suit your requirements.
* `_style_legend` - is to format the legends.
* `_add_background` - add the background colors and format to the chart.
* `_add_label` - to the charts and format them.
* `_add_shapes_to_lines` - add the circular shape and brush to the lines.
* `generate_chart` - this is the subroutine we will be calling from outside with the input json data.

You may have notices the underscore in front of subroutines name. This is because they are private to this module.

## Using the module
Let create our startup script to access this module for creating our chart.
```perl
#!/usr/bin/env perl

use strict;
use warnings;
use Cwd qw( abs_path );
use File::Basename qw( dirname );
use JSON;

BEGIN {
    $ENV{"SCRIPT_DIR"} = dirname(abs_path($0));
}

use lib $ENV{"SCRIPT_DIR"} . "/lib";
use CreateCharts;

my $chart_out_file = $ENV{"SCRIPT_DIR"} . "/output/lineChart.png";

sub read_json_file {
    my ($json_file) = @_;
    print "\nReading $json_file";

    open(my $in, '<', $json_file) or print "Unable to open file $json_file : $!";
    my $json_text = do { local $/ = undef; <$in>; };
    close($in) or print "\nUnable to close file : $!";

    my $config_data = decode_json($json_text);
    return $config_data;
}

sub main {
    my $data_in_json = read_json_file($ENV{"SCRIPT_DIR"} . "/input/input_data.json");

    my $chart = CreateCharts->new();
    $chart->generate_chart($chart_out_file, $data_in_json);
}

main;
```
We are reading the JSON data from input file and calling `generate_chart` with it.

## Running the script

Not lets run this script and check the output.

![output_chart](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8g3425cgjb0nkvqc0rpu.PNG)

The output which you are seeing above is from a linux machine.
One of the limitation I found about this module is it is difficult to make it run on windows. Maybe because of cairo or something else - not sure. Installing it on windows is not straight forward. I tried it but failed to make it working. If you know a way, let me know.

## Usage
As I mentioned before because of new modern libraries available, there is little chance you will be tempted to use this module.
The one place I can see its usability today is - 
* You want to quickly create a chart and you have no intention of using it on the web.
* Creating a chart over a period of time and sending it as mail attachment. Your input can be from a SQL or NoSQL database. That's the reason I have created the input as JSON so it can be created easily.
* I have taken the example of only line chart, but it also support many [other types](https://metacpan.org/pod/Chart::Clicker#FEATURES) also. You can look at the [examples](https://github.com/gphat/chart-clicker-examples/) for more understanding.
* Remember that colors play an important role in chart. Choosing a proper color with proper opacity (hue/saturation) play a vital role. The same chart can look good or bad based on color choices. There are various online tools which can help you. My personal favorite is [W3 color picker](https://www.w3schools.com/colors/colors_picker.asp). You can look at the color tutorial for more understanding.

The above example is also available at [github](https://github.com/rai-gaurav/Charts/tree/main/ChartClicker)

## What next
I will be writing more about **Chart::Plotly** and its usage as I found it more modern.
Also, I will be writing about **Amcharts** and how you can use it with Perl(Mojolicious) and React.js

Cover image taken from [here](https://www.amcharts.com/demos/multiple-value-axes/?theme=material)
