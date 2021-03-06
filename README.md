befunge
=======

A Befunge-93 and doublefunge interpreter written in haskell by Joe Jevnik,
licensed under the GPL v2.

Befunge-93
----------

### What is Befunge-93? ###

Befunge-93 is a stack based esoteric programming language. What makes Befunge
interesting when compared to other simmilar languages is that is not read
linearly. In Befunge-93, the program's state includes a pointer into a
"playfield", and different operations can change the direction that the pointer
is going. Befunge-93 is also self-modifying and referencing. Source code must
fit into one standard screen, 80x25, for it to be read, anything outside of that
area will be ignored. An space not filled in will be read as a space.
Programmers may then query the playfield at a given location, or place a value
from the stack into the playfield at a given location.

### The Stack ###
Befunge-93 is stack based. This means that the only way to store a value is to
push it onto the stack, implying that there is only one register. Many
operations act on the stack by popping values and pushing the result of some
computation. Befunge-93 uses reverse polish notation, meaning:

    1 2 +

translates to push 1 push 2, pop 2 pop 1 and push their sum (3). Therefore:

    1 2 + 5 *

translates to push 1 push 2 pop 2 pop 1 push their sum (3) then push 5, pop 5,
pop 3 and push their product (15).

The stack is represented internally as a list of Int32's. When putting a value
back into the playfield or printing as a char, these values are taken mod 256
and converted to their ASCII character.

Some operations that act on the stack are:

- `0-9` Any integer literal will push itself onto the stack, there is no way
  to push a literal greater than 9 without using an arithmetic operator.
- `+` pop a pop b push (b + a)
- `-` pop a pop b push (b - a)
- `*` pop a pop b push (ba)
- `/` pop a pop b push (b / a)
- `%` pop a pop b push (b mod a)
- `` ` `` pop a pop b push (if b > a then 1 else 0)

### The Playfield ###
Befunge-93 source code must fit within the bounds of a single screen, 80
characters wide by 25 characters tall. The playfield is indexed (row,column)
with (0,0) being the top left corner, and (24,79) being the the bottom right
corner. The playfield may contain only ASCII characters. The playfield can be
accessed and modified at runtime making befunge very unique. For example, if you
put a character into the playfield and then the pointer moves over it, it will
be interpreted as Befunge-93 source code.

The playfield is interfaced with:

- `g` pop r pop c push (The ASCII value stored at (r,c))
- `p` pop r pop c pop v put v mod 256 into the playfield at (r,c)


If you try to read from a location that does not exist inside the playfield, IE
(-1,95), you will always push a zero (0) onto the stack. If you try to write off
the playfield, the command is ignored. No errors are raised.


### The Pointer ###
Befunge-93 is not read linearly. Befunge-93 programs continue to run until the
terminating operator `@` is read. The command `><^v` affect the direction that
the pointer is traveling. All programs start with the pointer at (0,0) moving
right until a command tells changes that.

For example, imagine the program:

    >  v
	^  <

The pointer travels to the right until it hits `v`. At this point the program
will begin to read downwards, where the next character is `<`. This command will
tell the pointer to move left until `^` is hit, where it will cycle back into
the `>`. This is an example of an infinite loop in Befunge-93. Another way to
know this program is infinite is that it does not contain an `@` so it has no
way to end.

The most simple infinite loop is an empty playfield. This is because the
playfield is a torus. This means that if the pointer is traveling right,
and it encounters the right edge, it will wrap around to the left side. This
property is true reading in any direction.


### Flow of Control ###
Befunge-93 uses pointer direction to influence the flow of control. For this
reason, there are 2 types of boolean checks, the horizontal if `_`, and the
vertical if `|`. The if statements work by popping the top element of the stack
and inspecting it. In the case of the horizontal if, if the value is 0, change
direction to read to the right, any other value will change it to the left.
With the vertical if, if it pops a 0, it will change to read down, any other
value will change to read up.

For example:

    v  @   _  52*"!ereh era eW",,,,,,,,,,,,,@

    >  1   |
           @

    This is a comment.

This program starts by reading down, then to the right. It pushes a 1 to test
the vertical if. Because it is non-zero, it moves up. Then, it hits the
horizontal if where it pops from a now empty stack. As discussed before, this
will pop a zero, which makes the if go right. Now, to print the string, we need
to push it it in reverse. This example also introduces the String concept. When
a `"` is read, it will push all the ASCII value of all characters it reads to
the stack until a closing `"` is read. In this example, we push the string
"!ereh era eW" so that when we print it using the `,`, we will get it in the
proper order. We also pushed a `52*` = 10 onto the stack first to get the
newline character. Finally, we see that comments are merely characters that will
never have the pointer run over them.


### IO ###
There is no file IO in Befunge-93; however we have a couple of ways of
interacting with stdin and stdout. We can read input from the user in 2 ways,
with:

- `&` which reads the next integer. IE inputting 55 pushes 55 onto the stack.
- `~` which reads the next character and pushes its ASCII value onto the stack.

