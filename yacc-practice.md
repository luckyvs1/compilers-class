---
layout: default
img: green_dragon_book
img_link: "https://en.wikipedia.org/wiki/Principles_of_Compiler_Design"
caption: "Principles of Compiler Design, by Alfred Aho and Jeffrey Ullman, published in 1977, is the classic textbook on compilers."
title: Yacc Practice
active_tab: practice
---

## Parsing with Yacc/Bison: Practice Problems 

### Clone the repository

The files below and other example programs are available in a git repository

You must have git and python (3.x) on your system. Once you've confirmed this, run this command:

    git clone https://github.com/anoopsarkar/compilers-class-hw.git

If you have already cloned the repository earlier you can
get the new homework files by going to the directory
where you cloned the repository earlier and then doing:

    # go to the directory where you did a git clone earlier
    git pull origin master

Then go to the `yacc-practice` directory

    cd /your-path-to/compilers-class-hw/yacc-practice

### Getting started

Let's start with a simple lexical analyzer that we will use for our first yacc program.
Save the following Lex program to a file called `parens.lex`

    %%
    [ \t]+ { }
    .  return yytext[0];
    %%

To specify a context-free grammar (CFG) in yacc we write the left-hand side
of the rule just before the `:` followed by the right-hand side of each
rule. Rules with the same left-hand side are grouped together using
the alternation symbol `|`. Save the following Yacc program to a file
called `parens.y`:

    %%
    S: '(' S ')'
     |
     ;

The above grammar has two CFG rules, which we usually write as:

    S -> "(" S ")"
    S -> <epsilon>

In Backus-Naur notation we write the two rules together using `|`:

    S -> "(" S ")" | <epsilon>

where `<epsilon>` is the empty string and in Yacc grammars
this is represented as having an empty right-hand side. So
the Yacc grammar equivalent of the above grammar is:

    S: '(' S ')' | ;

The ';' represents the end of all the `S` rules in the grammar.

To create a C or C++ program we will use `bison` (the GNU version of
`yacc`) and `flex` (the GNU version of `lex`). Let's using `bison`
and `flex` to generate a C version of the parser for the CFG above and compile it:

    bison -oparens.tab.c -d parens.y
    flex -oparens.lex.c parens.lex
    gcc -w -o ./parens parens.tab.c parens.lex.c -ly -ll

