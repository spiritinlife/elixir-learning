# elixir-learning
# Follow this project by going to each version, you will have the files needed from the actual old commit of elixir's project and my Readme that explains some stuff probably.

_______


# Start research from here
## git clone the elixir project and use git checkout with the provided commit tags to jump to history 
# traverse with git log --reverse --ancestry-path 337c3f2d569a42ebd5fcab6fef18c5e012f9be5b..master
### `git checkout be7809679c30fb00e98aae58581990d79da0da35`
#### Add test suite and start to parse the tree.

https://arifishaq.wordpress.com/2014/01/22/playing-with-leex-and-yeec/
http://blog.rusty.io/2011/02/08/leex-and-yecc/

Important files **elixir_lexer.erl** and **elixir_parser.yrl**

## Lexer

We start with leex. No theory here. There are plenty of resources on the web that explain what a “lexer” does.
It will break our string into tokens in a format yecc wants. Each token will say what type of input we are dealing with.

```
%% leex file structure
 
Definitions.
 
Rules.
 
Erlang code.
```

We’ll start in the middle. With the rules. They are the heart of it all.

A rule is made up of two parts. A regular expression (a la Erlang) and Erlang code. This code will be invoked if the parsed string matches the regular expression and will generally return a token tuple of the form `{token, Token}`.
 ***The regular expression and the code are separated by whitespace, a colon and other whitespace. The code can call functions implemented in the third section, Erlang code.***

The code may return
 - `{token, Token}`
 - `{end_token, Token}` when the ***returned token is known to be the last token***.
- `{skip_token, Token}`  when the token in question is to be ignored,
- `{error, ErrorDescription}` for errors

Back to the Token. 
***This is a tuple with generally three fields: {Category, Location, Content}.***

Token is `{Category, Location, Content}`
- `Category` tells us something about the ***nature of the token. Whether it is a number, a string, etc.***
- `Location` specifies ***where in the input we found the token***. This is probably used for debugging purposes, but is required by yecc. The Erlang code can assume there is a variable TokenLine which is bound to the line number.
- `Content` is what we would like the parser to believe was input as a token. It may be what was actually present in the parsed string, but it could be something we decide. The Erlang code can assume the availability of variables TokenLen and TokenChars. The former is the length of the matched substring. The latter is the matched substring.

> There are times when you will want the Category to be the same as the Content. In such a situation your code can also return just `{Category, Location}`.

While we are at it, let’s just say a word about the definitions sections. It allows us to define macros which we can use in specifying the regular expressions for the rules.

The definitions take the form of Macro = Value. No dot at the end here!


Now lets see what we have now in elixir

First lets compile everything and see by example
Lets first compile everything by hand
- `leex:file('./src/elixir_lexer').` gives leex our lexer
- `c("./src/elixir_lexer.erl").` compiles the lexer and exposes the string/1 function
- `elixir_lexer:string("2")` should return `{ok,[{integer,1,2}],1}`, this means it is working. 

> **_What happened now_**<br> "1" was fed to the string function of elixir_lexer and according to our specified rules it spits out erlang code that can now be fed to our parser 

Lets play a little bit before we continue with parser
Lets see how the lexer file now looks
```
elixir_lexer.erl

Definitions.

Digit = [0-9]

Rules.

%% Numbers
{Digit}+\.{Digit}+ : { token, { float, TokenLine, list_to_float(TokenChars) } }.
{Digit}+           : { token, { integer, TokenLine, list_to_integer(TokenChars) } }.

%% Operators

\+    : { token, { '+', TokenLine } }.
-     : { token, { '-', TokenLine } }.
\*    : { token, { '*', TokenLine } }.
/     : { token, { '/', TokenLine } }.
\(    : { token, { '(', TokenLine } }.
\)    : { token, { ')', TokenLine } }.

Erlang code.
```

We see now that for example this rule
<br>`{Digit}+           : { token, { integer, TokenLine, list_to_integer(TokenChars) } }.`<br>
follows the `{Category, Location, Content}` speicification. Category is integer, Location is TokenLine and Content is the erlang function result of list_to_integer.

We also see the usage of `{Category, Location}` in the Operators.For example
<br>`\+    : { token, { '+', TokenLine } }.`<br>
Says that the Category and the Content is the same and it is the plus sign (+) and the Location is TokenLine

