# P5-modules-to-P6-porting-guide
Porting Guide for porting Pumpkin Perl 5 Modules to Rakudo Perl 6

# while loops with list assignment

Using list assignments in `while` loops will not work, because the assignment
will happen anyway even if an empty list is returned, so that this:

    while (($key, $value) = each %hash) { }

will loop forever.  There is unfortunately no way to fix this in Perl 6 module
space at the moment.  But a slightly different syntax, will work as expected:

    while each(%hash) -> ($key,$value) { }

Also, this will alias the values in the list, so you don't actually need to
define `$key` and `$value` outside of the `while` loop to make this work.

# lazy lists

Many functions in Perl 6 have the same name, but return a `Seq`uence rather
than a list.  In most cases, that really doesn't matter, as you either assign
it to an array, or use the result to iterate over.  If you, for some reason,
really want a list, you can get that calling the `.list` method on it.

# undef vs Nil

`Nil` is the closest thing that Perl 6 has to Perl 5's `undef`.  `Nil` is the
value that indicates the absence of a value.  If you assign `Nil` to a
variable, it will reset the variable to its default value.  If you haven't
specified a default value, and you haven't specified a type, it will set the
variable to `Any` upon assignment of Nil.

If you need to be able to pass a `Nil` value around, you **can** do this, but
it requires some care: you will need to either only bind to values that can
have a `Nil` value, or make sure the default value of the variable is `Nil`.

    my $a := frobnicate(42);                 # binding
    my \b  = frobnicate(666);                # no sigil means binding
    my $c is default(Nil) = frobnicate(747); # specific default

    my $d; $d = Nil; # Any

# Exporter
Perl 6 has an Exporter built in, so you don't need to `use` one.

The basic functionality is explained at
[Exporting and Selective Importing](https://docs.perl6.org/language/modules#Exporting_and_Selective_Importing)

## Don't want to export anything by default

If you don't want to export anything by default, then mark the subroutines
with `is export(:FOO)` where `FOO` is a string that has meaning to you as a
developer.  To self-document this, you could use the string "DONTEXPORT":

    sub frobnicate($a) is export(:DONTEXPORT) { ... }

By specifying a specific name, you're preventing the standard export logic to
export that sub if you don't specify anything in the `use` statement.

Outside of the scope of any class in the compilation unit, you must create a
subroutine named `EXPORT` that takes any positional arguments.  Something
like:

    sub EXPORT(*@args) {
        my %imports = EXPORT::FOO::{ @args.map: '&' ~ * }:p;
        if @args.grep( { !%imports{$_} } ) -> @huh {
            die "PACKAGENAME doesn't know how to export: @huh.join(', ')";
        }   
        %imports
    }

This `EXPORT` sub will be executed when a `use` of the file is done.  If it
gets any arguments, it will look if they're marked with "is export(:FOO)".
If not all arguments were found, it means one is trying to import things the
module doesn't know about.  So let the world know it can't do that.
Otherwise return the hash with the export information and let the system
deal with it.

## Moose and friends (OO)

You don't need them. OO is builtin to Perl 6 to the extent that almost
everything is an object:

    class Geo::IP2Location::Lite

    has %!file is required;

You can also trivially define your own types:

    subset IPv4 of Str where / (\d ** 1..3) ** 4 % '.' /;

Then use them within method signatures:

    method get_country ( IPv4 $ip ) { ... }

View [Classes and Objects](https://docs.perl6.org/language/classtut) for much
more information about this

## timely destruction

There is no such thing as timely destruction in Perl 6.  Perl 6 *does* support
the `DESTROY` method in a class.  However, there is *no* guarantee when this
method will be called on an object, if ever.  If the program is done, it
will simply exit and *not* call the `DESTROY` method on remaining live
objects.  This can be troublesome for some cases.

The Perl 6 ecosystem contains a module [FINALIZER](http://modules.perl6.org/dist/FINALIZER)
that can be used by module developers to register code that will be run when
an object needs to be finalized in a timely fashion (for instance on program
exit).  In the simplest case, this looks like:

    submethod TWEAK() {
        &!unregister = FINALIZER.register: { .finalize with self }
    }

    method finalize() {
        &!unregister();  # make sure there's no registration anymore
        # do whatever we need to finalize, e.g. close db connection
    }

See the documentation of [FINALIZER](http://modules.perl6.org/dist/FINALIZER)
for more information.

## the _ prototype

In Perl 5 you can have a `_` prototype, that will default the parameter to
use the `$_` of the scope of the caller.  There is no such thing in Perl 6.
However, it can be mimicked with some work, by using multi-dispatch.  If
in Perl 5 you would do:

    sub foo(_) { my $value = shift; say $value }
    $_ = 42;
    foo;      # says "42"

You would write this as a multi-sub in Perl 6:

    multi sub foo()       { foo( CALLERS::<$_> ) }
    multi sub foo($value) { say $value }

The first mult-sub taking no parameter will call itself, but with the `$_`
(`<$_>`) of the callers scope (`CALLER::`).  The second multi-sub takes
one parameter, which will just `say`, as we did in the Perl 5 version.

## tie

There is no `tie` as such in Perl 6.

The Perl 6 ecosystem also contains the [P5pack](http://modules.perl6.org/dist/P5tie>)
module that exports `tie`, `untie` and `tied` functions that support
many of the features of Perl 5's tie / untie / tied.  But using that module
will rob you of many exciting new Perl 6 capabilities.

### tieing a scalar

If you like to use the functionality of tieing a scalar value, you should look
at [the Proxy object](https://docs.perl6.org/type/Proxy) and
[Custom Containers](https://docs.perl6.org/language/containers).

### tieing an array / hash

If you like to use the functionality of tieing an array or a hash, you should
look at [Subscripts / Custom Types](https://docs.perl6.org/language/subscripts#Custom_types).

## pack / unpack

pack / unpack are still experimental in Perl 6, however we are able to get
to the native data types with NativeCall so can implement most of what we need:

    use NativeCall;

    method !read8 ( IO::Handle $handle, Int $position ) {
        $handle.seek($position-1, SeekFromBeginning);
        my $data = $handle.read(1);
        nativecast((int8), Blob.new($data));
    }

This is roughly equivalent to:

    sub read8 {
        my ($handle, $position) = @_;
        my $data = "";
        seek($handle, $position-1, 0);
        read($handle, $data, 1);
        return unpack("C", $data);
    }

The Perl 6 ecosystem also contains the [P5pack](http://modules.perl6.org/dist/P5pack)
module that exports a `pack` and `unpack` function that supports many
of the features of Perl 5's pack / unpack.

## Tests

You will find the Test functions map quite nicely in Perl 6 to the Pumpkin
Perl 5 modules, and most of the extra Test:: namespace functions are builtin
to the Perl 6
[Test](https://docs.perl6.org/language/testing) class

    use Test;

    plan 10;

    ...

    for %ips.keys -> $k {
        is $ip2.get_country_short( $k ),%ips{$k},"$k resolves to { %ips{$k} }";
    }

Running them just requires the `--exec` argument to prove:

    /Volumes/code_partition/geo-ip2location-lite-p6 > prove --exec perl6 -Ilib -r
    ./t/004_exceptions.t ... ok
    ./t/005_lookup.t ....... ok
    ./t/010_lookup_full.t .. ok
    All tests successful.
    Files=3, Tests=35,  1 wallclock secs ( 0.02 usr  0.01 sys +  0.94 cusr  0.12 csys =  1.09 CPU)
    Result: PASS
