## Fitting in with Existing Ecosystem

The Unix command-line philosophy is to combine small specialized programs
with pipelines and i/o redirection.  Tools like `cat`, `head`, `grep`, etc are
universally available and play well together.  `awk` occupies a special place:
it _is_ possible to write programs using 'awk', but it
is nowadays mostly used for powerful one-liners, particularly for data with delimited
columns like tab or space separated files.

So any new tool should not do what can already be easily done with the standard
toolbox.
In particular, making it easier to use a programming language for one-liners
should not reinvent `awk` (this has already happened with `perl`, but I don't care for
dollar-languages).

`llua` is a small Swiss Army-style command utility which exposes the expressive
power of Lua to command-line scripters.

This will install `llua` and its common aliases to '/usr/local/bin' (the default is '~/bin')

```
$ sudo lua llua install /usr/local/bin
```

## Matching Lua string patterns

[Lua string patterns](https://www.lua.org/pil/20.2.html) are a powerful subset of
true regular expressions. Technical they aren't regular expressions because of the
lack of alternation (various possible matches separated by '|') but follow the same
notation, with '%' being the escape character rather than '\'.

`llua m` applies the Lua `string.match` function to every line in its input.
Here we're picking out proper names, defined as an upper case letter followed by
one or more lower-case letters. The second example picks out _two_ values defined as
one or more non-space characters. Here we must use parentheses to indicate the
captured groups.

```sh
$ echo 'your name is Frodo' | llua m '%u%l+'
Frodo
$ echo 'Bonzo is a dog' | llua m '(%S+) is a (%S+)'
Bonzo	dog
```
`llua m` is a powerful way to extract columns out of arbitrarily-structured data;
its alias is `lmatch`.

On Linux "ls /proc | lmatch '%d+'" will return all the pids of running processes by
picking out the integer entries.  If a line does not match the pattern,
it will be ignored.

```sh
$ mount | lmatch '(/dev/%S+) on (%S+)'
/dev/sda2	/
/dev/sdb1	/home
```

## Lua-style global substitution

`sed` is the designated tool of choice, but I think there's sufficient complementary
power provided by `string.gsub` to warrant exposing it on the command-line. `lsub`
is the alias for `llua g':

```sh
$ echo 'bonzo is a dog' | lsub '(%S+) is a (%S+)' '%2-%1'
dog-bonzo
```
`lsub` takes a pattern argument and a replacement argument; any matches in
the pattern are available in the replacement as numbered captures. Note that by
default it is a _global_ replacement - all matches on each line will be substituted.

Any `llua` subcommand can take the `-t` flag, which provides a string that becomes
the _single_ input line of the command. This is convenient if you want to avoid the
`echo` trick!

The other kind of replacement string is an _expression_:

```sh
$ lsub -t 'HOME is where the heart is'  -l '(%u+)' 'getenv(s)'
/home/steve is where the heart is
```

`llua` makes all the functions in the `math` and `os` tables available globally
for your convenience.  [os.date](https://www.lua.org/manual/5.3/manual.html#pdf-os.date)
can convert timestamps into human date/time using the same flags as `strftime`.  This is
useful for preprocessing log files which don't bother to present time in a friendly way.

```sh
$ echo '1465637777 had a bath' | lsub -l '^(%d+)' 'date("%T",s)'
05:36:17 had a bath
```
## Evaluating some expression for each line

`llua l` (alias `leval`) evaluates an expression for each line; same relaxed rule for function
visibility as with `lsub -l`. The expression is an implicit function of
l (the line), ll (the _last_ line) and lno (the line number).

```sh
$ leval -t 'hello dolly' 'l:sub(1,4):upper()'
HELL
$ seq 1 5 | leval '10*l, 100*l'
10	100
20	200
30	300
40	400
50	500
```
This only works because Lua will auto-convert strings into numbers if the context demands
it - arguably a misfeature with non-trivial programs, but very convenient for the current
use!

Note that it is easy to produce multiple output items - just separate expressions with
commas.  What if we wanted another output delimiter? CSV is a popular format:

```sh
$ seq 1 4 | leval -o, 'l/pi, sin(l)/pi'
0.31831,0.267849
0.63662,0.289438
0.95493,0.0449199
1.27324,-0.240898
```
`-o` applies to the other subcommands as well.

I did not intend to re-invent Awk, but manipulating pre-split columns is very convenient.
The `-s` flag splits the input according to the input delimiter (set with `-i`)

```sh
$ echo "10 20" | leval -s "2*F[1], 3*F[2]"
20	60
```
CSV files are convenient, if you don't rely on differences involving quoting values. 
The `-c` flag implies `-s` and "-i, -o,".  It assumes that the the first line contains the column
names. In this case, you can access the fields by name - if the original names contain
spaces or odd characters, they will be replaced by underscores. Also, there's an option
to specify _output_ column names using a SQL-like notation.
 
```sh
scratch$ cat tmp.csv
X,Y
0.840188,1.183149
0.783099,2.39532
0.911647,0.592653
0.335223,2.30469
scratch$ leval -c 'F.Y - 2*F.X as A, F.X-F.Y as B' tmp.csv
A,B
-0.497227,-0.342961
0.829122,-1.612221
-1.230641,0.318994
1.634244,-1.969467

If you don't use the 'as name' notation for each output field, then no header is 
written out.

## Sorting by column

As with `lsub/sed`, there is already `sort` - it can sort on columns. But the meaning
of column here is 'character column' not 'delimited column'. This seems rather feeble at
this point in computing history, to have to work with fixed-format data (which was hip
when guys wrote FORTRAN wearing ties.)

Hence 'llua s', aliased as `lsort`.

First of all, make some random data, and first sort on second column, ascending,
and then sort on first column, descending. (We need 'n' to specify that this is
to be intepreted as numbers - without 'n' it uses usual string comparison.)

```sh
scratch$ seq 1 5 | leval 'random(),random()' > rand.txt
scratch$ lsort 2n rand.txt
0.911647	0.197551
0.840188	0.394383
0.277775	0.55397
0.335223	0.76823
0.783099	0.79844
scratch$ lsort 1nd rand.txt
0.911647	0.197551
0.840188	0.394383
0.783099	0.79844
0.335223	0.76823
0.277775	0.55397
```
`lsort` is not a speed demon.  But it is a lot easier to use than `sort` when you
have delimited columns.

## Collecting instances of a captured value

`llua c` aka `lcollect` is occasionally useful. It extracts a value like `lmatch`, but
counts how often each unique value occurs. These are words that appear first
on each line of this readme.

```sh
$ lcollect '%a+' README.md
as	1
sed	1
lsub	2
to	2
On	1
Lua	3
HELL	1
is	2
LLUA	1
....
```
We can combine our cool tools into pipelines, naturally.

```sh
$ cat README.md | tr ' ' '\n' | lcollect '%a+' | lsort 2nd | head
the	54
is	40
a	28
to	25
as	17
for	17
llua	17
and	15
sh	15
with	14
```

## Evaluating Lua expressions

This is the odd one out. `llua e` (alias `lx`) does not consume standard input.
I include it because it's simply more powerful than `expr` - the standard mathematical
functions are available, hex literals, and if you are using Lua 5.3, bit operators as well.

```sh
$# all functions and constants from math.* are available
$ lx 'sin(1.2)*pi + 1'
3.92809
$# As well as os.*; this is 'one hour in the future'
$ lx 'time()+3600'
1465477097
$ lx -x '0x100 + 0x200'
0x300
```
The `-x` flag forces hexadecimal output; as with `leval`, multiple expressions can
be evaluated and the output delimiter is controlled with `-o`.

## Environment and Flags

All operations understand the  environment variable `LLUA_FMT` which is a
custom format to use for printing floating-point values:

```sh
LLUA_FMT='%4.2f' lx 'sin(0.234),cos(0.234)'
0.23	0.97
```

If `LLUA_CONFIG` is defined, `llua` will read a configuration file at that location.
If it's just 1, then `llua` will use the default location `~/.lluarc`. It may contain
any Lua definitions:

```
$ cat ~/.lluac
K=1024
function s(x) return x*x end
$ export LLUA_CONFIG=1
$ lx 's(K)'
1048576
```
This is very useful if you have some custom constants and operations that you
would like at your fingertips. The relaxed lookup rules apply in this
file as well.

Every operation except `lx` understands `-n` which prints out the line number
as well.

All operations understand `-o` for output delimiter, for instance `-o,`. The
column-oriented input readers like `lfmt` and `lsort` also understand the
equivalent `-i` for input delimiter.

## Conclusion

I've found these to be the most useful Lua one-liners; where they overlap existing
standard tools, I've included them where they offer more functionality than the
standard. 

I did not resist the temptation to reinvent `awk`, since the functionality was so
convenient.

[Lua patterns](http://man.openbsd.org/patterns.7) are also used for URL rewriting in OpenBSD's
`httpd` daemon.

[sbp](https://f.juef.tk/sbp/) by Svyatoslav Mishyn already implements `lmatch`
using the OpenBSD modified Lua string pattern matching code.