## Now lets dig deeper

1. We have a definition of Digit in regex which is any number from 0-9
2. Next we have the definitions for Numbers. We see a definition for integer and float
    1. `{Digit}+` is the integer definition and returns `{ token, { integer, TokenLine, list_to_integer(TokenChars) } }`. By our example `elixir_lexer:string("2")` gave `{ok,[{integer,1,2}],1}`. We are interested in the `{integer,1,2}` part.
    It conforms with what is written in our lexer. `integer` is a 'constant', `1` is the TokenLine and `2` is the result of `list_to_integer(TokenChars)` an erlang function that take as input a string and gives an integer. ( You can test it out in the erl repl, type `list_to_integer("12").` )
    2. `{Digit}+\.{Digit}+` is the float definition and returns `{ token, { float, TokenLine, list_to_float(TokenChars) } }`. Pretty similar to integer, with the difference that if there is a dot between two integers then this is a float. Utilizing erlangs `list_to_float(TokenChars)` we get the float needed in erlang. Lets test this out also. `elixir_lexer:string("2.1").` gives `{ok,[{float,1,2.1}],1}`.
    So lexer recognized this as a float. And returned `float` instead of `integer`. And as previously gave the TokenLine and the real number.
3. Next we have the Operators definition. We wont analyze them all for brevity, but you can see that for elixir operators are (`+`,`-`,`*`,`/`,`(`,`)`) 
    1. `\+    : { token, { '+', TokenLine } }.`. What this says is that when the lexer finds a plus sign + it will return `{ token, { '+', TokenLine } }`.
    We can see that if we type `elixir_lexer:string("+").`, it returns `{ok,[{'+',1}],1}`.


> You can do more <br>
What if we gave `elixir_lexer:string("11*11+(5-10)/2").` this command. Well lexer handles it like a champ and gives out<br>```{ok,[{integer,1,11},{'*',1},{integer,1,11},{'+',1},{'(',1},{integer,1,5},{'-',1},{integer,1,10},{')',1},{'/',1},{integer,1,2}],1}```<br> 

> What if we put `elixir_lexer:string("11 + 11").`. This fails miserably and says
`{error,{1,elixir_lexer,{illegal," "}},1}`. The reason is specified in the result. We have not given a definition for whitespace to our lexer yet.



## Parser (yecc)
yecc uses the tokenized description of the string to parse and make stuff happen. Right now elixir lexer handles the arithmetic operations.
 To do this, it has to be instructed on the semantics of a valid arithmetic representation. These instructions we provide yecc in a yrl file. (elixir_parser.yrl)

Empty yrl whould look like that
```
Nonterminals.

Terminals.

Rootsymbol arithmetic.

Erlang code.
```

1. Nonterminals - these are syntactic elements that are composed of terminal systems in various patterns;
2. Terminals - these are indivisible symbols that form building blocks for composed structures. These are the tokens supplied by the lexer;
3. Rootsymbol - this is the root of the syntax tree, i.e. the contextual starting point for interpreting the input symbols;
4. Rules - these tell the parser how to classify and structure the tokens it is consuming. A rule has a name, the pattern it matches and the action to be taken when the rule fires. These return the final data structures to the application, so this is the point where you can perform any contextual transformations that are necessary;
5. Erlang code - as with the lexer grammar, this is where you can define functions to abbreviate the actions defined in the rules.

Lets split the actiual elixir_parser.yrl file to see what is going on.

### Terminals
```
Terminals
  float integer
  '+' '-' '*' '/' '(' ')'
       .
```
Those are what the lexer spit out. We see all the values that we got when analyzing the lexer. All these things are needed to define arithmetic operations.

### Nonterminals

```
Nonterminals
  arithmetic
  add_op
  mult_op
  number
       .
```

> A Grammar rule says how to infer a non-terminal from a sequence of terminals. Erlang code may optionally be associated with the rule and this code may make use of functions in the last section of the instruction file, the Erlang code section.

> The general form of the grammar rule is then:<br>
`Nonterminal -> sequence of terminals : associated Erlang code.`

Lets see a grammar rule<br>

```
number -> float   : '$1'.
number -> integer : '$1'.
```

Those define what a number nonterminal is for the parser
It can be either float or integer and gives '$1'. But what is '$1' ?

