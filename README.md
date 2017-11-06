# crane

So, this is basically just this idea I've had kicking around evolving in my head for a while ever since I first started learning C, about what I'd want a better compiled language to look like. Some of it's *purely* theoretical, and may have broad unrecognized (or just impractical) gaps in its epistemology; ultimately, I suspect the final product would end up looking something like [Nim](https://nim-lang.org/). (Many of the ideas in it took shape after looking at Lua, Haskell, Tcl, Bash, and Lisp.)

## Parsing

The parse structure would be *very* simple, outputting a tree where all the leaf nodes are simply byte-sequence string tokens (which then essentially represent S-expressions).

The syntax producing these tokens would be pretty much just "everything separated by whitespace is a leaf", with indentations to make a branch. There'd be a few bits of syntactic sugar:

- A backslash before flow-controlling whitespace would cancel that flow control (so a backslash at the end of that line would continue the next line as part of the same line, even if indented, and a backslash after *some* indentation would make it as if the next token started at that backslash)
- Any character outside of quotes, including non-flow-controlling-whitespace or a normally-starting quote, can be backslash-escaped as a single-character literal, Bash-style.
- Parentheses denote an inline branch. The first level of indentation within parentheses is ignored, to make it easier to span a branch across multiple lines.
- Colons might be reserved for flow control (with only indented lines after a colon starting a new branch), this is one of those things I get hung up on whenever I visit this idea.
- String syntax like `"Hello World"` would produce a construct equivalent to `(\" Hello\ World)`, ie. the token `"` followed by the token `Hello World`. Double-quotes accept backslash escape sequences like `\n` for a newline (and borrowing Lua's `\z` to eat all following whitespace): single-quotes interpret everything until the next single quote (including backslashes) literally (so backslash, as a token, can be written as either `\\` or `'\'`).
- Tokens starting with a non-letter character (if not backslash-escaped or the first token) would be treated as infix, ie `1 + 1` produces `(\+ 1 1)`
- Most other non-letter characters *within* a token are considered just part of the token (eg. `M*A*S*H` would be interpreted as one token)
- Consecutive dots and colons count as their own tokens without having to be separated by whitespace, unless the token starts with a digit (so `foo.bar` turns into `(\. foo bar)` but `3.14159` and `4:20` stay as they are).
- Square braces work like infix operators (and don't have to be separated by whitespace), but with the second argument being the one *within* the braces: `foo[bar]` becomes `(\[] foo bar)`
- Curly braces work similarly: `{foo bar baz}` becomes `(\{} foo bar baz)`
- The long-string syntax would probably be more like Bash heredocs than Python's triple-quotes or Lua's any-number-of-equal-signs-between-two-square-braces, with the delimiter working as a second parameter (which would generally be ignored): I'm thinking `[[here stringhere` evaluating to `(\[[ string here)`. (I've got some ideas around including end braces after the start / end token, which could be used to distinguish whether indentation / first newlines should be discarded, but it'd require some code blocks to explore and I don't have any strong feelings about the specifics)
- The original idea I had for comment characters - keeping in mind that I was just learning C - was that it'd have both `//` and `#`, where `#` would be for comments oriented toward *tooling*. These days, I think that's a dumb idea, but I'm not sure what character I'd want to reserve / cut out of the character set for tokenization: honestly, I'm leaning toward Lisp's `;`, as there aren't many other uses for semicolons in this model (and it'd go nicely with having colons as a reserved character for flow alongside parentheses).

## Compiling

Compiling would evaluate the parsed tree in an environment halfway between Lua and Haskell: the only types are boolean, number, table, string, function, nil, and maybe some kind of userdata (or coroutine), but all functions are curried, and the environment is built as a Prelude.

The environment would be a table with all global names in it, and would start with the primitive functions that can't be defined (ie. the kind of things that Lua provides in its Standard Library) preloaded (the Skyhook). This includes constructors like `function` and basic operations like addition (`+`) and indexing (`[]` and/or `.`).

At the head of every branch, the interpreter goes into the global environment table and calls the corresponding function with the following token (or result of a branch) in the branch (if there is any), and so on until the branch is consumed.

The environment would have a stack where further lexical contexts inherit non-conflicting names and that whole thing - basically, there'd be the *private* environment, which is generally encapsulated with scope in other languages, and the *public* environment, which I've never really seen in a stack-based context (ie. not just global clobbering) outside of shell programming, to be honest. (It feels weird to say that a concept useful for *functional* programming could come from experience with *Bash*, but there it is.)

At this point, you'd have preludes that would define the macros for converting whatever expressions into whatever machine code (or JavaScript or whatever) following whatever rules. The compiler would run through these to fill out the environment that could be said to define the actual "language" (which would be, like I said, something like Nim, with lots of nice advanced language features that are all optional if you just want to write something super hard-coded and stuff), then it'd go through the whole "compiling" stuff that I fully realize would make up 99.9% of the actual work of developing this (which is why this has all stayed a *thought experiment* for me instead of a project I've actually *undertaken*).
