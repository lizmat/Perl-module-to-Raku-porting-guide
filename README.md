# P5-modules-to-P6-porting-guide
Porting Guide for porting Pumpkin Perl 5 Modules to Rakudo Perl 6


# Exporter
Perl 6 has an Exporter built in, so you don't need to `use` one.

The basic functionality is explained at
[Exporting and Selective Importing](https://docs.perl6.org/language/modules#Exporting_and_Selective_Importing)

## Don't want to export anything by default

If you don't want to export anything by default, then mark the subroutines with
`is export(:FOO)` where `FOO` is a string that has meaning to you as a developer.
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
