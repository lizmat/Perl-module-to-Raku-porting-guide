# P5-modules-to-P6-porting-guide
Porting Guide for porting Pumpkin Perl 5 Modules to Rakudo Perl 6

# undef vs Nil

`Nil` is the closest thing that Perl 6 has to Perl 5's `undef`.  `Nil` is the
value that indicates the absence of a value.  If you assign `Nil` to a variable,
it will reset the variable to its default value.  If you haven't specified a
default value, and you haven't specified a type, it will set the variable to
`Any` upon assignment of Nil.

If you need to be able to pass a `Nil` value around, you **can** do this, but it
requires some care: you will need to either only bind to values that can have
a `Nil` value, or make sure the default value of the variable is `Nil`.

    my $a := frobnicate(42);                 # binding
    my \b  = frobnicate(666);                # no sigil means binding
    my $c is default(Nil) = frobnicate(747); # specific default

    my $d; $d = Nil; # Any

# Exporter
Perl 6 has an Exporter built in, so you don't need to `use` one.

The basic functionality is explained at
[Exporting and Selective Importing](https://docs.perl6.org/language/modules#Exporting_and_Selective_Importing)

## Don't want to export anything by default

If you don't want to export anything by default, then mark the subroutines with
`is export(:FOO)` where `FOO` is a string that has meaning to you as a developer.

    sub frobnicate($a) is export(:FOO) { ... }

By specifying a specific name, you're preventing the standard export logic to
export that sub if you don't specify anything in the `use` statement.

Outside of the scope of any class in the compilation unit, you must create a
subroutine named `EXPORT` that takes any positional arguments.  Something like:

    sub EXPORT(*@args) {
        if @args { 
            my $imports := Map.new( |(EXPORT::FOO::{ @args.map: '&' ~ * }:p) );
            if $imports != @args {   
                die "PACKAGENAME doesn't know how to export: "
                  ~ @args.grep( { !$imports{$_} } ).join(', ')
            }   
            $imports
        }   
        else {
            Map.new
        }   
    }

This `EXPORT` sub will be executed when a `use` of the file is done.  If it gets
any arguments, it will look if they're marked with "is export(:FOO)".  If not all
arguments were found, it means one is trying to import things the module doesn't
know about.  So let the world know it can't do that.  Otherwise return the `Map`
with the export information and let the system deal with it.
