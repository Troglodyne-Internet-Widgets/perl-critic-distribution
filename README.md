# NAME

Perl::Critic::Distribution - Parse a distribution once, for every policy that needs all of it.

# VERSION

version 0.002

# SYNOPSIS

```perl
package Perl::Critic::Policy::Something;
use parent qw{Perl::Critic::Policy};
use Perl::Critic::Distribution;

sub initialize_if_enabled {
    my ( $self, $config ) = @_;
    Perl::Critic::Distribution->register(
        name    => __PACKAGE__,
        version => Perl::Critic::Distribution->stamp(__FILE__),
        collect => sub {
            my ( $ppi, $file, $area ) = @_;
            return { subs => [ map { $_->name } @{ $ppi->find('PPI::Statement::Sub') || [] } ] };
        },
    );
    return $self->SUPER::initialize_if_enabled($config);
}

sub violates {
    my ( $self, $elem, $doc ) = @_;
    my $dist = Perl::Critic::Distribution->for_file( $doc->filename ) or return;
    my $all  = $dist->collected(__PACKAGE__);    # { $file => what collect returned }
    ...
}
```

# DESCRIPTION

[Perl::Critic::Document](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADocument) is what a policy knows about one file.  Some
questions need the whole distribution: whether anything calls a sub, or what a
sub in another file returns.  A policy that asks such a question has to parse
every file of the distribution, and two such policies parse every file twice.

