
##DSLs Critique Week 2
### Critiquer: Men Cheol Jeong

### Language Design

It looks like you have a very exact idea of what your langauge would look like from the user point of view.
It also looks like you have taken a lot of it from the make/CMake build systems for what constitutes a program.
Have you thought about other approaches to solving this problem? From people like us, who have already used such a system this is an intuitive solution but I remember that the make system was not very intuitive on first pass.

For operations, again see if you can find other formats that could be useful for describing how code is compiled.
It seems like an easy way to implmeent this is to simply take a file, find and convert all web files to local files and then pass a fixed version to the make system. I also wonder if something like this exists (or if there is a way in make to use webfiles that is difficult). I imagine that since make just computes each lines as if it were a terminal/console that you can use a couple different command sequences to do the same thing in make.

That's a good way to handle parsing errors. The parsing errors should also indicate where the parser failed.
I think you've covered all the cases (although there are always more cases...). The way you're handling errors in part 2 are a little confusing to me. It sounds like you will have some cached outputs that feed into other outputs. It would seem more elegant to me for each of the intermediate files to either be removed at tht end of the computation or put in a separate directory. I guess control over where outputs go is also important.
Are you going to implmenent a way for the intermediate files or the final output to be placed online?

Yeah, I would not want to write a DSL in shell. Have you decided on a language?
Great ideas to start with!

Did you post a project notebook? I can't seem to be able to locate it. 
Sorry if my critique didn't address what you were looking for :-(
