One of the goals of EOlang was clear and readable code. 
That's why it has some codestyle checks processed by the compiler.

Firstly, level of nesting is controlled by the indentations.
Reason – code with different indents is hard to read and understand,
and code with correct indents is clear without parentheses. The
size of indent is strict – two spaces. It was done for big decoration 
hierarchy. Also, EOlang strictly controls all space symbols – no more
than one whitespace between words, no more than one blank line between 
objects and attributes, no blank lines in code of one attribute,
no tabs. If a code looks ugly without extra blank lines, it is a good 
marker that you should create a new object or attribute.