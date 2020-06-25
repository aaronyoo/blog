---
layout: post
title: CodeGen With Mem2Reg
date: 2020-06-23
comments: false
external-url:
categories: Software
---

Implementing code generation for my [C-like compiler](https://github.com/aaronyoo/minicc) was my first plunge into using LLVM as a backend code generator. To build a compiler, all I had to do was to use the API to generate valid LLVM IR. Even though LLVM is an SSA based infrastructure, it doesn't require that you generate IR in an traditional SSA form, which is nice. The reason this is particularly beneficial is that you can use memory based constructs when generating code; this allows the code generation pass to be very simple and absent of any of the details of SSA form. Strictly speaking, the code that is given to LLVM is SSA form; but I like to think of it as "SSA with memory" with the idea that memory automatically violates the referential transparency of SSA. To illustrate how convenient LLVM IR (SSA with memory) is, consider the following example:

```C
int factorial(int n) {
  int ret;
  if (n == 0) {
    ret = 1;
  } else {
    ret = n * factorial(n - 1);
  }
  return ret;
}
```

This is a simple factorial program, which my compiler currently generates as:

```
define i64 @factorial(i64 %0) {
entry:
  %n = alloca i64
  store i64 %0, i64* %n
  %ret = alloca i64
  %1 = load i64, i64* %n
  %2 = icmp eq i64 %1, 0
  %ifcond = icmp eq i1 %2, true
  br i1 %ifcond, label %then, label %else

then:                                             ; preds = %entry
  %3 = load i64, i64* %ret
  store i64 1, i64* %ret
  br label %end

else:                                             ; preds = %entry
  %4 = load i64, i64* %ret
  %5 = load i64, i64* %n
  %6 = load i64, i64* %n
  %7 = sub i64 %6, 1
  %8 = call i64 @factorial(i64 %7)
  %9 = mul i64 %5, %8
  store i64 %9, i64* %ret
  br label %end

end:                                              ; preds = %else, %then
  %10 = load i64, i64* %ret
  ret i64 %10
}
```

This code is very inefficient but easily generated. Perhaps the penultimate example of this is the way I currently handle identifiers. In its current form, every identifier is put on the stack so that it can be stored to and loaded from later. This leads to two interesting code generation patterns. First, arguments are "spilled" to the stack in the function prologue. The factorial begins with the moving of the immutable `%0` argument into a mutable named stack reference `n`. This is done so that later parts of the code generation can find `n` by name. It also can help when introducing operations that require lvalues such as the address-of operator. Second, expressions that contain multiple references to the same expression are loaded multiple times. This can be seen in the `else` label of the generated code as `n` is loaded twice. Put simply, every identifier assumed to be an lvalue by the code generator.

LLVM's optimizing backend allows us to generate code without worrying about SSA details at all. In fact, my code generator puts all variables in memory, an almost direct violation of SSA. To showcase how LLVM can turn the code above into more efficient IR, we can run the `mem2reg` optimizing pass on it:

```
define i64 @factorial(i64 %0) {
entry:
  %1 = icmp eq i64 %0, 0
  %ifcond = icmp eq i1 %1, true
  br i1 %ifcond, label %then, label %else

then:                                             ; preds = %entry
  br label %end

else:                                             ; preds = %entry
  %2 = sub i64 %0, 1
  %3 = call i64 @factorial(i64 %2)
  %4 = mul i64 %0, %3
  br label %end

end:                                              ; preds = %else, %then
  %ret.0 = phi i64 [ 1, %then ], [ %4, %else ]
  ret i64 %ret.0
```

We can see that LLVM has optimized away all of the `alloca` instructions because none of them were true lvalues. A phi node at the beginning of the `end` label has also been introduced to account for the two different values that can flow into the return value. In general, LLVM's Mem2Reg optimization operates by looking at each variable on the stack (all of them in our case) and determines if the address escapes. If not, it turns them into LLVM variables and uses a typical dominance frontier based SSA construction algorithm to produce valid SSA form. Mem2Reg needs to know if the an address "escapes" because if so it cannot perform an optimization as the value is required to be on the stack.

While this type of separation of concerns between parts of a compiler pipeline isn't novel, I felt that this specific example of generating naive code and having LLVM optimize it was compelling enough to write about. Even the [official tutorial](https://llvm.org/docs/tutorial/) for creating a language with LLVM uses phi node construction and other advanced SSA topics. I would argue that spilling all variables to the stack naively during code generation makes the generation routine a lot simpler at no additional runtime cost.

<br>

---

- https://llvm.org/docs/tutorial/
- https://llvm.org/docs/Passes.html#mem2reg-promote-memory-to-register
