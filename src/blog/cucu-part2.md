title: "cucu: a compiler you can understand (2/3)"
description: Compilers is fun. Want to write your own one?
keywords: compiler, programming, C, smallC, easy, tiny, small, tutorial, parser, variables
date: 2012-10-24
---
cucu: a compiler u can understand (part&nbsp;2)
==============================================

So far, we have defined language grammar and have written a lexer. In this part 
we will write a parser for our language. Before we start, we need some helper 
functions:

	int peek(char *s) {
		return (strcmp(tok, s) == 0);
	}

	int accept(char *s) {
		if (peek(s)) {
			readtok();
			return 1;
		}
		return 0;
	}

	int expect(char *s) {
		if (accept(s) == 0) {
			error("Error: expected '%s'\n", s);
		}
	}

`peek()` returns non-zero value if the next token is equal to the given string.
`accept()` reads the next token, if it's equal to the given string, otherwise it
 returns 0. And `expect()` helps us to check language syntax.

the harder part
---------------

As you can see from the language grammar, statements and various expression 
types are strongly interconnected. It means we have to write all parser 
functions at once, keeping in mind the recursion. Let's go again from top
to bottom. Here's our top-level compiler() functions:

	static int typename();
	static void statement();

	static void compile() {
		while (tok[0] != 0) { /* until EOF */
			if (typename() == 0) {
				error("Error: type name expected\n");
			}
			DEBUG("identifier: %s\n", tok);
			readtok();
			if (accept(";")) {
				DEBUG("variable definition\n");
				continue;
			} 
			expect("(");
			int argc = 0;
			for (;;) {
				argc++;
				typename();
				DEBUG("function argument: %s\n", tok);
				readtok();
				if (peek(")")) {
					break;
				}
				expect(",");
			}
			expect(")");
			if (accept(";") == 0) {
				DEBUG("function body\n");
				statement();
			}
		}
	}

It reads type name, then an identifier. If it's followed by a semicolon -
it's a variable declaration. If it's followed by a paren - it's a function.
Function scans function arguments one by one, and if function is not
followed by a semicolon - it's a definition (function with a body), otherwise - 
it's just a declaration (just function name and prototype).

Here, `typename()` is function that just skips the valid type name. We accept
only `int` and `char` and various pointers to them (`char *`):

	static int typename() {
		if (peek("int") || peek("char")) {
			readtok();
			while (accept("*"));
			return 1;
		}
		return 0;
	}

The most interesting part is the `statement()` function. It parses a single 
statement, which can be a block, a local variable definition/declaration, 
a `return` statement etc. Here how it should look like:

	static void statement() {
		if (accept("{")) {
			while (accept("}") == 0) {
				statement();
			}
		} else if (typename()) {
			DEBUG("local variable: %s\n", tok);
			readtok();
			if (accept("=")) {
				expr();
				DEBUG(" :=\n");
			}
			expect(";");
		} else if (accept("if")) {
			/* TODO */
		} else if (accept("while")) {
			/* TODO */
		} else if (accept("return")) {
			if (peek(";") == 0) {
				expr();
			}
			expect(";");
			DEBUG("RET\n");
		} else {
			expr();
			expect(";");
		}
	}

So, if it's a block `{ .. }` - just read statements until end of block is met.
If it starts with a type name - it's a local variable. Conditional statements 
("if/then/else") and loops are just stubs for now. Think of how you would
implement them according to the grammar we use.