In my implementation `&` will error with an InvalidInputError if the input is
not an integer. If the value overflows, it will be taken mod 4294967295 to fit
into an Int32.

The functions to interact with stdout are:

- `.` which pops the top value from the stack and prints it as an integer.
- `,` which pops the top value and converts it to a character by taking the
  value mod 256.


### Command Summary ###

    COMMAND         INITIAL STACK (bot->top)RESULT (STACK)
    -------         -------------           -----------------
    + (add)         <value1> <value2>       <value1 + value2>
    - (subtract)    <value1> <value2>       <value1 - value2>
    * (multiply)    <value1> <value2>       <value1 * value2>
    / (divide)      <value1> <value2>       <value1 / value2> (nb. integer)
    % (modulo)      <value1> <value2>       <value1 mod value2>
    ! (not)         <value>                 <0 if value non-zero, 1 otherwise>
    ` (greater)     <value1> <value2>       <1 if value1 > value2, 0 otherwise>
    > (right)                               PC -> right
    < (left)                                PC -> left
    ^ (up)                                  PC -> up
    v (down)                                PC -> down
    ? (random)                              PC -> right? left? up? down? ???
    _ (horizontal if) <boolean value>       PC->left if <value>, else PC->right
    | (vertical if)   <boolean value>       PC->up if <value>, else PC->down
    " (stringmode)                          Toggles 'stringmode'
    : (dup)         <value>                 <value> <value>
    \ (swap)        <value1> <value2>       <value2> <value1>
    $ (pop)         <value>                 pops <value> but does nothing
    . (pop)         <value>                 outputs <value> as integer
    , (pop)         <value>                 outputs <value> as ASCII
    # (bridge)                              'jumps' PC one farther; skips
                                            over next command
    g (get)         <x> <y>                 <value at (x,y)>
    p (put)         <value> <x> <y>         puts <value> at (x,y)
    & (input value)                         <value user entered>
    ~ (input character)                     <character user entered>
    @ (end)                                 ends program

### Sources ###

[Esolangs Wiki][], [Catseye language spec][]

[Esolangs Wiki]:http://esolangs.org/wiki/Befunge
[Catseye language spec]: https://github.com/catseye/Befunge-93/blob/master/doc/Befunge-93.markdown

Doublefunge:
------------

### What is Doublefunge? ###

While implementing this interpreter, I had an idea for an extension that I now
call doublefunge (please contact me if this name means something different or
the concept goes by another name). The main difference with doublefunge is that
you now have 2 pointers into the playfield. This is not the same as running the
same befunge-93 program twice because the secondary pointer starts at (24,79)
moving left, and they both share the same playfield and stack. Doublefunge
programs should use the .df file extension to show that they are not intended
for a befunge-93 interpreter.

### How the Secondary Pointer Interacts with the Playfield ###

When running a doublefunge program, the primary pointer always has precedence
over the secondary. For example, at the start of a program, the first thing done
will be to read and evaluate the command at (0,0), the location of the primary
pointer, then read and evaluate the command at (24,79), the location of the
secondary pointer. Also, if the value read by the primary pointer is `9`, and
the value read by the secondary pointer is `5`, because of the precedence, the
stack will first push `9` and then push `5`.

### String Mode ###

Although the primary and secondary pointers share the stack, they each have
their own string mode. This means that when the primary pointer encounters
a `"` command, only the primary pointer will begin to read the ASCII values
into the stack, unless of course the secondary pointer is already in string mode
itself. The primary pointer will then stay in string mode until it reads a `"`.
This means that the secondary pointer may modify the contents of the string
while it is being read.

### Halting ###
A doublefunge is the same as befunge-93 with respect to terminating. As soon as
the terminate operator `@` is read, the program will exit. It does not matter if
this command is read by the primary or seconday pointer. Also, if the primary
pointer reads the terminate command, the value at the second pointer will never
be evaluated.

### Examples ###

In this example, we see that the two pointers share the stack, and the order in
which things are evaluated. We also see that the output would be very different
if run through a befunge-93 interpreter.

    >5+:. 52*"pot"52*,,,,,                                                         @























                ,,,,,,,"bottom"*52                                                5<

When run through the traditional befunge-93 interpreter with
`runbefunge example.df` we get the following output:

    5
    top

However, when run through the proper interpreter with the `-d` flag, we get:

    10
    top
    bottom

This is because without the second pointer, we never push the 5 to add to the
other 5, and we never push and print the "bottom" string.

Another example is a flip flop program. This shows how the two pointers can both
modify the playfield for eachother:

    52*"pilf",,,,,






















    >                                                                    99*64*+50pv
    ^p05+*99*65                                                                    <

This program when run with `runbefunge string.df` will infinitly print `flip` on
new lines. When run with `runbefunge -d string.df`, the secondary pointer will
change the 'i' to an 'o' and will print alternating `flip` and `flop` on new
lines.
