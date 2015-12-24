# Introduction

This project, temporarily named the GATHER programming language (for GATHERing And Transforming HTTP Elements Recursively), is a build system integrating Web pages in addition to local resources.

## Motivation

There have been several times when I wanted to download and process webpages, but the existing tools are inadequate. Existing build tools rarely handle webpages well (just try to create a Make rule that depends on one!), while existing download tools like `wget` and `curl` don’t allow the kind of complex processing often required. Before this project, I would attempt to build a tool for each case from scratch, or just do it manually. However, that is time consuming and error prone, and often I would decide that whatever final result I was after was not worth all of the work of building an entirely new tool. With this project I could simply write GATHER programs to download the process the webpages, which would be much simpler than writing an entirely separate tool. 

## Language domain

The language is for a build tool which allows downloading and building from webpages. This allows people to write automatic website scrapers which automatically get rid of unwanted parts of webpages, output ebooks, or otherwise process the results of the downloaded pages. Equivalently, it allows people to write build scripts for ebooks or other compiled output, where the source files might not all be local to the machine.

## Example use cases

Some example use cases are:

* Downloading an entire book from [Wikibooks](https://en.wikibooks.org/wiki/Main_Page) or [Wikisource](https://en.wikisource.org/wiki/Main_Page). These sites have some downloading tools, but they don’t make it easy to select all of the pages in a book at once.
* Downloading all of the pages of a paginated news article and combining them together into a single, easily-readable page.
* Downloading stock market information for several stocks (such as the ones which had the highest percentage gains in a given time period, or all the components of an ETF) and performing automatic analysis of that information.
* Downloading several related blog posts in a multi-part series, and all the related comments, and putting them in a readable ebook.
* As shown in the presentation, downloading the entire archives of a comic or other serial publication and keeping only the main information and not extraneous headers.
* Downloading a site which often changes, and keeping a version history in a version control system like Git or Mercurial.

All of these have the obvious similarities of downloading something, usually from multiple webpages, not all known in advance, and performing some sort of analysis on the results. If all the pages are known in advance, then a shell script with `curl` or `wget` could often suffice. Even in those cases, it would be useful to have a build system to check notice when files had actually changed, in order to avoid extra processing. However, some existing build systems would be confused by the rules to download files without any local dependencies.

Even though all the above examples involve downloading from the Web, that is not strictly necessary for this program. Other use cases, however, would usually be better satisfied by existing build systems. Although I have not used it for much, I took inspiration from the [`tup`](http://gittup.org/tup/) build system, and believe it offers a majority or perhaps all of the features of GATHER for projects that do not depend on web downloading.

# Language Design

## What constitutes a program

The user writes files in this language as structured text files, as with most programming languages. It is not intended to be used interactively (like, e.g., shell and Python), but instead it is expected that the entire program will be read in before it is run. Although the execution model is designed to allow the source code to be compiled and then executed, I have no plans to actually implement a separate compiler for the foreseeable future.

Each program will have as its input the contents of the current directory it is run in (or potentially another specified on the command line), and any Web resources it accesses. As output, it will create files and/or directories within its current directory. The expected behavior is that almost all files will at some point originate on the Web, but that there might also be local copies cached; I list the current directory as an input because the program might use these cached copies in the same way that `make` uses the intermediate files it previously built if possible. This does allow for purely local builds, but other build systems are probably preferable in those cases.

For the purposes of this project, programs are to be written in standard text editors. I currently have no intention of providing even a syntax highlighter for this language.

## Standard operation

The DSL does not exactly have control structures because it does not have a standard definition of control flow. Instead, it has a list of actions to run which it builds into a dependency graph, like most other build systems). Each action can have inputs and outputs, and if none of the inputs change then the outputs are not recomputed. However, there are four main differences between this and standard build systems:

1. The inputs can be web pages instead of files. This is the primary reason for making this project.
2. Inputs can be sets of web pages or files, not just single ones. This is because you might want to do the same thing to many different webpages, such as strip out site headers or combine them all into an ebook. Many build systems have something similar, like `make`’s implicit rules which allow you to easily turn a set of `.c` files into a single executable, but as far as I know none of them generalize well to URLs.
3. A single rule can have some inputs and outputs the same. This means that this rule will be run repeatedly until a fixed point is reached.
4. There can be larger cycles in the dependency graph, as long as each output is adding members to sets and each input is a `foreach` calculation on those sets. This is run until no members are actually added to the output set. This is useful for recursively downloading many links: if you want everything linked to by a page, and everything linked to by those pages, etc., then you can set a rule with the same URL group as both input and output in order to add each new link to the URL group.

Other types of recursion and dependency cycles might be allowed in later versions, but I have no idea how to implement them in a reasonable way now. Unfortunately, the current design fails my standard of “able to write a build script for LaTeX files with automatic running of `bibtex`, `makeindex`, etc.”; fortunately, most recursive building scenarios aren’t as complex as LaTeX.

The current version of the program has types for strings, file paths, URLs, and sets of other types. The semantics of sets of sets aren’t fully decided yet and my current plan is to just make those type errors whenever they’re encountered. Future versions of the program might also support some common Web types like XPaths, CSS paths, or MIME types.

### Detailed program translation

1.	Parse input file into rules and variable declarations
2.	Determine module outputs
	* In current version, all variables which are the output of any rule are considered outputs.
	* Future plans allow the module to only specify a subset of these as desired products
3.	Work backwards through the set of rules from these outputs; construct directed maybe-cyclic graph of rules.
	* The graph should have a single pseudorule as the endpoint which has as inputs all the desired outputs of the module and puts these in their final output location. This is useful for the post-translation execution.
	* If any rule requires an input which is never built and not declared as a variable, throw an error and halt.
	* If any rule’s action requires an input that isn’t listed in that rule’s inputs, throw an error and halt.
	* Make sure output and input types match
4.	Sanity checks
	* If more than one rule outputs to a single non-set variable or outputs the value of the set instead of adding to it, throw an error and halt.
	* If there is a cycle other than as described above, throw an error and exit.
5.	If there is a cycle of multiple rules (each using sets and foreach as described above), then for each such cycle identify the first rule of the cycle to be run.
	1. If at least one of the sets in the cycle is added to or initialized by a rule outside that cycle, then narrow down the set of candidate first rules to one of the ones with such a set as input.
	2. If there is more than one candidate rule and at least one of the candidate rule input sets starts nonempty, narrow down the set of candidate rules to ones where the cycle input set is nonempty.
	3. If there is more than one candidate rule, narrow the candidate set down to the ones with the fewest dependencies outside the cycle (not counting any dependencies as described in the previous case).
	4. Choose the candidate rule from earliest in the file to be the first rule of the cycle.
6.	Find the set of runnable rules. This is initally the set of rules for which no inputs depend on other rules, unioned with the set of first rules of cycles (as described above) with no inputs outside the cycle.
	

### Detailed post-translation execution

Note that the graph is pruned at this point; only the rules which matter for the final outputs are listed.

1.	If, at runtime, the set of outputs is smaller than the compiled set, prune the tree further. This step is not necessary for the initial implementation because there is no advance compilation.
2.	While the set of runnable rules is not empty:
	1.	Check if the inputs have changed from the last time the rule was run.
	2.	If any of the inputs have changed, run the rule; otherwise, just mark it as run.
	3.	Determine the newly runnable rules
		1.	If this rule has an output that matches one of its inputs where that output is different from how it was when the rule was before the rule was run, mark this rule as runnable again and stop determining newly runnable rules.
		2.	If this rule is part of a cycle of multiple rules:
			* If the output which is part of the cycle was not changed and all the rules in the cycle have run, then consider all rules which are not part of the cycle and have an input which is output by the cycle as potential next rules for the next step of determining the newly runnable rules.
			* If the output was changed  or not all rules in the cycle have been run, change the first rule of the cycle to the rule of the cycle following the current rule. This is the only potential next rule.
		3.	If the list of potential next rules has not already been determined, then they are the ones which have inputs which correspond to this rule’s outputs.
		4.	Mark all outputs of this rule which are inputs of a newly runnable rule as computed, unless there are unrun rules which output to the same variables.
		5.	For each potential next rule, if all of its inputs have been computed or it is the first rule of a cycle, mark that rule as runnable.
3.	If the final pseudorule has not been run, then there has been some sort of error, probably with the above algorithm. Print debug information and exit. Otherwise, exit with success.
4.	Save metadata about the state of all inputs and outputs so that re-running the program can skip unneded steps if things have not changed.

This algorithm is designed to be easy to parallelize, but is not quite parallelizable yet. In particular, there will be extra work done in some cases involving cycles (both one-rule long and the set-addition cycles) unless those are handled carefully. This could probably be fixed by having a “half-computed” state for variables instead of just “computed” and “not computed”.

## Error Handling

Sometimes, things don’t go according to plan. In some cases, this is completely irrecoverable: a parse error, for example, will force the program to quit right away. Assuming the program is read in correctly, it might still fail if it cannot download webpages, subcommands fail, etc. In this case, there are two main things the program does (or doesn’t do):

1. It reports the error (to the standard error stream), in as much detail as relevant (controllable with verbosity flags).
2. It *does not* change the outputs of the current action, and continues execution (although there may be a way to get abort-on-error functionality). For example, instead of saving the 404 page of a website, the program will act as though the webpage is completely unchanged; instead of creating a blank file when some potprocessing fails, the file will remain the same. This is important because in many cases the final outputs of the program are still usable in their old states, but would not be usable if the program were aborted halfway through. As an example, imagine that part of the program is supposed to download the images on a webpage, but instead fails. In this case, a likely cause is that the images URLs are dead or the image server is temporarily down, and the rest of the page is still partially useable without the images.

## Syntax

The syntax is a work in progress; I had to make a lot of compromises to make it easy to parse. I hope to fix it up into a neater, easier-to-use syntax later. In the meantime, it matches that shown in the presentation with only a few changes.

# Example programs

Not provided because I ran out of time and because the parser is untested.

# Language Implementation

## Ideal

### Overview
The language is implemented as an external DSL, with the processing written in Scala. I chose Scala because we covered it in class and I knew how to do parsing in it; the only other language I could say that about was Haskell, and it seemed like it would be particularly difficult to get my program to work when most of the functionality would need to be wrapped in the `IO` monad. I chose an external DSL instead of an internal DSL because it seemed like that would be easier to implement and it allowed me to handle the variables and types more easily than I could if I needed those to be part of an internal DSL.

I have separate modules for parsing, the intermediate representation, and the actual execution. For the parsing, I used the same `scala.util.parsing.combinator` library we used for Garden and  the external piconot projects. For the intermediate representation, I used an AST built from case classes and case objects I wrote myself. The backend and executable intermediate representation should use an existing graph library like [Graph for Scala](http://www.scala-graph.org/), but I am not actually familliar with any such libraries and the necessary algorithms are easy enough to write myself.

### Parsing and AST
The parsing uses the same tools we have seen before: `JavaTokenParsers` extended with `PackratParsers`. It can parse the syntax just fine, with some custom regular expressions being used for parsing. This parser outputs a sequence of Statements from the AST.

The AST contains two types of Statement, the Rule and the Declaration. Declarations consist of a type, a variable name, and an initial value expression. Expressions can be either literals or sets of literals. Rules contain assignments for input and output variables, and an action to take with those inputs and outputs. The actions can be shell escapes, calling other modules, or builtins.

### Execution and executable IR
As detailed above, the AST is transformed into an executable IR, which consists of a directed potentially-cyclic grapho of rules, a set of those rules deemed “runnable”, a dictionary of variable names to values, and the set of variable which have been computed (approximately meaning their values are final, although this doesn’t actually work when there are cycles involved and I plan to change how this set works because of the difficulties involved in cycles).

Because Scala is immutable by default but in GATHER all variables can be changed until the time they’re declared computed (and sometimes even then, with cycles involved), some extra work needs to be done to ensure that these graphsa nd other data structures behave as expected.

## Current State

The parsing compiles but is untested and the execution is completely unwritten. Moreover, the parsing fails to handle the interpolation in shell escapes (and the AST handles them in a weird but workable way), and the format for literals was changed to be easier to parse by making them string literals prefixed with `f` for file or `u` for `url`.

The URL update frequency only allows a few preset values, and does not allow more complex schedules or schedules which include months or years (because months and years change the number of days in them).

Only shell escape actions are supported by the AST at the moment, not builtin or module actions. It would be easy ta add module actions, but to get them supported by the parser I would need to figure out exactly how to distinguish them from builtins (if at all; maybe builtins should really just be pre-provided modules).

# Evaluation

## Good parts

The design of the language semantics is pretty good. The specific execution algorithm above has some problems, but I’m pleased with my design for recursive and nonlocal building. The language feels like something I could use for the use cases I mentioned at the start of the document, with the potential exception of keeping a version-controlled archive (which I could do with a combination of GATHER and ordinary shell scripting).

## Problem areas in the project

I kept trying to design the project to perfection before implementing part of it. I have never been a fan of agile development for the beginnings of projects; I feel like the final product often works better when there is a good, solid design in the beginning. Repeated incremental changes can build some great products, but can also magnify small initial problems until they are almost impossible to deal with.

Unfortunately, that sort of design approach could not work for this project. It was a half-semester-long project for one of several courses I was taking as a college student. I could easily spend at least a semester full time desiging the perfect Web-compatible build system. Although that might lead to a better design, without the problems the current design has aronud cycles and syntax, it would also take too long for this project. I had trouble building the actual product instead of keeping overdesigning it.

This misfocus on design also caused problems where I overdesigned some aspects and did not put enough effort into other aspects, or where I wasted time on scope creep and didn’t even have a good piece of a design to show for it. This is why the syntax is still pretty bad (I kept focussing more on the semantics) and part of why the cycle semantics is weird (I initially tried to allow much more complicated cycles, but found that to be difficult or impossible and had to scale back).

## Problems with the current design and implementation

As mentioned several times before, there are issues with the syntax and with the implementation of how cycles work. Specifically, the syntax for rules is very clunky with its explicit “in“ and “out” keywords, and the current implementation of treating URLs and file paths as tagged string literals is ugly. I am not sure how to handle the input and output parameters; I haven’t really seen a good syntax for those. I have some ideas, like emulating `tup`’s, but that would require other changes I haven’t worked out yet. Better literals could be implemented simply by getting some regular expressions for file paths and URLs, but deciding what exactly those should be is difficult. The URL regular expressions should be mostly doable; I could require any special characters to be percent-escaped without running into any problems with the rest of the syntax. The files are harder, especially if I keep wanting to support both Windows and its backslash-delimited path names and POSIX style with backslashes escaping spaces. There is probably some good solution somebody else already came up with if I look hard enough, though.

As for the cycles, I think I know how to clean those up. The problems aren’t really with the semantics but the specific algorithm for implementing those semantics. Ideally, more complex graphs could also be supported, but I’ve looked into this problem and have no indication it has ever been solved.

Finally, there are a lot of implementation problems. The current implementation is unfinished (due to time constraints) and untested (because for some reason the test library doesn’t seem to work on my computer and I have no time to debug). I don’t think any more needs to be said about that which hasn’t been said in the Language Implementation section.

I definitely plan to solve both the design and implementation problems in the future; I came up with the idea for this project because it would be useful for me, and that usefulness hasn’t stopped just because the semester has.