Anyway, most of the statement contain expressions inside. So, we need to make a
function that parses an expression. Expression parser is a recursive descent 
parser, so it's a number of functions that call each other recursively until 
primary expression is found. Primary expression as we can see from the grammar
is a number (constant) or an identifier (variable or function).

	static void prim_expr() {
		if (isdigit(tok[0])) {
			DEBUG(" const-%s ", tok);
		} else if (isalpha(tok[0])) {
			DEBUG(" var-%s ", tok);
		} else if (accept("(")) {
			expr();
			expect(")");
		} else {
			error("Unexpected primary expression: %s\n", tok);
		}
		readtok();
	}

	static void postfix_expr() {
		prim_expr();
		if (accept("[")) {
			expr();
			expect("]");
			DEBUG(" [] ");
		} else if (accept("(")) {
			if (accept(")") == 0) {
				expr();
				DEBUG(" FUNC-ARG\n");
				while (accept(",")) {
					expr();
					DEBUG(" FUNC-ARG\n");
				}
				expect(")");
			}
			DEBUG(" FUNC-CALL\n");
		}
	}

	static void add_expr() {
		postfix_expr();
		while (peek("+") || peek("-")) {
			if (accept("+")) {
				postfix_expr();
				DEBUG(" + ");
			} else if (accept("-")) {
				postfix_expr();
				DEBUG(" - ");
			}
		}
	}

	static void shift_expr() {
		add_expr();
		while (peek("<<") || peek(">>")) {
			if (accept("<<")) {
				add_expr();
				DEBUG(" << ");
			} else if (accept(">>")) {
				add_expr();
				DEBUG(" >> ");
			}
		}
	}

	static void rel_expr() {
		shift_expr();
		while (peek("<")) {
			if (accept("<")) {
				shift_expr();
				DEBUG(" < ");
			}
		}
	}

	static void eq_expr() {
		rel_expr();
		while (peek("==") || peek("!=")) {
			if (accept("==")) {
				rel_expr();
				DEBUG(" == ");
			} else if (accept("!=")) {
				rel_expr();
				DEBUG("!=");
			}
		}
	}

	static void bitwise_expr() {
		eq_expr();
		while (peek("|") || peek("&")) {
			if (accept("|")) {
				eq_expr();
				DEBUG(" OR ");
			} else if (accept("&")) {
				eq_expr();
				DEBUG(" AND ");
			}
		}
	}

	static void expr() {
		bitwise_expr();
		if (accept("=")) {
			expr();
			DEBUG(" := ");
		}
	}

It's a big piece of code, but don't be afraid - it's really simple.
Every function that parses expression type first tries to call a
more prioritized expression parser. Then, if an expected operator is found - 
it calls more prioritized expression parser again. Now it has parsed both
parts of a binary expression (like x+y, or x&y, or x==y), so it can perform
an operation and return. Some expression can be "chained" (like a+b+c+d), so 
we parse them with loops.

We put debug output after every expression parser function. This will give us 
an interesting result. For example, if we parse this piece of code:

	int main(int argc, char **argv) {
		int i = 2 + 3;
		char *s;
		func(i+2, i == 2 + 2, s[i+2]);
		return i & 34 + 2;
	}

we will get this ouput:

	identifier: main
	function argument: argc
	function argument: argv
	function body
	local variable: i
	 const-2  const-3  +  :=
	local variable: s
	 var-func  var-i  const-2  +  FUNC-ARG
	 var-i  const-2  const-2  +  ==  FUNC-ARG
	 var-s  var-i  const-2  +  []  FUNC-ARG
	 FUNC-CALL
	 var-i  const-34  const-2  +  AND RET

All our expressions are written in a postfix form (instead of `2+3` it's `2 3
+`).  This is a natural form for stack machines, when operands are placed on
the stack, then a function called pops up the operands, processes them and puts
the result back on the stack.

Though it might not be an optimal architecture for most modern CPUs, which are
register-based, it's still very simple and fits our compiler needs.

symbols
-------

Ok, we are good. We've got a lexer and a parser in less than 300 lines of code.
What we need to do is to add some functions to work with the symbols (like
variable names, or functions). A compiler should have a table of symbols to
quickly find their addresses, so when you write "i = 0" - it means put zero
into the location at address 0x1234 in RAM (if symbol "i" has address 0x1234 in
memory).
Also, when you call "func()" it means - jump to address 0x5678 (if symbol "func"
has value of 0x5678).

We use the following structure for symbols:

	struct sym {
		char type;
		int addr;
		char name[];
	};

Here `type` has special meaning. We use a single-letter codes to detect symbol 
type:

* `L` - is a local variable. `addr` stores variable location on the stack
* `A` - function argument. `addr` also stores the location on the stack
* `U` - undefined global variable. `addr` stores absolute address in RAM.
* `D` - defined global variable. Same as above.

So far, I've added two functions: `sym_find(char *s)` to find symbol by its 
name, and `sym_declare()` to add a new symbol.

Now we're are ready to develop [backend architecture &rarr;][1]

_Check [part 1](/blog/cucu-part1.html) if you missed something_



[1]: /blog/cucu-part3.html

