[Excel::Writer::XLSX](https://metacpan.org/pod/Excel::Writer::XLSX) is one of the de facto standard for creating excel file in Perl.
The documentation of module contain the ample example of the usage.
I am just adding one more case where you have JSON data and you have to write it in excel.
Before starting, I have to confess something. I hate writing in excel. I never liked creating excel. But it is very popular in business world. I have got several request where people ask data in excel format. One time the request was to create charts in excel. As a technical person who knows little bit of JavaScript and charting library, creating chart in excel is never interest to me.
So, I have created a script for this purpose only.
So lets get started.
##1. Input data-##
The input to this script is a JSON file. It is in a particular format. As it is JSON, it is easy to generate either programmatically or manually. You can generate it from database. If you are using NoSQL it will be piece of cake. Even for SQL generating JSON is very easy.
Let look at the the input JSON file.
For demo purpose I am taking the sample financial data available at [Microsoft docs](https://docs.microsoft.com/en-us/power-bi/create-reports/sample-financial-download).
```json
{
    "Financial Info": [
        { "Date": ["01-22-2021", "04-05-2017", "07-12-2019", "19-05-2020"] },
        { "Segment": ["Government", "Enterprise", "Channel Partners", "Midmarket"] },
        { "Country": ["Canada", "Germany", "France", "India"] },
        { "Product": ["Montana", "Paseo", "VTT", "Hero"] },
        { "Unit Sold": [1618, 888, 1545, 5693] },
        { "Sale Price": [20, 350, 300, 693] },
        { "Profit": [2457, 0, -350, 3763] }
    ],
    "User Info": [
        { "Name": ["Gaurav", "Saurabh"] },
        { "Age": [10, 5] },
        { "Salary": [5673, 3355] }
    ]
}
```
I will walk through the JSON file but before that if you look at the generated excel file things will become clear.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/3ir0r6gwid628pjgtj6i.PNG)
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/1z8puwn0z94eyt4irm5r.PNG)
These two images are from a single excel file having 2 worksheet.
Now if you compare the JSON file with the images you will have a pretty good idea about. Lets visit those.
* The first keys `Financial Info` and `User Info` are name of the 2 `worksheet`. You can have any name you want.
* The value of these contains the data which you wan to show in tabular format in sequential order(hence inside array).
* Inside those data the keys are table `headers` or top heading which you wan to highlight.
* The values of these key contain the data which you want to put in the column having that key as header.

You can add more worksheet or add more headers or do any thing. The baseline is you just have to stick to this format.

##2. Parsing the JSON file##
Lets start with the parsing this JSON file.
```perl
#!/usr/bin/env perl
use strict;
use warnings;

use JSON;

sub read_json_file {
    my ($json_file) = @_;
    print "\nReading $json_file";

    open(my $in, '<', $json_file) or die "Unable to open file $json_file : $!";
    my $json_text = do { local $/ = undef; <$in>; };
    close($in) or die "Unable to close file : $!";

    my $config_data = decode_json($json_text);
    return $config_data;
}

sub main() {
    my $output_file = "output.xlsx";
    my $data        = read_json_file("input.json");
}

main();
```
Here we are slurping the whole file and decoding it.

##3. Writing to excel##
Lets try to creating excel file out of parsed JSON file to get the expected output as mentioned in image above.
We will create a separate function for that. Adding this code to above one
```perl
use Excel::Writer::XLSX;

sub write_to_excel {
    my ($output_file, $data_to_write) = @_;
    print "\nWriting output to $output_file ";
    my $workbook = Excel::Writer::XLSX->new($output_file);

    # https://metacpan.org/pod/Excel::Writer::XLSX#add_format(-%25properties-)
    # Format for 'heading' (background - light blue)
    my $header_format = $workbook->add_format(
        border    => 1,
        bg_color  => '#99c2ff',
        bold      => 1,
        text_wrap => 1,
        valign    => 'vcenter',
        align     => 'center',
    );
    my %font = (color => 'black', valign => 'vcenter', align => 'left', font => 'Calibri', border => 1);

    # Format for a normal text (background - white)
    my $normal_format = $workbook->add_format(%font);

    }
}
```
We have created 2 different format for now - 
* Header format where we want to highlight the header
* A normal standard format for all other text
We are using [add_format](https://metacpan.org/pod/Excel::Writer::XLSX#add_format(-%properties-)) for that
Now just take a look at excel file and understand how to access a particular cell.
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/w91h8j2ep26sjjvflbi2.PNG)
The first cell is located at A1 or (0, 0), second B1 or (0, 1) and so on.
Now to automatically access these cell we have to write code in such way that will get increment to B,C,D... while going horizontally. Similar for going vertically the number should increment i.e. 1,2,3... etc.
We can create a separate module for that where we can generate the next string.
```perl
package StringIterator;
use strict;
use warnings;

sub new {
    my ($class, @arguments) = @_;
    my %self = @arguments;
    if (!exists $self{set}) {
        $self{set} = ["A" .. "Z"];
    }
    if (!exists $self{value}) {
        $self{value} = -1;
    }
    $self{size} = @{$self{set}};

    return bless \%self, $class;
}

# Increment the string to the next one
sub increment {
    my $self = shift;
    return $self->{value}++;
}

# get the current string (e.g. A or B ...)
sub get_current {
    my $self = shift;
    my $n    = $self->{value};
    my $size = $self->{size};
    my $s    = "";

    while ($n >= $size) {
        my $offset = $n % $size;
        $s = $self->{set}[$offset] . $s;
        $n /= $size;
    }
    $s = $self->{set}[$n] . $s;

    return $s;
}

# Get next string
sub get_next {
    my $self = shift;
    $self->increment;
    return $self->get_current;
}

1;
```
I have to give credit to someone for this code. While doing this work, I have found the help on the internet sometime back which I don't remember now from where. I tried to search now but unable to found it. I have modified it little bit for my need. But if someone found the actual source please let me know.
In the above code we have an array contains character form "A" to "Z". We also have a variable having initial value '-1'. With every `increment` call we are increasing the value of variable and returning the character present at that index in `get_current`.