> yecc provides the atoms ‘$1’, ‘$2’, etc., that can be used in the Erlang code associated with a semantic or grammar rule. These atoms represent the actual token associated with each terminal in the rule. Thus ‘$1’ is the token associated with the first terminal, ‘$2’ the one associated with the second terminal and so on. The token, as we remember from the leex part, looks like: {Category, Location, Content}. This is why the implementation of number uses $1 because it is actually the Category(since Category=Content for numbers)

Pretty much the same for the oparators

```
%% Addition operators
add_op -> '+' : '$1'.
add_op -> '-' : '$1'.

%% Multiplication operators
mult_op -> '*' : '$1'.
mult_op -> '/' : '$1'.
```

We have an `add_op` nonterminal if we see a `+` or a `-` terminal<br>
We have a `mult_op` nonterminal if we see a '*' or a '/' terminal

Now lets see the interesting stuff that combine our nonterminals to create the semantics for our `arithmetic` (nonterminal) operations

```
arithmetic -> arithmetic add_op arithmetic :
  { binary_op, ?line('$1'), ?op('$2'), '$1', '$3' }.

arithmetic -> arithmetic mult_op arithmetic :
  { binary_op, ?line('$1'), ?op('$2'), '$1', '$3' }.

arithmetic -> '(' arithmetic ')' : '$2'.
arithmetic -> number : '$1'.
```

Lets start with the easy ones.

1. `arithmetic -> number : '$1'.` That states that if we have a number then we also have an arithmetic operation that returns the actual number
2. `arithmetic -> '(' arithmetic ')' : '$2'.` That states that if we have an arithmetic non terminal between `()` terminals, then we also have an arithmetics operation and the result is the second token from the lexer. Because if we write `elixir_lexer:string("(10)").` then what we get from the lexer is `{ok,[{'(',1},{integer,1,10},{')',1}],1}`. And `$2` is `{integer,1,10}`
3. `arithmetic -> arithmetic add_op arithmetic :{ binary_op, ?line('$1'), ?op('$2'), '$1', '$3' }.` That states that if you see something like 2 + 2 then there is a binary operation
at the line of the first terminal, the operator of the second terminal and the numbers for addition are the $1 and $3, meaning the first and third terminal
4. Like 3. but for mult_op

The ?line and ?op functions are utility functions defined at the end of the file 

```
Erlang code.

-define(op(Node), element(1, Node)).
-define(line(Node), element(2, Node)).
```

They utilize erlangs element function to take the appropriate value from the tuple
eg. for (3) if expressions was 2+2, ?line($1) is like saying ?line({integer, 1, 2}) which translated to element(2, {integer, 1,2}) and returns 1 as the line number, which is the second argument of the tuple.


Last but not least there is a definition for operators precedence 
```
Left 100 add_op.
Left 200 mult_op.
```
Which states that mult_op > add_op ( should be calculated first)


Now lets see it in actions!

Lets compile the parser
1. yecc:file("./src/elixir_parser.yrl").
2. c("./src/elixir_parser.erl"). Exposes parse function to take input from lexer
3. `elixir_parser:parse(element(2,elixir_lexer:string("1+2"))).` returns `{ok,{binary_op,1,'+',{integer,1,1},{integer,1,2}}}`. All makes perfect sense!

We used element in the above example to get only the tokens from the lexer and feed them immediately to the parser. Because if you remember the lexer for elixir_lexer:string("1+2")
returns {ok,[{integer,1,1},{'+',1},{integer,1,2}],1} and we only want [{integer,1,1},{'+',1},{integer,1,2}].


Finaly there is one more file called elixir.erl that actually exposes a function parse/1 and does what we did with this command `elixir_parser:parse(element(2,elixir_lexer:string("1+2"))).` so that we can execute elixir code easier.

```
-module(elixir).
-export([parse/1]).

parse(String) ->
	{ok, Tokens, _} = elixir_lexer:string(String),
	{ok, ParseTree} = elixir_parser:parse(Tokens),
	ParseTree.
```
To use it you just need to compile the elixir.erl file
1. c("./src/elixir.erl").
2. `elixir:parse("1+2"). and vouala!



That concludes this commit


So make compile and we are done 
Next go to a terminal and type erl to get into erlang repl

 ### First official release : 6052352b6281752b905b30eb5b08fac0f51f68cd
 - Date May 24, 2012