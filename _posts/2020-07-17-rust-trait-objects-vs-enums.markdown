---
layout: post
title: Rust Trait Objects vs Enums
date: 2020-07-17
comments: false
external-url:
categories: Software
---

My brain is wired to think OOP most times, which causes some trouble when trying to play with Rust and its "structs + traits" system. One project that I am currently working on requires me to design an Intermediate Language (IR) that encapsulates all the semantics of ANSI C. One step in this process is creating my IR instructions in a "general" way. Normally, in OOP, I would create a base class that is an IR instruction and has all of the default things that an IR instruction would have and then create subclasses that inherit from said base class. In C++ it might look something like this:

    // Abstract instruction class
    class Inst {
       virtual string get_string_repr() = 0;
       // ....
    }

    // Concrete instruction
    class InstReturn : Inst {
        override string get_string_repr();
        // ...
    }

All my other instructions would follow in place. To make my functions have runtime polymorphism I would pass around an `Inst*` ptr and use a `dynamic cast` to figure out which type of derived instruction it is. One thing to note is that we really want the ability to pass a more general type (`Inst*` in this case) and then get a more specific type back out of that (using `dynamic cast` in this case) when it is time to operate on it. This would allow certain passes to operate on all types of instructions in a general way.

To achieve that same functionality in Rust I tried to use a trait object. This turned out to be the wrong choice but I find it still useful to review and discuss why its the wrong choice. At first it seemed tantalizing because you can do something like this:

    pub trait Inst {
        fn get_string_repr(&self) -> String;
    }

    pub struct InstReturn {...}

    impl Inst for InstReturn {
        fn get_string_repr(&self) -> String { "InstReturn".to_string() }
    }

Traits in Rust function kind of like interfaces in other languages. I can have all my instructions implement a certain trait then have my passes take a trait object (sometimes denoted `Inst` or `dyn Inst`). The problem with taking a trait object is that a trait object is opaque. It doesn't allow downcasting or the inspection of the underlying value (setting certain crates or `Any` aside for the moment). The reason for this is that traits have an open world assumption; that is, anyone can add structs that implement a certain trait, meaning that it would be difficult to ensure that each possible case of decomposition through matching is satisfied. That is why it is recommended in this situation to use enums instead, in part because they have a closed world assumption. An implementation with enums would be:

    enum Inst {
        InstReturn(InstReturn),
    }

    impl Inst {
        fn get_string_repr(&self) -> String {
    	match self {
    	    InstReturn(_) => "InstReturn".to_string(),
    	}
        }
    }

    pub struct InstReturn {...}

One can see here that at first the structure of the code is a little bit confusing as it separates the `get_string_repr` implementation from InstReturn and makes it the responsibility of the Inst enum. This type of separation can be jarring at first, but it is important to remember that an enum is a closed world assumption so the compiler will actually tell us when we missed a case.