Now lets use this module in our code and try to fill the excel.
```perl
#!/usr/bin/env perl
use strict;
use warnings;

# https://metacpan.org/pod/Excel::Writer::XLSX
use Excel::Writer::XLSX;
use JSON;
use Cwd qw( abs_path );
use File::Basename qw( dirname basename );
use lib dirname(abs_path($0));
use StringIterator;

sub write_to_excel {
    my ($output_file, $data_to_write) = @_;
    print "\nWriting output to $output_file ";
    my $workbook = Excel::Writer::XLSX->new($output_file);

    # https://metacpan.org/pod/Excel::Writer::XLSX#add_format(-%25properties-)
    # Format for 'heading' (background - light blue)
    my $header_format = $workbook->add_format(
        border    => 1,
        bg_color  => '#99c2ff',
        bold      => 1,
        text_wrap => 1,
        valign    => 'vcenter',
        align     => 'center',
    );
    my %font = (color => 'black', valign => 'vcenter', align => 'left', font => 'Calibri', border => 1);

    # Format for a normal text (background - white)
    my $normal_format = $workbook->add_format(%font);

    # Excel column start from 1
    my $row = 1;
    foreach my $key (keys %{$data_to_write}) {
        my $iter      = StringIterator->new();
        my $worksheet = $workbook->add_worksheet($key);

        # Column A width set to 20
        $worksheet->set_column('A:A', 20);

        # Add a table to the worksheet.
        if (@{$data_to_write->{$key}}) {
            foreach my $element (@{$data_to_write->{$key}}) {
                foreach my $header (keys %{$element}) {
                    my $column_initial = $iter->get_next;

                    $worksheet->write($column_initial . $row, $header, $header_format);
                    $worksheet->write_col($column_initial . ($row + 1),
                        $element->{$header}, $normal_format);
                }
            }
        }
    }
    $workbook->close();
    print "\nFinished writing to file : $output_file";
    return 1;
}

sub main() {
    my $output_file = "out.xlsx";
    my $data        = read_json_file("data.json");

    write_to_excel($output_file, $data);
}

main();
```
* Here we looping through our JSON data (`$data_to_write`) and adding the initial keys as worksheet name in workbook([add_worksheet](https://metacpan.org/pod/Excel::Writer::XLSX#add_worksheet(-$sheetname-))).
* Next we are setting the width of first column(A) as 20 by using [set_column](https://metacpan.org/pod/Excel::Writer::XLSX#set_column(-$first_col,-$last_col,-$width,-$format,-$hidden,-$level,-$collapsed-))
* After that, we are looping through the values which contain the actual data.
* In each iteration we are getting the column initial (i.e A or B or C etc), writing the header using [write](https://metacpan.org/pod/Excel::Writer::XLSX#write(-$row,-$column,-$token,-$format-)) with the header format and adding the whole array of data for that particular header using [write_col](https://metacpan.org/pod/Excel::Writer::XLSX#write_col(-$row,-$column,-$array_ref,-$format-)).

Save the file and try to run it. We will get the exact output as mentioned in Step 1.
The above code is generic enough to generate a excel file if you provide the JSON in that particular format without any issue.

##4. Some more addition - making specific##
Sometime there is also a requirement to highlight a particular field or set of fields.
E.g. For finance data, we want to highlight field where we are in loss as red, profit as green, no profit no loss as yellow. Also, we wan to highlight field where we have sold more than 1600 unit.
Lets update our code to accumulate this requirement.
* We will use [add_format](https://metacpan.org/pod/Excel::Writer::XLSX#add_format(-%properties-)) to create more format for profit, loss and others.
* We will be using [conditional_formatting](https://metacpan.org/pod/Excel::Writer::XLSX#conditional_formatting()) to format specific cell satisfying certain criteria.
Adding more to the subroutine above.
```perl
    # Format for error text (background - red)
    my $error_format = $workbook->add_format(%font, bg_color => '#ff0000');

    # Format for success text (background - green)
    my $success_format = $workbook->add_format(%font, bg_color => '#00ff00');

    # Format for neutral text (background - yellow)
    my $neutral_format = $workbook->add_format(%font, bg_color => '#ffff00');

    # Excel column start from 1
    my $row = 1;

    foreach my $key (keys %{$data_to_write}) {
        my $iter      = StringIterator->new();
        my $worksheet = $workbook->add_worksheet($key);
        if ($key eq "Financial Info") {
            my $row_size = scalar @{$data_to_write->{$key}->[6]->{'Profit'}};
            # 'Profit' is in 'G' column or '6'.
            # The data start from (1, 6) to (no of elements, 6)
            # If the value is greater than 0, apply the success format
            $worksheet->conditional_formatting(1, 6, $row_size, 6,
                {
                    type     => 'cell',
                    criteria => 'greater than',
                    value    => 0,
                    format   => $success_format
                }
            );
            # If the value is equal to 0, apply the neutral format
            $worksheet->conditional_formatting(1, 6, $row_size, 6,
                {
                    type     => 'cell',
                    criteria => 'equal to',
                    value    => 0,
                    format   => $neutral_format
                }
            );
            # If the value is less than 0, apply the error format
            $worksheet->conditional_formatting(1, 6, $row_size, 6,
                {
                    type => 'cell',
                    criteria => 'less than',
                    value => 0,
                    format => $error_format
                }
            );
            # 'Unit Sold' is in 'E' column or '4'.
            # The data start from (1, 4) to (no of elements, 4)
            # If the value is greater than 1600, apply the success format
            $worksheet->conditional_formatting(1, 4, $row_size, 4,
                {
                    type => 'cell',
                    criteria => '>',
                    value => 1600,
                    format => $success_format
                }
            );
        }

        # Column A width set to 20
        $worksheet->set_column('A:A', 20);

```
I have added the comments for clarity.
Save the file and run it. You can see the colored output.
![colored_output](https://dev-to-uploads.s3.amazonaws.com/i/lot049jnymn52hejmzlm.PNG)

##5. Adding charts ##
We can also add different charts. There are lot of option available at [Excel::Writer::XLSX::Chart](https://metacpan.org/pod/Excel::Writer::XLSX::Chart).
For now lets create a column chart for number of units sold.
We will be using [add_chart](https://metacpan.org/pod/Excel::Writer::XLSX#add_chart(-%properties-)) for this.
```perl
        # Add a column chart
        # https://metacpan.org/pod/Excel::Writer::XLSX#add_chart(-%properties-)
        my $chart = $workbook->add_chart(type => 'column', name => 'chart', embedded => 1);

        # https://metacpan.org/pod/Excel::Writer::XLSX::Chart#add_series()
        # ranges: [ $sheetname, $row_start, $row_end, $col_start, $col_end ]
        $chart->add_series(
            name       => "Unit Sold",
            categories => ["Financial Info", 1, $row_size, 2, 4],
            values     => ["Financial Info", 1, 4, $row_size, 4],
            line       => {color => 'blue'},
        );

        # https://metacpan.org/pod/Excel::Writer::XLSX#insert_chart(-$row,-$col,-$chart,-%7B-%25options-%7D-)
        $worksheet->insert_chart('J2', $chart);
```
I have again added the comments for explanation.

##6. Final look ##
Lets look at our final full script after the addition.
```perl
#!/usr/bin/env perl
use strict;
use warnings;

# https://metacpan.org/pod/Excel::Writer::XLSX
use Excel::Writer::XLSX;
use JSON;
use Cwd qw( abs_path );
use File::Basename qw( dirname basename );
use lib dirname(abs_path($0));
use StringIterator;

sub write_to_excel {
    my ($output_file, $data_to_write) = @_;
    print "\nWriting output to $output_file ";
    my $workbook = Excel::Writer::XLSX->new($output_file);

    # https://metacpan.org/pod/Excel::Writer::XLSX#add_format(-%25properties-)
    # Format for 'heading' (background - light blue)
    my $header_format = $workbook->add_format(
        border    => 1,
        bg_color  => '#99c2ff',
        bold      => 1,
        text_wrap => 1,
        valign    => 'vcenter',
        align     => 'center',
    );
    my %font = (
        color  => 'black',
        valign => 'vcenter',
        align  => 'left',
        font   => 'Calibri',
        border => 1
    );

    # Format for a normal text (background - white)
    my $normal_format = $workbook->add_format(%font);

    # Format for error text (background - red)
    my $error_format = $workbook->add_format(%font, bg_color => '#ff0000');

    # Format for success text (background - green)
    my $success_format = $workbook->add_format(%font, bg_color => '#00ff00');

    # Format for neutral text (background - yellow)
    my $neutral_format = $workbook->add_format(%font, bg_color => '#ffff00');

    # Excel column start from 1
    my $row = 1;

    foreach my $key (keys %{$data_to_write}) {
        my $iter      = StringIterator->new();
        my $worksheet = $workbook->add_worksheet($key);
        if ($key eq "Financial Info") {
            my $row_size = scalar @{$data_to_write->{$key}->[6]->{'Profit'}};
            # 'Profit' is in 'G' column or '6'.
            # The data start from (1, 6) to (no of elements, 6)
            # If the value is greater than 0, apply the success format
            $worksheet->conditional_formatting(1, 6, $row_size, 6,
                {
                    type     => 'cell',
                    criteria => 'greater than',
                    value    => 0,
                    format   => $success_format
                }
            );
            # If the value is equal to 0, apply the neutral format
            $worksheet->conditional_formatting(1, 6, $row_size, 6,
                {
                    type     => 'cell',
                    criteria => 'equal to',
                    value    => 0,
                    format   => $neutral_format
                }
            );
            # If the value is less than 0, apply the error format
            $worksheet->conditional_formatting(1, 6, $row_size, 6,
                {
                    type => 'cell',
                    criteria => 'less than',
                    value => 0,
                    format => $error_format
                }
            );
            # 'Unit Sold' is in 'E' column or '4'.
            # The data start from (1, 4) to (no of elements, 4)
            # If the value is greater than 1600, apply the success format
            $worksheet->conditional_formatting(1, 4, $row_size, 4,
                {
                    type => 'cell',
                    criteria => '>',
                    value => 1600,
                    format => $success_format
                }
            );

            # Add a column chart
            # https://metacpan.org/pod/Excel::Writer::XLSX#add_chart(-%properties-)
            my $chart = $workbook->add_chart(type => 'column', name => 'chart', embedded => 1);

            # https://metacpan.org/pod/Excel::Writer::XLSX::Chart#add_series()
            # ranges: [ $sheetname, $row_start, $row_end, $col_start, $col_end ]
            $chart->add_series(
                name       => "Unit Sold",
                categories => ["Financial Info", 1, $row_size, 2, 4],
                values     => ["Financial Info", 1, 4, $row_size, 4],
                line       => {color => 'blue'},
            );

            # https://metacpan.org/pod/Excel::Writer::XLSX#insert_chart(-$row,-$col,-$chart,-%7B-%25options-%7D-)
            $worksheet->insert_chart('J2', $chart);
        }

        # Column A width set to 20
        $worksheet->set_column('A:A', 20);

        # Add a table to the worksheet.
        if (@{$data_to_write->{$key}}) {
            foreach my $element (@{$data_to_write->{$key}}) {
                foreach my $header (keys %{$element}) {
                    my $column_initial = $iter->get_next;

                    $worksheet->write($column_initial . $row, $header, $header_format);
                    $worksheet->write_col($column_initial . ($row + 1),
                        $element->{$header}, $normal_format);
                }
            }
        }
    }

    $workbook->close();
    print "\nFinished writing to file : $output_file";
    return 1;
}

sub read_json_file {
    my ($json_file) = @_;
    print "\nReading $json_file";

    open(my $in, '<', $json_file) or die "Unable to open file $json_file : $!";
    my $json_text = do { local $/ = undef; <$in>; };
    close($in) or die "Unable to close file : $!";

    my $config_data = decode_json($json_text);
    return ($config_data);
}

sub main() {
    my $output_file = "out.xlsx";
    my $data        = read_json_file("data.json");

    write_to_excel($output_file, $data);
}

main();
```
After running the script we will get following output -
![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/cek3axj36msfcp1cypex.PNG)

You can update the specific things to suite your requirements.
It is quite handy where you can generate JSON data in particular format(which is quite easy across any language) and convert it into beautiful excel.
Code is also available at [Github](https://github.com/rai-gaurav/perl-toolkit/tree/master/ExcelOperations)

Perl Onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Excel logo taken from [here](https://support.microsoft.com/en-us/office)
JSON logo taken from [here](https://www.json.org/json-en.html)
