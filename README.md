# elixir-learning
# Follow this project by going to each tag, you will have the files needed from the actual old commit of elixir's project and my Readme that explains some stuff probably.
# Tags will follow very simple numbering v0,v1,v2... And when we reach a release version of the elixir project then it will be annotated in the tag.
_______


# Start research from here
## git clone the elixir project and use git checkout with the provided commit tags to jump to history 
## traverse with git log --reverse --ancestry-path 337c3f2d569a42ebd5fcab6fef18c5e012f9be5b..master
#v0
## `git checkout be7809679c30fb00e98aae58581990d79da0da35`
### Add test suite and start to parse the tree.

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




#v1
## `git checkout eafbf8d9b29709bf3c05869f13593a5879b73f70`
### Basic operations in progress.
Nothing special about this commit just create a way to actually execute our operations.
This done by erl_eval:expr/2 function which takes erlang syntax tree and evaluates the result. So what we do is the following
Lexer -> Parser -> Elixir Syntax Tree -> Tranform -> Erlang Syntax Tree -> erl_eval -> result


## `git checkout 09143d726232fec11ef289eff8ef10d4901bf73b`
### Allow whitespaces

This is very easy just change a little bit the lexer.
Add the Definition for a Whitespace
```
Definitions.

Digit = [0-9]
Whitespace = [\s]
```
And then just tell the lexer to skip this character

```
%% Skip
{Whitespace}+ : skip_token.
```

And we can now execute code like that `elixir:eval("1 + 2 + 3")`.

Nice.

## `git checkout 02882038ada642defdb2202484103647c9436b98`
### Proper operator precedence.
But much more happen in this commit

The elixir_parser.yrl got new things
which are the following
```
Nonterminals
  expr
  add_expr
  mult_expr
  unary_expr
  max_expr
  number
  unary_op
  add_op
  mult_op
  .

  Rootsymbol expr.

expr -> add_expr : '$1'.

%% Arithmetic operations
add_expr -> add_expr add_op mult_expr :
  { binary_op, ?line('$1'), ?op('$2'), '$1', '$3' }.

add_expr -> mult_expr : '$1'.

mult_expr -> mult_expr mult_op unary_expr :
  { binary_op, ?line('$1'), ?op('$2'), '$1', '$3' }.

mult_expr -> unary_expr : '$1'.

unary_expr -> unary_op max_expr :
  { unary_op, ?line('$1'), ?op('$1'), '$2' }.

unary_expr -> max_expr : '$1'.

%% Minimum expressions
max_expr -> number : '$1'.
max_expr -> '(' expr ')' : '$2'.

%% Unary operator
unary_op -> '+' : '$1'.
unary_op -> '-' : '$1'.

```

We see new terminals and that the rootsymbol is now expr instead of arithmetic.
At this point elixir now is defined as language that takes expressions and those expr are the following

First of all `expr -> add_expr : '$1'.`. 
If add_expr matches the input then we have an `expr.`
`add_expr -> add_expr add_op mult_expr : { binary_op, ?line('$1'), ?op('$2'), '$1', '$3' }.`
We know that from previous but it is now expanded to cover cases like  2+4 + 2*3.
It uses recursive definitions to accomplish this because
`add_expr` could be another `add_expr add_op mult_expr` or it could a `mult_expr` which then could be `mult_expr mult_op unary_expr` or could be a `unary_expr` which again could be
`max_expr` or a `unary_op max_expr` which again could be `number` or a `expr` which again could be all the above.

> Unary operators are `+1` or `-1` or just `1`

This new grammar can cover any mathematical expression.

Furthermore a lot new tests were added to arithmetic_test.erl to cover the new grammar.
Do a make test to check them out.


## `git checkout 50599762a8ad85d4254762a01fb3ccad2a4b873a`
### Basic assignment works.

Asignment operations added. No language is complete without assignment. 
Lets see how it is done.

First lets see the lexer
Assignment means variables which means we need to define what elixir sees as variables.
New Definitions in the lexer
```
UpperCase = [A-Z]
LowerCase = [a-z]
```

Which are used to define the variable names
```
({LowerCase}|_)({UpperCase}|{LowerCase}|{Digit}|_)* : { token, { var, TokenLine, list_to_atom(TokenChars) } }.
```