It will accept strings with balanced parentheses where no close paren
can precede an open paren:

    $ ./parens
    (())

    (()
    syntax error

Similar to how we can attach actions to regular expressions in Lex
we can also attach actions to CFG rules in Yacc:

    %%
    S: '(' S ')' { $$ = $2+1; printf("%d\n", $$); }
     | { $$ = 0; }
     ;

In the above Yacc grammar, the symbol `$2` refers to the value
of the second element in the right-hand side of the rule in the
context of a parse tree for some input string. In the above rule
`S: "(" S ")"` the `$2` refers to the value `S` in a parse tree.
`$$` refers to the value created for the left-hand side and it
is passed up the parse tree. After we re-compile the new Yacc
file, we see the following behaviour:

    $ ./parens
    (())
    1
    2

    ((()))
    syntax error

Let's look at a schematic for the parse tree for `(())`:

    S: $1: "(" 
       $2: S: $1: "("
              $2: S:  { $$ = 0; } # no print, sets LHS to 0
              $3: ")"
            { $$ = $2+1; printf("%d\n", $$); } # prints 1, sets LHS to 1 
       $3: ")"
         { $$ = $2+1; printf("%d\n", $$); } # prints 2, sets LHS to 2

Here are some exercises to test your understanding:

1. Add a new CFG rule `TOP: S` to the top of the Yacc grammar (the order is important).
2. Does anything change in the strings this new Yacc grammar accepts?
3. What is the value of `$1`  in the rule `TOP: S`?
4. Add an action to the rule to print out `$1` to see if you could predict the right value.

### Simple Expression Interpreter

The yacc program below is a very simple (and incomplete) 
expression interpreter. The `{ ... }` section contains arbitrary C/C++ code and the 
`%token` definitions is a list of tokens provided by
the lexical analyzer. 

    %{ 
    #include <stdio.h> 
    extern int yylex(void);
    extern int yyerror(char *);
    %}
    %token NAME NUMBER
    %%
    statement: NAME '=' expression { printf("%c = %d\n", $1, $3); }
         | expression  { printf("%d\n", $1); }
         ;

    expression: expression '+' NUMBER { $$ = $1 + $3; }
         | expression '-' NUMBER { $$ = $1 - $3; }
         | NUMBER { $$ = $1; }
         ;
    %%

If you save the above program to a file called `simple-expr.y` then
the program `bison` can be used to convert this parser definition
into a parser implementation by using the following command:

    bison -osimple-expr.tab.c -d simple-expr.y

The `-d` option produces a header file called `simple-expr.tab.h`
to convey information about the tokens to the lexical analyzer. Examine
the contents of this file. 

The lexical analyzer is shown below as a lex program. 

    %{
    #include "simple-expr.tab.h"
    #include <stdlib.h>
    extern int yylval;
    %}
    %%
    [0-9]+   { yylval = atoi(yytext); return NUMBER; }
    [a-z]    { yylval = yytext[0]; return NAME; }
    [ \t\n]  /* ignore whitespace */
    .        return yytext[0];
    %%


Save it to a file called `simple-expr.lex`.
The lex program can be compiled to a C program using `flex`:

    flex -osimple-expr.lex.c simple-expr.lex

The final binary is created by compiling the output
from flex and bison with a C/C++ compiler as follows. 

    cc -o ./simple-expr simple-expr.tab.c simple-expr.lex.c -ly -lfl
    echo "a=2+3+5" | ./simple-expr

If you see the following error:

    ld: library not found for -lfl
    error: linker command failed with exit code 1 (use -v to see invocation)

Then use `-ll` instead of `-lfl`. And if that doesn't work then you
probably need to install bison and flex on your local machine. The
CSIL Linux machines should have these tools installed already.

Convert the above yacc and lex programs so that it can handle
multiple expressions, exactly one per line.
You will need a recursive context-free rule in the yacc definition
in order to handle multiple lines
of input. Try different ways of writing this recursive rule.
Note that we can assign a value to a variable, e.g `b=5+10-5` but
we cannot yet use `b` in a following expression, e.g. `a=b+10`
(which fails with a syntax error).

You can automate the creation of the binary using a `makefile`. `make` is very finicky about whitespace, especially the actions have to be indented using a literal tab character. To avoid headaches download the `makefile` directly by using the `view raw` link below.
<script src="https://gist.github.com/anoopsarkar/8aebb39f2ef1bb939a4b2e875172e025.js"></script>

Run `make simple-expr` assuming you used the same filenames suggested above. Then you can
test it with various inputs:

    echo "12" | ./simple-expr
    echo "a=2+3+5" | ./simple-expr

The yacc and lex code above does not yet handle assignments to
variables. In order to implement this, we need two different kinds
of values to be returned from the lexical analyzer: one for numbers,
and another for variable names. The lex code below shows you how to do
that. Save this lex code to the file `simple-varexpr.lex`.

    %{
    #include "simple-varexpr.tab.h"
    #include <math.h>
    %}

    %%
    [0-9]+    { yylval.rvalue = atoi(yytext); return NUMBER; } /* convert NUMBER token value to integer */
    [ \t\n]   ;  /* ignore whitespace */
    [a-z]     { yylval.lvalue = yytext[0] - 'a'; return NAME; } /* convert NAME token into index */
    .         return yytext[0];
    %%

For numbers it returns `yylval.rvalue` and for variable names it
returns `yylval.lvalue`. The two types of return value, `rvalue`
and `lvalue` are defined in the yacc program `simple-varexpr.y`
shown below.

    %{
    #include <stdio.h>
    #include <stdbool.h>
    int symtbl[26];
    bool issym[26];

    int yylex(void);
    int yyerror(char *);
    %}

    %union {
      int rvalue; /* value of evaluated expression */
      int lvalue; /* index into symtbl for variable name */
    }

    %token <rvalue> NUMBER
    %token <lvalue> NAME 

    %type <rvalue> expression

    %%
    statement: NAME '=' expression { symtbl[$1] = $3; issym[$1] = true; }
             | expression  { printf("%d\n", $1); }
             ;

    expression: expression '+' NUMBER { $$ = $1 + $3; }
             | expression '-' NUMBER { $$ = $1 - $3; }
             | NUMBER { $$ = $1; }
             ;
    %%

The two different return values are represented using the `%union`
declaration which is the same concept as the [union type in C and C++](https://en.wikipedia.org/wiki/Union_type). 
The `%union` declaration can include complex datatypes.
The yacc code defines a type not just for the tokens, but also for
nonterminals, which is specified in the `%type` definition.  This
allows yacc to check that the type of the non-terminal expression
is `rvalue`, an integer type.

#### Your Task

Extend this code in two ways:

* First, add the ability to process multiple statements so that the following works:
    echo "a=10\nb=10" | ./simple-varexpr
* Second, properly handle assignments of values to variables. The
variable on the left hand side of an equation will be an $$\ell$$_-value_
and the variable used on the right hand side of an equation will
be a _r-value_.

### Adding Functions to your Expression Interpreter

Extend your expression interpreter to include constants of type `double`,
and variables that can hold either integer or double types. Finally, add 
the functions: `exp`, `sqrt`, `log` so that you can interpret
the following types of input:

    a = 2.0
    b = exp(a)
    b

To avoid issues with precedence of operators use the yacc grammar provided
below:

    %token T_DOUBLE T_NUMBER T_NAME T_EXP T_SQRT T_LOG

    %%
    statement_list : statement '\n' statement_list
       |
       ;

    statement: T_NAME '=' expression
       | expression
       ;

    expression: expression '+' T_NUMBER
       | expression '-' T_NUMBER
       | expression '+' T_DOUBLE
       | expression '-' T_DOUBLE
       | expression '+' T_NAME 
       | expression '-' T_NAME 
       | T_NUMBER
       | T_DOUBLE
       | T_NAME 
       | T_EXP '(' expression ')'
       | T_SQRT '(' expression ')'
       | T_LOG '(' expression ')'
       ;

    %%

### Writing a Decaf Program

Read the [Decaf specification](decafspec.html) and write a Decaf
program that implements the quicksort algorithm to sort a list.
Create an array variable `list` with 100 elements. Then initialize
it using the following Decaf loop:

    func initList() void {
        var i int;
        for (i = 0; i < 100; i = i + 1) {
            list[i] = (i * 2382983) % 100;
        }   
    }   

Sort this list using quicksort and then print the sorted list by
iteratively calling the `print_int` library function in the format
shown in the Homework 2 `testcases` directory.

### Expression interpreter

Write down a yacc parser for the following context-free grammar:

    e -> e PLUS t
    e -> t
    t -> t TIMES f
    t -> f
    f -> LPAREN e RPAREN 
    f -> ID

The tokens `PLUS, TIMES, LPAREN, RPAREN` are defined to be `+`, `*`, `(`, `)` respectively.
And the token ID is defined to be an identifier as in the [Decaf specification](decafspec.html).
These tokens should be defined using a lexical analyzer produced using lex.

For the input string `x + y * ( z )` the output produced by the
yacc parser should be the parse tree for the input string in the
format shown below. The tree below is indented but you should simply
print your parse tree as a single line of output text.

    (e (e (t (f (ID x))))
       (PLUS +)
       (t (t (f (ID y)))
          (TIMES *)
          (f (LPAREN \()
             (e (t (f (ID z))))
             (RPAREN \)))))

Note the backslash preceding each instance of a literal parenthesis
to avoid confusion with the parentheses used to denote the tree
structure.  You may need to augment the grammar to produce the right
output.

### Marking Precedence and Associativity in Yacc

Use the following expression grammar in Yacc:

    %{
    #include <stdio.h>
    #include <ctype.h>
    #include <stdlib.h>
    %}

    %token ID

    %%

    top: e ';' { printf("\n"); exit(EXIT_SUCCESS); }

    e: e '+' e { printf("+"); }
     | e '-' e { printf("-"); }
     | e '*' e { printf("*"); }
     | e '/' e { printf("/"); }
     | '(' e ')'
     | ID  { printf("%c", $1); }
     | '-' e { printf("-"); } 
     ;

    %%

    yylex() {
        int c=getchar();
        while ((c == ' ') || (c == '\n') || (c == '\t')) {
            /* skip whitespace */  
            c = getchar();
        }
        if (islower(c)) {
            yylval = c;
            return(ID);
        }
        return(c);
    }

If you run it through `bison` you will get shift/reduce conflicts:

    $ bison expr-conflicts.y 
    expr-conflicts.y: conflicts: 20 shift/reduce

Add precedence declarations just below the `%token` line in the
yacc program. Precedence is added using the `%left` or `%right`
associativity declaration for tokens. The precedence is marked
by having a list of such declarations with the highest precedence
token appearing at the bottom of the list. So if we wish `*` to
have higher precedence than `-` we would write:

    %left '-'
    %left '*'

Two tokens can be declared with the same precedence and left
associativity by declaring them on the same line:

    %left '+' '-'

However, this does not solve the problem of the unary minus
production. Unary minus should have a precedence higher than
multiplication or division. To mark this we have to add a
precedence annotation on the rule itself:

    e: '-' e %prec UMINUS

Then we can add this to our list of precedence  

    %left '-'
    %left '*'
    %left UMINUS

Modify the yacc program above to eliminate all shift/reduce conflicts
using precedence and associativity declarations in yacc.

### Inherited Attributes

For the following context-free grammar in yacc format:

    block : '{' var-decl-list '}'
    var-decl-list : var-decl var-decl-list 
        |
        ;
    var-decl : type id-comma-list ';'
    id-comma-list : id ',' id-comma-list 
        | id 
        ;
    type : 'int' 
        | 'bool'
        ;

Provide a yacc program that passes the type information for each variable
by using _inherited attributes_. The program should enter each 
variable name into a symbol table along with its type information.

