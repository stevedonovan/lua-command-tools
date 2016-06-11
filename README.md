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
dollar-languages).  It turns out that this is easier said than done.

## Matching Lua string patterns

Lua string patterns 