This module parses each file once per process, and hands the parsed document
to every collector that is registered.  A collector is a sub that a policy
registers.  It reads the [PPI::Document](https://metacpan.org/pod/PPI%3A%3ADocument) and returns what the policy needs
from that file as plain data, which JSON can hold.  The policy then asks for
what its collector returned, for every file.

The files are the Perl files under `bin/`, `lib/`, `t/` and `xt/` of the
distribution.  The distribution of a file is found as
["root\_of"](#root_of) says.

## WHEN A COLLECTOR RUNS

Register a collector from `initialize_if_enabled` in the policy.
Perl::Critic calls that for each enabled policy before it critiques the first
file, so a disabled policy adds no work, and every collector is registered
before the first policy asks for anything.  The first `collected` call of a
process then parses each file once and runs every registered collector on it.

A collector that registers later, after a distribution was read, is filled in
on its first `collected` call.  That walk parses each file again, and runs
only the collectors whose data is missing or stale.

## THE CACHE ON DISK

An editor integration such as PerlNavigator starts a new process for every file
that it checks.  Without a cache, every such check parses the whole
distribution.

So what the collectors return is also kept on disk, one file for each
distribution, as JSON compressed with gzip.  For each file it holds a stamp of
the file: its device, inode, size, and modification and change times.  It also
holds what each collector returned, with the version of that collector.  A new
process reads the cache.  It parses only a file whose stamp differs, or a file
that lacks the data of a registered collector at its current version.  A file
that is gone drops out, and a new file is parsed.  The cache is written again
only when something changed.

The data of a collector that is not registered in this process is kept, as
long as the stamp of its file is the same, for the next process that registers
it.

The cache also records the stamp of this module's own file.  So a new version
of this module, or an edit to it, starts from an empty cache.

Each time a cache is written, the cache of each distribution whose root is
gone, such as a deleted checkout, is removed from the cache directory.  The
root is in the gzip header of each file, so this does not read the files
whole.

A cache that cannot be read, does not parse, or cannot be written is ignored,
and the distribution is read as though there were none.  A check never fails
because of the cache.

## CAVEATS

A distribution is read once per process for each set of collectors.  A file
that is edited after that is not seen again in that process.  ["THE CACHE ON
DISK"](#the-cache-on-disk) is how the next process sees it.

Two policies that pass different `cache_dir` values to `for_file` get two
objects, and so two parses.  Leave `cache_dir` to its default, or give each
policy the same one.

## METHODS

### register

```perl
Perl::Critic::Distribution->register( name => $name, version => $version, collect => \&collect );
```

Registers a collector for every distribution of this process.  `name` is the
key that `collected` and `stash` take, usually the package of the policy.
`version` is any string, and a change to it runs the collector again on every
file.  The stamp of the policy's own file, from `stamp`, changes whenever the
policy does.  `collect` is called as `collect->( $ppi, $file, $area )`,
with the [PPI::Document](https://metacpan.org/pod/PPI%3A%3ADocument), the absolute path of the file, and `bin`, `lib`,
`t` or `xt`.  It returns what that policy needs from the file, as data that
JSON can hold.

Registering a name again replaces it.  Returns the name.  Dies on a missing
name or a `collect` that is not a code reference.

### stamp

```perl
my $stamp = Perl::Critic::Distribution->stamp($file);
```

What changes when a file does: its device, inode, size, and modification and
change times, as one string.  The inode changes when an editor saves by
renaming a new file over the old one, and the times are to the nanosecond, so
an edit that keeps the size within one second still changes it.  Undef if the
file cannot be read.

### root\_of

```perl
my ( $root, $area ) = Perl::Critic::Distribution->root_of($file);
```

The root of the distribution that `$file` belongs to, and which of `bin/`,
`lib/`, `t/` and `xt/` of it the file is in.  An empty list when it is in
none of them.

The root is the nearest directory above the file with a `dist.ini`,
`Makefile.PL`, `Build.PL`, `META.json`, `META.yml`, `cpanfile` or `.git`
in it.  So a file in `t/lib` is in `t/`, not in a distribution whose root is
`t/`.  With no such directory anywhere above, the root is the directory that
holds the nearest `bin/`, `lib/`, `t/` or `xt/` that the file is in.

### default\_cache\_dir

`$XDG_CACHE_HOME/perl-critic-distribution`, or
`~/.cache/perl-critic-distribution` when `XDG_CACHE_HOME` is not set.  Undef
when neither that nor `HOME` is set.

### for\_file

```perl
my $dist = Perl::Critic::Distribution->for_file( $file, cache_dir => $dir );
my $dist = Perl::Critic::Distribution->for_file( $file, cache => $on, cache_dir => $dir );
```

The distribution that `$file` belongs to, as `root_of` finds it, or undef
for a file in none.  Source with no file name, such as a string handed to
`critique`, belongs to none.

`cache_dir` is where the cache is kept.  Without it, `default_cache_dir`.
Undef keeps no cache.  The same root and cache directory return the same
object for the life of the process.

`cache` is for a policy that takes the parameters of `policy_parameters`,
and passes on what it got.  With `cache`, a false value keeps no cache, and
an empty `cache_dir` means `default_cache_dir`.

### root

The root directory of the distribution.

### collected

```perl
my $by_file = $dist->collected($name);
```

What the collector `$name` returned for each file of the distribution, as a
hash reference keyed by the absolute path of the file.  A file that PPI cannot
parse is not in it.  Reads the distribution the first time any collector asks,
and again for a collector that registered after that.  Dies on a name that is
not registered.

### area\_of

```perl
my $area = $dist->area_of($file);
```

`bin`, `lib`, `t` or `xt`: which part of the distribution a file it read
is in.  Undef for a file it did not read.

### stamp\_of

The stamp, as `stamp` makes it, that a file had when the distribution was
read.  A policy that keeps derived data in its `stash` can compare it, to
tell which files changed.  Undef for a file it did not read.

### stash

```perl
my $data = $dist->stash($name);
```

What the policy registered as `$name` last kept with `keep`, in this process
or in the cache on disk.  Undef when there is nothing, or when it was kept by
another version of the collector.

It is for data that a policy works out from what its collector returned for
every file, and that costs time to work out again.

### keep

```
$dist->keep( $name, $data );
```

Keeps `$data` as the stash of `$name`, and writes the cache.  Returns true if
the cache was written, which is never when there is no cache directory.

## FOR POLICIES

What a policy that reads the whole distribution usually needs as well.

### policy\_parameters

```perl
sub supported_parameters {
    return ( { name => 'my_own', ... }, Perl::Critic::Distribution->policy_parameters() );
}
```

The parameters `cache` and `cache_dir`, for `supported_parameters` of a
policy.  `cache` is on by default, and an empty `cache_dir` means
`default_cache_dir`.  Perl::Critic puts them in `$policy->{_cache}` and
`$policy->{_cache_dir}`, which `for_file` takes as they are:

```perl
Perl::Critic::Distribution->for_file( $file, cache => $self->{_cache}, cache_dir => $self->{_cache_dir} );
```

Give every policy that reads the distribution the same `cache_dir`, or none,
so that they share a parse.

### called\_by\_perl

```perl
my @names = Perl::Critic::Distribution->called_by_perl();
```

The subs that Perl or a framework calls by name, so that no code in the
distribution calls them: the blocks such as `BEGIN`, `AUTOLOAD`,
`DESTROY`, `import` and `unimport`, `CLONE`, the hooks of Moose and Moo
such as `BUILD`, and the methods of the tie interface.  A policy that judges
a sub by who calls it leaves these alone.

### walk

```perl
Perl::Critic::Distribution->walk( $ppi, $visit );
Perl::Critic::Distribution->walk( $node, $visit, package => $package, sub => $sub );
```

Calls `$visit->( $element, $package, $sub )` for each element below
`$node`, in source order.  `$package` is the package that the element is
in.  `$sub` is the qualified name of the sub that the element is in, such as
`Some::Thing::helper`, or undef outside every sub.

A package statement changes the package of the statements after it in the
same block, and a package block changes the package inside its block.  The
visit sees the package statement, and then what is in its block, but not the
name in the statement.  A sub statement is seen with the sub around it, and
what is in it with the sub that it defines.  A forward declaration defines no
sub.

`package` and `sub` give the context of `$node`, for code that is not a
whole file, such as a document parsed from inside a string.  Without them,
`$node` is in `main` and in no sub.

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/teodesian/perl-critic-distribution/issues](https://github.com/teodesian/perl-critic-distribution/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
