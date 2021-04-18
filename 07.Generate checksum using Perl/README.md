In this article we will be looking at how to generate different type of [checksum](https://en.wikipedia.org/wiki/Checksum) across Linux and Windows.

In Perl, there are various module available to generate checksum -
1. [Digest::SHA](https://metacpan.org/pod/Digest::SHA) - Perl core module
2. [CryptX::Digest](https://metacpan.org/pod/Crypt::Digest)

There are few other but either they are specific to particular digest or abandoned.

`Digest::SHA` is in Perl core since `v5.9.3`. So, if you have Perl installed having higher version than 5.9.3 you will have Digest::SHA and installation is not needed.
But what if you are in some closed system, stuck with some older Perl version. What if you don't have permission and you can't install/use any external modules. I have faced similar issue and I have created this script just for that purpose.

So lets get started.
Both linux and windows comes with inbuilt utilities to calculate different hash - 
* Linux   - md5sum, sha1sum, sha256sum, sha512sum etc.
* Windows - [certutil.exe](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)

We can execute these commands using [backticks](https://perldoc.perl.org/perlop#Backticks) operator and get there output.
For linux the commands are present in `/usr/bin`. For windows it is in `C:/Windows/System32/`

```perl
#!/usr/bin/env perl

use strict;
use warnings;

# HashAlgorithm choices: MD2 MD4 MD5 SHA1 SHA256 SHA384 SHA512
sub get_checksum {
    my ($path_to_file, $hash_algorithm) = @_;
    my $checksum;
    my $os = $^O;
    eval {
        if (-s $path_to_file) {
            if ($os =~ /MSWin32/i) {
                $checksum = (
                    split(
                        '\n',
                        `C:/Windows/System32/certutil.exe -hashfile $path_to_file $hash_algorithm`
                    )
                )[1];
            }
            else {
                my $cmd = "/usr/bin/" . lc($hash_algorithm) . "sum $path_to_file";
                $checksum = (split(' ', `$cmd`))[0];
            }
        }
        else {
            $checksum = "File is empty";
        }
    };
    if ($@) {
        print("Unable to calculate the $hash_algorithm for $path_to_file : $@");
    }
    return $checksum;
}

# Calculate checksum of the given file
my $checksum = get_checksum("test.txt", "SHA256");
print $checksum;
```
The output from external command contains some extra elements also. We are removing those and getting only the checksum.
Running this script generated the following output -
```shell
946576fd7b973e3a8c78e5695637727399f34ca8d6fd6b89f1f5328c1e0b0bdc
```

We can pass different type of checksum parameter to `get_checksum` function and and get different output. 
You can also modify the script to use it as command line tool.
This can come in handy if you want to check integrity of file across different OS without changing your code.

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Hash function image taken from [here](https://en.wikipedia.org/wiki/Cryptographic_hash_function)
