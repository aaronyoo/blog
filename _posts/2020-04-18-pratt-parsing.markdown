---
layout: post
title: Pratt Parsing
date: 2020-04-18 15:46
comments: false
external-url:
categories: Software
---

While working on a Rust implementation of [Writing and Interpreter in Go](https://interpreterbook.com/) by Thorsten Ball, I learned about an elegant type of recursive parsing called Top Down Operator Precedence Parsing, otherwise known as Pratt Parsing. I had never written a programming language parser before and I thought the code elegance of Pratt Parsing nice enough to warrant this writing. As wil most of my written work, consider this a further exploration of Pratt Parsing in the context of both its history and current use.

## Background

The job of any parser is to turn a stream of tokens into an abstract representation. Said representation is usually a tree. For example, a parsing of `5 + 2` might  parse to the following tree: 

<center><div class="mermaid">
graph TD
  A(+) --> B(5)
  A --> C(2)
</div></center>

And parsing `5 + 2 * 3` might parse to:

<center><div class="mermaid">
graph TD

  A(+) --> B(5)
  A --> C(*)
  C --> D(2)
  C --> E(3)

</div></center>

Notice how operator associatively is actually disambiguated by the tree. The above tree indicates that the expression is of the form `5 + (2 * 3)` which is correct since multiplication is first in the canocial order of operations. Writing parsers to generate these trees is a widely studied problem. There are two main approaches:

1. Formalize your parser into a set of rules called a *grammar* and feed the grammar into a parser generator which will generate your parser for you.
2. Handwrite your parser. One example of such a handwritten parser is the recursive descent parser, of which the Pratt parser is a descendant.

In an industrial "paid by the job" setting, the first option is typical preferred since it is more bug free and more resilient to change. That is, it is simpler to change your grammar and rerun the parser generator than to edit the source code of your handwritten parser. Despite the advantages of using a parser generator, it is still interesting to see how a handwritten parser works. And thus, with practical considerations aside, we continue.

## Pratt Parsing

Pratt Parsing allows the efficient parsing of expressions in which operator precedence matters. In its simplest form it relies on three main concepts: 

1. binding power
2. the distinguishment between infix and postfix operators
3. recursive structure inspired by its close relative, the recursive descent parser

### Binding Power

Pratt parsing relies on the concept of **binding power**, which one can think of intuitively as the propensity for an operator to bind to its operands. This implies that binding power is a *function on operators*. There are also approaches to Pratt parsing which involve a left and right binding power; but for simplicity we will ignore these other variants and stick with a single binding power per operator type that describes how much "pull" the operator has on its operands. The way to assign these binding powers relates to the precedence of the operators. Recall the simple expression:

```
5 + 2 * 3
```

Which we want to parse as:

```
5 + (2 * 3)
```

When processing the input, once we get to the `2` we will have to make a decision. We can either associate the `2` with the `+`, turning the expression incorrectly into `(5 + 2) * 3` or we can associate the `2` with the `*`, correctly parsing the expression as `5 + (2 * 3)`. To make sure that the second equation is the one that is parsed we can assign a binding power to `*` (multiplication) that is higher than the binding power of `+` (addition). As Eli Bendersky writes in his article on Pratt parsing, "what's important is that `bp(*) > bp(+)`". Here, he uses `bp` as the binding power function. By assigning a higher binding power to `*`, the multiplcation operation is able to pull the `2` away from the `+` operator because it has the higher binding power. In general, binding power is used to handle precedence, which is a large part of expression parsing.

### Infix vs. Prefix

Another important feature of Pratt parsing is the way it distinguishes between infix and prefix operators. Pratt parsing can handle other variants of "n-fix" operators such as the "mixfix" ternary operator: `a ? b : c`; but we will once again ignore these more advanced variants in the name of simplicity. The reason why it is important to distinguish between infix and prefix operators is because some operators have different binding powers when they are used as infix versus postfix operators. The textbook example of this is the subtraction operator `-`. Used as subtraction, it has a binding power similar to addition, but used as a unary negation operator, it has a binding power much higher than addition. We expect `5 + - 2 + 3` to be parsed as `5 + (-2) + 3` and not `5 + -(2 + 3)`. This means that the unary `-` operator needs to pull at its one and only right operand, in this case a `2`, much more strongly than the `+` also pulls on the `2`. One can begin to see the need for handling the possible infix versus prefix use of the same operator.

### Recursive Structure

Lastly, the Pratt parser relies heavily on the recursive style of handwritten parsers. In order to understand how this plays out during execution, we bring out a more complicated example:

```
5 + 2 * 3 + 4 / 8 - 1
```

<center><div class="mermaid">
graph TD
  D(+) --> E(5)
  A(*) --> B(2)
  A --> C(3)
  D --> A
  F(+) --> D
  F --> G(/)
  G --> H(4)
  G --> I(8)
  J("-") --> F
  J --> K(1)
</div></center>

In the case of `... + expr * ...`, where an expression is stuck between an addition and a multiplication operator, we have already used binding power to determine which operator will pull the expression towards it. However, we fail to determine how big this is `expr` actually is. For example, we could have a situation in which the expression is something like this `... + 5 ** 3 * ...` Here, the exponentiation operator takes precedence over both the addition and multiplication operators, we can no longer look at the area between two operators as a single token but an expression. As the above example displays, it is difficult to reason about binding power when the expression gets large. This is where we can take advantage of the recursive nature of expressions. Here is some pseudocode for the main Pratt Parsing algorithm:

```python
def parse_expression(binding_power):
  # Try to parse any prefix operator.
  let prefix = self.parse_prefix();
  let left_exp = prefix;
  
  # Continue while the binding power of the token is less than "our" binding power
  while binding_power < self.current_binding_power:
    # Try to parse any infix operator.
    let infix = self.parse_infix(left_exp);
    left_exp = e;
  return left_exp;
```

That's it! It is ultimately the responsibility of parse infix and parse prefix to recursively call `parse_expression`; but the magic lies in the `binding_power < self.current_binding_power` line. The parameter `binding_power` represents "our" binding power, which means the binding power of our current `left_exp`. Initially this value will be zero, or the lowest possible value to ensure that we can parse any infix expression initially. Running through a simple example can help:

```
5 + 2 * 3

5 + 2 * 3
^ bp = 0

5 + 2 * 3
  ^ bp = 0 < bp(+)

Recursive call to parse_expression(bp(+)) from parse_infix

5 + 2 * 3
    ^ bp = bp(+)

5 + 2 * 3
      ^ bp = bp(+) < bp(*)

5 + (2 * 3)

```

One can see that in trying to parse the `+` we need to determine what the operands are. Since these are expressions we call `parse_expression` and pass in `bp(+)`. This recursive `parse_expression` will keep collecting terms and calling recursively as long as the expression that it is collecting has a higher binding power than itself. In other words, since we know things that have higher binding powers bind first, we must try to collect all of the terms with higher binding power before we resolve the initial `+` operation.

## Handling Right Associative Operators

There is one last area that I belive is confusing enough to be mentioned here. With the scheme that I have described above, it is easy to create left associative operators, since the terminating clause for the main while loop is `binding_power < self.current_binding_power`. Because a number is not less than itself, it follows that the parser treats the same operator as a higher binding power operator. This means that expressions will be left associative. In order to make, for example `+`, right associative, we need to allow the while loop to proceed even for multiple of the same operator. This can be done elegantly when calling `parse_expression` instead of passing in the binding power of the operator that called you, subtract one from that value before providing it as an argument to `parse_expression`. This will ensure that the while loop can always proceed when seeing the same operator as the caller, creating right associative operations.

## Limitations

Finally, Pratt Parsing falls a bit short in two main categories. First, it works best when used on programming languages that are of a dynamic and functional flavor. Programming languages that are static and imperative make writing a Pratt parser much more difficult. Second, as mentioned before, using the Pratt Parsing algorithm means writing a handwritten parser, which comes with its own collection of disadvantages in terms of maintenance and correctness.

<br>

---

- Pratt, Vaughan. "[Top down operator precedence](https://web.archive.org/web/20151223215421/http://hall.org.ua/halls/wizzard/pdf/Vaughan.Pratt.TDOP.pdf)." *Proceedings of the 1st Annual ACM SIGACT-SIGPLAN Symposium on Principles of Programming Languages* (1973).
- Crockford, D (2007-02-21). ["Top Down Operator Precedence"](http://crockford.com/javascript/tdop/tdop.html).
- https://ycpcs.github.io/cs340-fall2016/lectures/lecture05.html
- https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html
- https://eli.thegreenplace.net/2010/01/02/top-down-operator-precedence-parsing