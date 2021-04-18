
We will be looking at sending HTML emails with attachments using Perl.
As per Perl TIMTOWTDI philosophy there are multiple ways to send email. Few of them are -

1. Using [/usr/sbin/sendmail](https://en.wikipedia.org/wiki/Sendmail) utility - for linux
2. [MIME::Lite](https://metacpan.org/pod/MIME::Lite)
3. [Email::Send](https://metacpan.org/pod/Email::Send)
4. [Email::Sender](https://metacpan.org/release/Email-Sender)

I remember when I started using Perl and when a day arrived where I have to send email, I found MIME::Lite. It was good for that time being and its gets the job done.
But as the time progress multiple alternative appears and also in current day MIME::Lite is not the recommended way to send email. From there documentation -

> MIME::Lite is not recommended by its current maintainer. There are a number of alternatives, like Email::MIME or MIME::Entity and Email::Sender, which you should probably use instead. MIME::Lite continues to accrue weird bug reports, and it is not receiving a large amount of refactoring due to the availability of better alternatives. Please consider using something else.

Also, for Email::Send -

> Email::Send is going away... well, not really going away, but it's being officially marked "out of favor." It has API design problems that make it hard to usefully extend and rather than try to deprecate features and slowly ease in a new interface, we've released Email::Sender which fixes these problems and others. As of today, 2008-12-19, Email::Sender is young, but it's fairly well-tested. Please consider using it instead for any new work.

I also got opportunity to work on some older codes and there were extensive use of MIME::Lite and sendmail (which is the Linux utility for sending emails) and I thought these are de-facto standards but as the time progress I realize there are more and better ways to send email. 
Today we will be seeing one of those ways to end email which is recommended as of now - Email::Sender.

Lets revisit few of the basic process of sending email-

1. A mail template containing mail body in HTML format. In case you want to send plain text email you can ignore this.
2. Creating mail with the subject, attachment and mail body created in Step 1.
3. Send Email using SMTP server.

## 1. Creating the Email package -##
We will be creating a package containing all our email utilities(SendMail.pm).
```perl
package SendMail;
use strict;
use warnings;

sub new {
    my ( $class, @arguments ) = @_;
    my $self = {@arguments};
    bless $self, $class;
    return $self;
}

1;
```
## 2. Generating Mail Template -##
We will be creating a method for generating our mail template. You can use any library of your choice to generate this template. Few notable are -

* [Text::Template](https://metacpan.org/pod/Text::Template)
* [Template::Toolkit](https://metacpan.org/release/Template-Toolkit)
* [HTML::Template](https://metacpan.org/pod/HTML::Template)

I am using HTML::Template for this job as our scope is limited only to creating mail body and it is right tool for the job.
If you have some complex requirement you can check out other two (especially Text::Template).
Adding HTML::Template to the above code.
```perl
use HTML::Template;

sub generate_mail_template {
    my ( $self, $filename, $parameters ) = @_;

    # create mail body template. We don't want to die/exit if any of the parameters is missing
    my $template = HTML::Template->new( filename => $filename, die_on_bad_params => 0 );
    $template->param($parameters);
    return $template;
}
```
This takes the absolute filename of the  template file and the parameters we will be substituting in that template file.
Right now I am using a simple template which can be extended based on requirement (template.html)
```html
<html>
    <head>
        <title>Test Email Template</title>
    </head>
    <body>
        My Name is <TMPL_VAR NAME=NAME>
        My Location is <TMPL_VAR NAME=LOCATION>
    </body>
</html>
```
## 3. Creating Email to send -##
Now comes the part where we will be creating our email with subject, attachment and body before sending.
But before that, we want to make the users to which we will be sending email configurable and for that we will be creating a config file.
Right now I am creating a json based config (config.json). But, you can create it in any format of you choice.
```json
{
    "mail" : {
                "mail_from" : "test@xyz.com",
                "mail_to" : ["grai@xyz.com"],
                "mail_cc_to" : ["grai@xyz.com", "grai2@xyz.com"],
                "smtp_server" : "abc.xyz"
             }
}
```
The key names are self explanatory.

Now back to the part of creating email. We will create another method for this by adding to above code.
```perl

use Email::MIME;
use File::Basename qw( basename );

sub create_mail {
    my ( $self, $file_attachments, $mail_subject, $mail_body ) = @_;

    my @mail_attachments;
    if (@$file_attachments) {
        foreach my $attachment (@$file_attachments) {
            my $single_attachment = Email::MIME->create(
                attributes => {
                    filename     => basename($attachment),
                    content_type => "application/json",
                    disposition  => 'attachment',
                    encoding     => 'base64',
                    name         => basename($attachment)
                },
                body => io->file($attachment)->all
            );
            push( @mail_attachments, $single_attachment );
        }
    }
    # Multipart message : It contains attachment as well as html body
    my @parts = (
        @mail_attachments,
        Email::MIME->create(
            attributes => {
                content_type => 'text/html',
                encoding     => 'quoted-printable',
                charset      => 'US-ASCII'
            },
            body_str => $mail_body,
        ),
    );

    my $mail_to_users    = join ', ', @{ $self->{config}->{mail_to} };
    my $cc_mail_to_users = join ', ', @{ $self->{config}->{mail_cc_to} };

    my $email = Email::MIME->create(
        header => [
            From    => $self->{config}->{mail_from},
            To      => $mail_to_users,
            Cc      => $cc_mail_to_users,
            Subject => $mail_subject,
        ],
        parts => [@parts],
    );
    return $email;
}
```
This method will take list of attachments, subject and body and create a MIME part using [Email::MIME](https://metacpan.org/pod/Email::MIME).
Also there is [Email::Stuffer](https://metacpan.org/pod/Email::Stuffer) which you can also use for similar part. But for now I am using Email::MIME.

## 4. Sending Email -##
Now our mail to ready for send. We will create another method for it in addition to above methods.
```perl

use Email::Sender::Simple qw( sendmail );
use Email::Sender::Transport::SMTP;

sub send_mail {
    my ( $self, $email ) = @_;
    my $transport = Email::Sender::Transport::SMTP->new(
        {
            host => $self->{config}->{smtp_server}
        }
    );
    eval { sendmail( $email, { transport => $transport } ); };
    if ($@) {
        return 0, $@;
    }
    else {
        return 1;
    }
}

1;
```
This method will just take the email  which we have created before and sent it to requested recipient.
Since our work is done here we will end the module with a true value.

## 5. Using the module -##
Now we have the module ready. Lets create a perl script and try to utilize the methods which we have created(mail.pl).
```perl
#!/usr/bin/env perl

use strict;
use warnings;
use JSON;
use Cwd qw( abs_path );
use File::Basename qw( dirname );
use lib dirname(abs_path($0));
use SendMail;

sub read_json_file {
    my ($json_file) = @_;
    print "Reading $json_file";

    open (my $in, '<', $json_file) or print "Unable to open file $json_file : $!";
    my $json_text = do { local $/ = undef; <$in>; };
    close ($in) or print "Unable to close file : $!";

    my $config_data = decode_json($json_text);
    return ($config_data);
}

# Read the config file
my $config = read_json_file("config.json");

my $mail = SendMail->new("config" => $config->{'mail'});

# param which you want to substitute in mail template
my $mail_parameters = "NAME => 'Gaurav', LOCATION => 'INDIA'";

# path to mail attachments
my $attachments = ["abc.txt"];

# path to mail template
my $mail_template = "mail_template/template.html";

print "Generating HTML template for mail body";
my $template = $mail->generate_mail_template($mail_template, $mail_parameters);

print "Creating mail with body and attachments to send";
my $mail_subject = "Test Mail";
my $email = $mail->create_mail($attachments, $mail_subject, $template->output);

print "Sending email...";
my ($mail_return_code, $mail_exception) = $mail->send_mail($email);

if (defined $mail_exception) {
    print "Exception while sending mail: $mail_exception";
    return 0;
}
else {
    print "Mail Sent successfully";
    return 1;
}
```
And we are done. You can run this script from your terminal and test this. The full code for reference is also available at [Github](https://github.com/sudo-batman/perl-toolkit/tree/master/Mail) for your reference

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Mail logo taken from [here](https://fontawesome.com/icons/at?style=solid) and [here](https://fontawesome.com/icons/envelope-open-text?style=solid)
