title: "cucu: a compiler you can understand (3/3)"
description: Compilers is fun. Want to write your own one?
keywords: compiler, programming, C, smallC, easy, tiny, small, tutorial, parser, code, generate, virtal machine
date: 2012-10-25
---
Let's talk about compiler backends. C should be a portable language, and there
is no need to rewrite the whole compiler if you want to port it to the new CPU
architecture.

Backend is a part of the compiler that generates low-level byte code. Compiler
itself just calls backend functions. Good backend design makes the compiler
highly portable.

I wanted CUCU to be a portable compiler (actually, a cross-compiler).
So, I decided to move backend code generator to a separate module.

But before we dive into the backend code generation, let's think of how we will
test it.

minimal target architecture
---------------------------

Our minimal target has two registers (let's call them A and B), and a stack.
Register A is an accumulator. We could use fixed-size instruction codes, as
many RISC CPUs do, but there's not much fun in decoding hexadecimal numbers to
find out what the code actually does.

I have chosen a simpler way. Every instruction is 8 bytes long (yes, it's huge,
but who cares - it's a test imaginary architecture). And the first 7 bytes of
the instruction are just ASCII symbols, and the last one is 0x10 ('\n').

This allows us to use human-readable instruction codes, like `A:=A+B`,
`A:=1ef8`, or `push A`. These seem to be self-explanatory ("add register B 
to register A", "put 0x1ef8 to register A" and "push the value of register A
to the stack").

* `A:=NNNN` - put value of 0xNNNN to register A
* `A:=m[A]` - put value at address stored in register A to register A (as byte)
* `A:=M[A]` - put value at address stored in register A to register A (as int)
* `m[B]:=A` - store the value of A to address stored in B (as byte)
* `M[B]:=A` - store the value of A to address stored in B (as int)
* `push A` - push the value of A on the stack
* `pop B` - pop the value from the stack to B
* `A:=B+A` - add A and B
* `A:=B-A` - subtract A and B
* `A:=B&A` - bitwise AND operation
* `A:=B|A` - bitwise OR operation
* `A:=B!=A` - A is 1 if B!=A, and 0 otherwise
* `A:=B==A` - A is 1 if B==A, and 0 otherwise
* `A:=B<<A` - shift left the value of B to A bits
* `A:=B>>A` - shift right the value of B to A bits
* `A:=B<A` - A is 1 if B&lt;A, and 0 otherwise
* `popNNNN` - drop NNNN items from the stack
* `sp@NNNN` - put the value at address NNNN on the stack to the register A
* `jmpNNNN` - jump to address NNNN
* `jpzNNNN` - jump to address NNNN if A is zero
* `call A` - call function at address stored in A
* `ret`    - return from the function

cucu backend design
-------------------

We include "gen.c" file, which is actually a backend implementation.
Let's start with simple functions: `gen_start()` and `gen_finish()`.
They are used to generate application prologue (like PE header, or ELF header)
and to post-process byte code.

Compiler provides the function `emit()`, that emits byte code to the `code[]`
array. At the very end, `code[]` contains a ready-to-use compiled program.

So, compiler calls backend function, and backend just calls emit() with the
specific byte-codes, and this is how we get compiled machine code.

Now we need to define what are the most common constructions, that backend 
should implement. Let's start with this simple program:

	int main() {
		return 0;
	}

Let's talk about calling convention. This is how arguments are passed to the 
function and how the return value is fetched. We stated before, that arguments
are placed on the stack (the 1st argument is pushed 1st). Let's make a deal, 
that the value of the register A when function returns is its return value.

Actually, we will store all values to register A. Register B will be used as
a temporary register.

For the program above I would expect the byte code to be something like:
	
	A:=0000
	ret

So, we need a function to put immediate numeric value to the register A, and a 
function to do the return. We will call them `gen_const(int)` and `gen_ret()`.

`gen_const` will be called every time the compiler finds a primary expression 
which is a number, and `gen_ret` is called when the compiler finds a return
statement. Though, some functions can be `void`, and thus have no explicit
`return` statement. So, for safety and simplicity we will add an extra
`get_ret()` at the end of every function, even if there has been an explicit
return before.

_Our compiler never claimed to be optimal or fast or safe, so double-return is
fine for us_

maths
-----

Now let's compile some arithmetic expressions. They are all similar, so I'll
show how it's done with an example of addition. Remember how parser works? It
parses (and theoretically, compiles) the left part of the expression, then the
right part, and then performs an operation.

This is how a typical math expression is compiled (remember a joke about an
elephant, a giraffe and a fridge?):

	..evaluate left part
	push A
	..evaluate right path
	pop B
	A:=A+B

After we compiler the left part we need to store the results (register A)
somewhere. Stack is the most simple way to do it. So, an expression 
`1+2+3` will be compiled to:

	A:=0001  -+     -+
	push A    |      |
	A:=0002   | 1+2  |
	pop B     |      |
	A:=A+B   -+      | +3
	push A           |
	A:=0003          |
	pop B            |
	A:=A+B       ----+

other stuff
-----------

Work with symbols is easy, too.

To call a function we put its address to register A,
then to a `gen_call()` which emits `call A`.

Accessing local symbols is done using `gen_stack_addr` which
return the address of the given item on the stack.

Accessing global symbols is done using `gen_sym_addr()`.
Also, when a new symbols is created the compiler might need to 
generate some code (e.g. when generation assembler code).
`gen_sym` is used for such cases.

`gen_pop` drops N items from the stack (increases stack pointer).

`gen_unref` is used to unreference pointers. Depending on the type (byte or int)
it generates `A:=m[A]` or `A:=M[A]` code.

`gen_array` puts the constant array on the stack. Or maybe if there is a
separate segment for data it should store the array there.

Finally, `gen_patch()` is used to patch jump address when compiling if/while
statement. The reason is that when we meet such statement we must generate a 
jump instruction, but the address is not known yet - it depends on how many code
is compiled for the body statetment. So, after the body is compiled
we patch jump address with the correct value.

We are done. Let's try the following program:

	int main() {
		int k;
		k = 3;
		if (k == 0) {
			return 1;
		}
		return 0;
	}

	jmp0008 # by gen_start(): jump to main(), which is the next instruction at 0x8
	push A  # add space for local variable "k"
	sp@0000 # get the address of the local variable #0 ("k")
	push A  # store it
	A:=0003 # put 3 to A
	pop B   # get the stored earlier address of "k"
	M[B]:=A # put the value of A to "k" as int
	sp@0000 # get the address of "k"
	A:=M[A] # get its value as int
	push A  # store it
	A:=0000 # put 0 to A
	pop B   # get the value of "k" stored earlier
	A:=B==A # compare A and B ("k" and zero)
	jmz0090 # if false (A!=B, k!=0) - jump to 0x90
	A:=0001 # store 1 to A as return value
	pop0001 # free space allocated for "k" on the stack
	ret     # and return
	jmp0090 # "else" branch should be here, instead jump to 0x90 (next instruction)
	A:=0000 # store 0 to A as return value
	pop0001 # free space allocated for "k" on the stack
	ret     # and return
	ret     # remember about double-return for safety? ;)

Yeah, the code is so dirty and bloated. But it works. And which is more
important, you know now how compilers work and how create your own one.

But I should warn you...

warning
-------

Never, please, never do it this way! If you want to write your own compiler -
use real grown-up tools:
	
* flex/lex/jlex/...
* yacc/bison/cup...
* ANTLR
* Ragel
* and many others

Also, it's worth checking real literature, like [Dragon Book][1]. Maybe the
courses from [coursera.org][2] will be useful for you, too.

If you need to port existing languages for your own architecture - 
you'd better learn how to write LLVM backends or GCC backends.

If you want to read more about toy compilers - check [SmallC][3].

If you want to write compiler for a simple language - check PL/0 or Basic or C.

But please, never write compilers manually for real tasks.

final word
----------

The full sources of the compiler are available on [my bitbucket page][4].  They
are licensed under MIT, feel free to use and modify.  I'm not sure if there is
any sense in submitting issues to this project, so do it only if you know how
to fix them :)

Anyway, compilers is fun. I hope you liked it!

_Check [part 1](/blog/cucu-part1.html) or [part 2](/blog/cucu-part2.html) if
you missed something_



[1]: http://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools
[2]: https://www.coursera.org/course/compilers
[3]: http://en.wikipedia.org/wiki/Small-C
[4]: https://bitbucket.org/zserge/cucu
