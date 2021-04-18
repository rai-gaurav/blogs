
We will be looking at how to automate windows application using Perl.
Again there are multiple ways to achieve this. Few of them are -

1. [Win32::GuiTest](https://metacpan.org/pod/Win32::GuiTest)
2. [X11::GUITest](https://metacpan.org/pod/X11::GUITest)
3. [Win32::OLE](https://metacpan.org/pod/Win32::OLE)

There are other [autohotkey](https://www.autohotkey.com/) and [autoitscript](https://www.autoitscript.com/) which are quite good in windows automation. I encourage you to check them out.

For this blog we will be using Win32::GuiTest for automating windows application.
We will start with the a small example which can later be extended.
Lets visit what we will be doing -

1. We will open the given directory in windows explorer (Right now C:/Perl64/lib/)
2. Search for a file or folder (e.g. strict.pm)
3. Right-click on that particular file or folder(this is achieved using keyboard shortcuts)
4. Select an option (right now 'Open with Notepad++')
5. Wait for the action to be completed because of the selected option
6. After the action is successful close it (close Notepad++)
7. Close the windows explorer

See, that quite simple thing we will be doing. We will take baby steps by starting with our first step.

## 1. Creating the package - ##
We will be creating a package containing all our automation utilities(Win32Operations.pm).
```perl
package Win32Operations;
use strict;
use warnings;

sub new {
    my $class = shift;
    my ($self) = {@_};
    bless $self, $class;
    return $self;
}
```
## 2. Starting Windows Explorer operations -##
Now we have to open a dir in windows explorer. We will create a method for this.
```perl
sub start_explorer_operations {
    my ($self, $root_directory, $filename_to_open, $max_retry_attempts) = @_;

    # Path to windows explorer. The Windows directory or SYSROOT(%windir%) is typically C:\Windows.
    my $explorer = "%windir%\\explorer.exe";

    # Since we are on windows removed forward slash with backslashes
    $root_directory =~ s/\//\\/g;

    # seconds to spend between keyboard key presses
    my $key_press_delay = 1;

    # this is the interval the tester sleeps before checking/closing
    # any window; this is just for an eye effect so we can
    # watch what happens
    my $wait_time_for_windows = 3;

    my $count = 0;
    while ($count < $max_retry_attempts) {
        $count++;
        $self->{'logger'}->info("Opening dir: $root_directory in windows explorer");
        my $status_code = system($explorer , "$root_directory");
        if ($status_code == -1) {
            $self->{'logger'}->error(
                "Unable to open 'explorer.exe' with mentioned dir : $root_directory. Exited with return code :". $status_code);
            next;
        }
        elsif ($status_code & 127) {
            $self->{'logger'}->error("Child died with signal : " . ($status_code & 127));
            next;
        }
        else {
            # Windows explorer is opened hear and we are in the given dir
        }
    }
}
```
The comments are self explanatory. We will be opening windows explorer. It will retry for some  time in case it fails.
Also we have to cross check whether the open window is the location to our expected folder or not.
We will also give some time to system for this operation.
For this we will create a generalized method -
```perl
sub _wait_for {
    my ($self, $title, $wait_time) = @_;
    my $win;
    while (1) {
        # This will look for the window with the given title
        ($win) = FindWindowLike(0, $title);

        if (defined $win) {
            $self->{'logger'}->info("Found window with title : $title.");
            last;
        }
        else {
            $self->{'logger'}->info("Unable to find window with title : $title. Retrying...");
            sleep $wait_time;
        }
    }
    return $win;
}
```
## 3. Bringing Window to front -##
Now we are able to open the explorer windows and confirm that it is that window.
We will be trying to bring back that particular window to front. Reason being what if in between opening of explorer and starting our operation, our focus moved to somewhere else (someone clicked outside the explorer or open some other application). For this we will be creating another method -
```perl
sub _bring_window_to_front {
    my ($self, $window, $retry_attempts, $pause_between_operations) = @_;
    my $success = 1;
    my $count   = 0;
    while ($count < $retry_attempts) {
        $count++;
        if (SetActiveWindow($window)) {
            $self->{'logger'}->info("* Successfully set the window id: $window active");
        }
        else {
            $self->{'logger'}->warn("* Could not set the window id: $window active: $!");
            $success = 0;
        }
        if (SetForegroundWindow($window)) {
            $self->{'logger'}->info("* Window id: $window brought to foreground");
        }
        else {
            $self->{'logger'}->warn("* Window id: $window could not be brought to foreground: $!");
            $success = 0;
        }
        sleep $pause_between_operations;
        my $foreground_window_id = GetForegroundWindow();
        if ($foreground_window_id =~ /$window/i) {
            last;
        }
        else {
            $self->{'logger'}->info(
                "Found - $foreground_window_id instead of expected - $window. Will try again...");
            next;
        }
    }
    return $success;
}
```
The system restricts which processes can set the foreground window. A process can set the foreground window only if one of the following conditions is true:

* The process is the foreground process.
* The process was started by the foreground process.
* The process received the last input event.
* There is no foreground process.
* The foreground process is being debugged.
* The foreground is not locked.
* The foreground lock time-out has expired.
* No menus are active.

Please keep these in mind. Otherwise this method will fail.

## 4. Opening and closing in Notepad++ - ##
Now back to our 'else' part in 'start_explorer_operations' where we will be using these methods.
Now we have to search for our file, 'right-click' on it and select 'edit with Notepad++'.
After opening we will close it and close the explorer also.

1. Searching file - We may have noticed that if we open explore and start typing our file name on our keyboard, window will automatically select that particular file. We will be using similar thing here.
2. Right Click - we will be using `context menu key` on the keyboard (Another alternative - **Shift + F10**) for simulating right click.
3. Open with Application - If you press `context menu key` or `Shift + F10`, you can see different options and each options have a underline on one of the characters. These are the shortcuts to select that particular option/ In our case the underline is under `N` meaning if we press `N` we will be selecting that option. After that we will press `Enter` to open it.
4. Closing the application - We will be using the `close` command from the `System Menu` to close it. On the top of Notepad++ if you right click you can see the `System Menu`
```perl
sub start_explorer_operations {
    ....
        }
        else {
            my $window = $self->_wait_for(basename($root_directory), $wait_time_for_windows);
            $self->{'logger'}->info("Opened 'explorer.exe'. Window id : $window");

            $self->_bring_window_to_front($window, $max_retry_attempts, $wait_time_for_windows);
            $self->{'logger'}->info("Opening the file in Notepad++...");

            # This will use your keyboard to
            # Write the 'filename' to search
            # Right click on it(context menu) - {APP}
            # Select 'N' for Notepad++ and press 'Enter' to open
            # Replace 'N' with your own application shortcut if you are using something other application
            my @keys = ("$filename_to_open", "{APP}", "N", "{ENTER}");
            $self->_keys_press(\@keys, $key_press_delay);

            $self->{'logger'}->info("Opened the file in Notepad++. Closing it...");
            # Checking wheteher we are actually able to open Notepad++ or not
            my $opened_explore_window = $self->_wait_for($filename_to_open, $wait_time_for_windows);
            if (!$opened_explore_window) {
                $self->{'logger'}->warn("Cannot find window with title/caption $filename_to_open");
                next;
            }
            else {
               # We will come here when we successfully opened the file in Notepad++
               $self->{'logger'}
                    ->info("Window handle of $filename_to_open is " . $opened_explore_window);
                print $opened_explore_window;
                $self->_bring_window_to_front($opened_explore_window, $max_retry_attempts,
                    $wait_time_for_windows);
                $self->{'logger'}->info("Closing Notepad++ window...");
                MenuSelect("&Close", 0, GetSystemMenu($opened_explore_window, 0));
                $self->{'logger'}->info("Bringing Explorer window to front...");
                $self->_bring_window_to_front($window, $max_retry_attempts, $wait_time_for_windows);
                $self->{'logger'}->info("Closing Explorer window...");

                # There are different way to close windows File explorer -
                # 1. Alt+F4 (Will close anything on the top - little risky)
                # 2. Alt+F, then C
                # 3. CTRL+w (only closes the current files you're working on but leaves the program open)
                SendKeys("^w");
                return 1;
            }
        }
}

1;
```
Again, the comments are self explanatory. Another thing to note here is that, there are various ways to close the application. One of the common one is `Alt + F4`. But is it also risky. Reason being it can close anything on the top, anything.
What if on top there is some other application which you don't want to close.
Better to use `context menu` or application defined close.

## 5. Using the module -##
Now we have the module ready. Lets create a perl script and try to utilize the methods which we have created(Win32Operations.pm). We will be using [Log4perl](https://metacpan.org/pod/Log::Log4perl) for our logging. You can use anyone of your choice.
```perl
use strict;
use warnings;
use Log::Log4perl;
use Cwd qw( abs_path );
use File::Basename qw( dirname );
use lib dirname(abs_path($0));

use Win32Operations;

sub initialize_logger {

    # initialize logger, you can put this in config file also
    my $conf = qq(
            log4perl.category                   = INFO, Logfile, Screen
            log4perl.appender.Logfile           = Log::Log4perl::Appender::File
            log4perl.appender.Logfile.filename  = win32op.log
            log4perl.appender.Logfile.mode      = write
            log4perl.appender.Logfile.autoflush = 1
            log4perl.appender.Logfile.buffered  = 0
            log4perl.appender.Logfile.layout    = Log::Log4perl::Layout::PatternLayout
            log4perl.appender.Logfile.layout.ConversionPattern = [%d{ISO8601} %p] [%r] (%F{3} line %L)> %m%n
            log4perl.appender.Screen            = Log::Log4perl::Appender::Screen
            log4perl.appender.Screen.stderr     = 0
            log4perl.appender.Screen.layout     = Log::Log4perl::Layout::SimpleLayout
        );
    Log::Log4perl->init(\$conf);
    my $logger = Log::Log4perl->get_logger;
    $Log::Log4perl::DateFormat::GMTIME = 1;
    return $logger;
}

my $logger    = initialize_logger();
my $win32_obj = Win32Operations->new("logger" => $logger);

# This will open the given dir in windows explorer
# Right click on the given filename and open it in Notepad++ and then close it.
my $input_dir          = "C:/Program Files/Notepad++";
my $filename_to_open   = "readme.txt";
my $max_retry_attempts = 5;

$win32_obj->start_explorer_operations($input_dir, $filename_to_open, $max_retry_attempts);
```
And we are done. You can run this script from your terminal and test this. The full code for reference is also available at [Github](https://github.com/rai-gaurav/perl-toolkit/tree/master/Win32) for your reference.

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Windows logo taken from [here](https://docs.microsoft.com)
 