What this states is that a variable can start with a lowecase character or a _ and be followed by any lower or upper case character or number or _.
In that the lexer returns as usual a `var` as `Category`, `TokenLine` as usual and 
`list_to_atom(TokenChars)` to return the actual name of the variable as an erlang atom.

One more operator is added ofcourse and this is `=`<br>
`=     : { token, { '=', TokenLine } }.` Pretty easy stuff!

Now lets see what happens in the grammar rules.

Ofcourse new nonterminal and terminal operators are added `assign_expr`, `var` and `=`.
New important additions to the grammar
```
expr -> assign_expr : '$1'.

%% Assignment
assign_expr -> add_expr '=' assign_expr :
  { match, ?line('$2'), '$1', '$3' }.

assign_expr -> add_expr : '$1'.

%% Minimum expressions
max_expr -> var : '$1'.
max_expr -> number : '$1'.
max_expr -> '(' expr ')' : '$2'.
```

An expr can now be an assign_expr also.
And a minimum expression can also be a var.
The grammar here i believe is little weird because i am allowed to 
write 2+2=2+3 and it is not the parser that gives the exception but actual erlang  that 
does not accept this input. We can check that by entering this command to the erlang repl
`elixir_parser:parse(element(2,elixir_lexer:string("2+2=2+3"))).`
Not sure if this is intended so lets move on.



#v2
## `git checkout 4475b4aa92dac2dbe236a975b839cbf1f2acffb7`
### Allow multiline expressions.

```
%% Skip
{Comment} : skip_token.
{Whitespace}+ : skip_token.

%% Newlines (with comment and whitespace checks)
({Comment}|{Whitespace})*(\n({Comment}|{Whitespace})*)+ : { token, { eol, TokenLine } }.
```
We see here the definition of a new line 
Comment is not yet defined

The grammar is updated to accept a list of expression with the use of the new `eol` terminal
So new terminal `eol` and new nonterminals `grammar`, `expr_list`.
Rootsymbol is now `grammar`

```
grammar -> expr_list : '$1'.
grammar -> '$empty' : [].

expr_list -> eol : [].
expr_list -> expr : ['$1'].
expr_list -> expr eol : ['$1'].
expr_list -> eol expr_list : '$2'.
expr_list -> expr eol expr_list : ['$1'|'$3'].
```

What expr_list does here is recursively split the expressions that are between new lines
in an array of expressions. See that an expression with just a new line returns [].
All possible cases are covered. and you can now do things like elixir:eval("a = 1\nb = 1").
Check the tests for more.


## `git checkout 212e90321bdafe8f456ff72aae0a12223a1717cd`
### I HAZ FUNCTIONZ + CLOJUREZ. ( best commit message yet )
Functions can't wait..

First in the lexer we have the addition of
`->    : { token, { '->', TokenLine } }.` which is used for functions
and the variables now have a guard mechanism to not allow variables that are in the reserved_word list.

The new terminals for the parser are
1. ->
2. do
3. end
And nonterminals
1. fun_expr
2. body
3. stabber

Lets see what is going on

```
unary_expr -> fun_expr : '$1'.

fun_expr -> stabber expr :
  { 'fun', ?line('$1'),
    { clauses, [ { clause, ?line('$1'), [], [], ['$2'] } ] }
  }.

fun_expr -> stabber eol body 'end' :
  { 'fun', ?line('$1'),
    { clauses, [ { clause, ?line('$1'), [], [], '$3' } ] }
  }.

fun_expr -> max_expr : '$1'.

%% Function bodies
body -> expr_list : '$1'.

%% Stab syntax
stabber -> '->' : '$1'.
stabber -> 'do' : '$1'.
```

That states that a function definitions can be any of the following
1. -> expr
2. do expr
3. -> \n expr end
4. do \n expr end

Check the function_test.erl to see all the cool stuff you can do.
Event this  `elixir:eval("b = 1; a = -> b + 2")` is valid now!

A lot of interesting stuff now happen in elixir.erl which is used to transform the code to erlang ast and run it.

The transform functions define all operations returned from the parser.

So make compile and we are done 
Next go to a terminal and type erl to get into erlang repl

 ### First official release : 6052352b6281752b905b30eb5b08fac0f51f68cd
 - Date May 24, 2012