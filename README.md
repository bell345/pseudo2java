# pseudo2java

A converter from object-oriented PASCAL-like pseudo code to Java source
code. The dialect has been designed to map cleanly from pseudo-code to
Java while allowing for reasonable shortcuts.

## Features

* Error handling and reporting
* Capable of object oriented code

## Installation

From the repository root, run:

    pip3 install .

or the following to install as a user package (making sure to add ~/.local/bin/ to your $PATH):

    pip install . --user

## Usage

Syntax:

    pseudo [file_name] [--trace output_trace.txt]

If run without a file name, you will be introduced into an interactive shell
where you can directly input programs.

## Syntax

_NOTE_: Further changes could be implemented at any time.

### Notes on the Meta-Syntax Notation

I originally attempted to mimic
[Backus-Naur Form](https://en.wikipedia.org/wiki/Backus-Naur_Form),
while at the same time extending some functionality.

A reference:

    > xyz <         = the character/sequence 'xyz'
    'xyz'           = the character/sequence 'xyz'
    a : b c ;       = a is defined as b followed by c, where b and c are separated by whitespace
    stmt1 | stmt2   = stmt1 or stmt2
    ['a' - 'z']     = any character between 'a' and 'z', including 'a' and 'z'
    [statement]*    = statement 0, 1 or more times
    [statement]+    = statement at least once
    [statement]?    = optional statement

### Syntax Units

The basic units of syntax are as follows:

    digit           : ['0' - '9']
                    ;

    letter          : ['A' - 'Z'] | ['a' - 'z']
                    ;

    hex_digit       : ['A' - 'F'] | ['a' - 'f'] | ['0' - '9']
                    ;

    stmt_end        : ':'
                    | ';'
                    | '\r\n'
                    | '\n'
                    ;

    ident_begin     : letter | '_'
                    ;

    ident_symbol    : ident_begin | ['0' - '9']
                    ;

    identifier      : ident_begin [ident_symbol]*
                    ;

    qual_name       : identifier
                    | qual_name '.' identifier

    string_char     : (any character not including > \ <, > " < or > ' < )
                    | '\r' | '\n'
                    | '\x' hex_digit hex_digit
                    | '\u' hex_digit hex_digit hex_digit hex_digit
                    | > \\ <
                    | > \' <
                    | > \" <
                    ;

    string          : > " < [string_char]* > " <
                    | > ' < [string_char]* > ' <
                    ;

    dec_literal     : [digit]+
                    ;

    hex_literal     : '0x' [hex_digit]+ | '0X' [hex_digit]+
                    ;

    sign            : '-' | '+'
                    ;

    exp_part        : 'e' [sign]? [digit]+
                    | 'E' [sign]? [digit]+
                    ;
    float_literal   : [digit]+ '.' [digit]* [exp_part]?
                    | [digit]+ exp_part
                    ;

    int_literal     : dec_literal | hex_literal
                    ;

    number          : dec_literal | hex_literal | float_literal
                    ;

### Comments

Comments are sections of text which are reproduced as Java comments in
the output. The `ASSERT:` syntax is designed to allow for pseudo-assert
statements, which are merely copied as comments that start with `ASSERT` for
now.

    comment         : '#' (any character except '\n')* '\n'
                    | '//' (any character except '\n')* '\n'
                    | 'ASSERT:' (any character except '\n')* '\n'
                    ;

### Types

All variables must have a type in Java - however, this is not necessarily
so in pseudocode. At the moment, type declarations or initialisations 
with types are required before use. Until automatic type inference is 
implemented, the plain initialisation form will not work and will throw 
an error.

    type_name       : qual_name
                    | array_type_name
                    ;

    array_type_name : 'ARRAY OF' type_name
                    | 'ARRAY OF' dec_literal type_name

    declaration     : identifier 'AS' type_name stmt_end
                    | identifier ':=' decl_rhs stmt_end
                    | identifier 'AS' type_name ':=' decl_rhs stmt_end
                    ;

    decl_rhs        : expression 
                    | array 
                    | array 'AS' array_type_name
                    ;

### Arrays

Arrays function similarly to those in Java. The literal array cannot be
used freely - only in pure assignment.

    argument_list   : argument_list ',' expression
                    | expression
                    ;

    array_item      : expression
                    | array
                    ;

    array           : '[' argument_list ']'
                    ;

### Expressions

Expressions aim to emulate those used in Java, with some verbose options
that can more clearly communicate intent. The complete expression syntax
can be found [here][java_bnf], with some changes:

* Assignments cannot be present in expressions.
  * `expression : conditional_expression;`
* There are various synonyms for existing operators:
  * `equality_op : '=' | '==' | 'EQ' ;`
  * `non_equality_op : '!=' | 'NEQ' ;`
  * `lt_operator : '<' | 'LT' ;`
  * `gt_operator : '>' | 'GT' ;`
  * `le_operator : '<=' | 'LE' ;`
  * `ge_operator : '>=' | 'GE' ;`
  * `logical_and_op : '&&' | 'AND' ;`
  * `logical_or_op : '||' | 'OR' ;`
  * `unary_not_op : '!' | 'NOT' ;`
  * `modulus_op : '%' | 'MOD' ;`
* Convenience operators:
  * 'DIV' has same precedence as '*', '/', '%' (integer division)
* Object creation can have an alternative form:

    class_instance_creation_expression :
                      'new' class_type '(' [argument_list]? ')'
                    | create_kw class_type 
                    | create_kw class_type using_kw argument_list 
                    ;

    create_kw       : 'CREATE'
                    | 'CONSTRUCT'
                    ;

    using_kw        : 'USING'
                    | 'WITH'
                    ;

### Classes

Classes are the bread and butter of Java programming. The declaration
supports `extends`, `implements`, `public`, `abstract` and `final`.

Note that `CONST`, `CONSTANT` are synonyms for `FINAL`, and that the plain
form of `field_decl` without a type may not be supported until type inferencing
is implemented.

    class           : class_decl [begin_stmt]? [class_content]* end_stmt
                    ;

    class_decl      : [class_modifier]? 'CLASS' identifier [class_extend]?
                      [class_iface]* stmt_end
                    ;

    class_modifier  : 'PUBLIC' | 'ABSTRACT' | 'FINAL'
                    | 'CONST' | 'CONSTANT'
                    ;

    class_extend    : 'EXTENDS' identifier
                    ;

    class_iface     : 'IMPLEMENTS' identifier
                    ;

    stmt_end        : ';'
                    | '\r\n'
                    | '\n'
                    ;

    begin_stmt      : 'BEGIN' stmt_end
                    ;

    end_stmt        : 'END' stmt_end
                    | 'END' identifier stmt_end
                    ;

    class_content   : field_decl
                    | method
                    | constructor
                    | destructor
                    ;

    field_decl      : identifier 'AS' [field_modifier]* type_name stmt_end
                    | identifier 'AS' [field_modifier]* type_name
                      ':=' decl_rhs stmt_end
                    | identifier ':=' decl_rhs stmt_end
                    ;

    field_modifier  : 'PUBLIC' | 'PROTECTED' | 'PRIVATE' | 'STATIC'
                    | 'FINAL' | 'CONSTANT' | 'CONST'
                    | 'TRANSIENT' | 'VOLATILE'
                    ;

### Methods

Within classes, methods are blocks of code that can be called repeatedly.
Every method requires a contract - that is, a specification of its inputs and
outputs. In Java, the inputs correspond to the method parameters and the
outputs correspond to the return values. At the moment, only one output
is supported for each method, and it may be omitted or given a 'NONE' value.

Constructors are special methods within a class that take imports with no
exports.

At the moment, 'throws' is not supported.

    method          : [method_mod]* 'METHOD' identifier end_stmt
                      [import_stmt]* export_stmt begin_stmt
                      [statement]* end_stmt

    method_mod      : 'PUBLIC' | 'PROTECTED' | 'PRIVATE' | 'STATIC'
                    | 'ABSTRACT' | 'FINAL' | 'CONSTANT' | 'CONST'
                    | 'SYNCHRONIZED' | 'NATIVE'
                    ;

    import_stmt     : 'IMPORT' identifier 'AS' type_name stmt_end
                    ;

    export_stmt     : 'EXPORT' identifier 'AS' type_name stmt_end
                    ;

    constructor     : [construct_mod]* 'CONSTRUCTOR' end_stmt
                      [import_stmt]* begin_stmt
                      [statement]* end_stmt
                    ;

    construct_mod   : 'PUBLIC' | 'PROTECTED' | 'PRIVATE'
                    ;

### Programs

Programs map to public classes with an automatic `public static void main()`
method.

    program         : program_decl begin_stmt [statement]* end_stmt
                    ;

    program_decl    : 'PROGRAM' identifier stmt_end
                    ;

### Statements

Statments can be either assignment, selection, iteration, jump or I/O
statments, an expression such as a module call, or a comment (so that they
can be preserved in the Java output):

    statement       : assignment_stmt stmt_end
                    | decl_stmt stmt_end
                    | selection_stmt stmt_end
                    | iteration_stmt stmt_end
                    | jump_stmt stmt_end
                    | io_stmt stmt_end
                    | expression stmt_end
                    | comment
                    ;

### Declarations

A declaration creates a variable with a specified type, and optionally an
initial value. In Java, all variables require declaration before use. In this
language, variables may not support use without declaration (like Java) until
automatic type inferencing is implemented.

    decl_stmt       : identifier 'AS' type_name
                    | identifier 'AS' type_name ':=' decl_rhs
                    ;

### Assignment Statements

Assignment statements assign a value to a variable. If a variable is referenced
without being assigned a value, an error will be raised.

    assigment_stmt  : identifier ':=' expression
                    ;

### Selection Statements

Selection statements conditionally execute other statements based upon an
expression. The switch statement works a bit differently than Java/C: each
case can have more than one match, and all cases have an automatic `break;`
appended to the end.

    selection_stmt  : if_stmt [statement]* end_stmt
                    | if_stmt [statement]* else_block end_stmt
                    | switch_stmt [case_block]* [otherwise_block]? end_stmt
                    ;

    if_stmt         : 'IF' expression [then_kw]? stmt_end
                    ;

    else_block      : 'ELSE' stmt_end [statement]*
                    | 'ELSE' if_stmt [statement]* [else_block]?
                    ;

    then_kw         : 'THEN'
                    | 'DO'
                    ;

    switch_stmt     : 'SWITCH' expression stmt_end
                    ;

    case_block      : 'IN CASE' int_literal stmt_end [sub_case]*
                      [expression]*
                    ;

    sub_case        : 'OR CASE' int_literal stmt_end
                    ;

    otherwise_block : 'OTHERWISE' stmt_end [expression]*
                    ;

### Iteration Statements

Iteration statements execute the same set of statements over and over
conditionally or for a certain number of times.

    iteration_stmt  : while_stmt [statement]* repeat_stmt
                    | for_stmt [statement]* repeat_stmt
                    | 'DO' stmt_end [statement]* while_stmt
                    ;

    while_stmt      : 'WHILE' expression [then_kw]? stmt_end
                    ;

    for_stmt        : 'FOR' assignent_stmt 'TO' expression [then_kw]? stmt_end
                    ;

    repeat_stmt     : 'REPEAT' stmt_end
                    | 'NEXT' stmt_end
                    | end_stmt
                    ;

### Jump Statements

Jump statements change the control flow unconditionally. In this pseudo-code
transpiler, most uses of jump statements are advised against.

In the following conditions, a warning is emitted:

* `BREAK` is used at all
    * note: for `SWITCH` statements, `break;` is appended automatically to all
      cases
* `CONTINUE` is used at all
* `RETURN` is used in a program
* `RETURN` is used more than once in a module
* `RETURN` is not at the end of a module with a return value


    jump_stmt       : 'BREAK' stmt_end
                    | 'CONTINUE' stmt_end
                    | 'RETURN' expression stmt_end
                    ;

### I/O Statements

I/O statements are the methods of input and output that pseudo programs
have available to them. These map to functions on System.out.
The `FORMAT WITH` syntax maps to System.out.printf, with the same string
format.

    io_stmt         : 'INPUT' identifier 'AS' type_name
                    | 'OUTPUT' expression
                    | 'OUTPUT' expression 'FORMAT WITH' argument_list 
                    ;
## Examples

Examples of (hopefully) valid pseudo-code programs that can be ran with
the converter are in the `test/` directory.

## Planned Improvements

* Import support

(C) Thomas Bell 2016-2019, MIT License.

[java_bnf]: https://users-cs.au.dk/amoeller/RegAut/JavaBNF.html
