# About Macros: Should You Write One?

This guide explores when (and if) you should write a macro in Common Lisp. We'll look at three different scenarios:

1. **Cases where writing a macro is a good idea** – This is what macros are for; go ahead and write one.
2. **Cases where you probably shouldn't, but might have to** – Ideally, you wouldn't, but in reality, it might be necessary. Just keep this in mind.
3. **Cases where you absolutely shouldn't write macros** – Even though you probably will, don't.

## 1. When You Should Write Macros

### 1.1 Creating New (or Updating Existing) Definitions

Before diving in, let's clarify what I mean by "definitions." Consider these two examples:

```lisp
(defclass helicopter () ())
```

```lisp
(defparameter *my-personal-helicopter* nil)
```

These two are very different. Only one of them is a *definition*, while the other is an *assignment*. 

**Definition:** Creates no runtime object. It might create something at compile-time, but you shouldn't worry about that. A definition provides a new description of some structure.

**Assignment:** Creates actual runtime values that you interact with. It doesn't create new descriptions; it only uses existing ones.

Examples of definitions in Common Lisp include `defstruct`, `defclass`, `deftype`, and `defgeneric`. On the other hand, assignments include `defvar`, `defparameter`, `defmethod`, and `let`. Now, you might be wondering: Where does `defun` fit in? (Pun intended!) The answer: It's a mixed bag. `Defun` creates a runtime object (a named function) and a definition (mainly a list of arguments, which might have type signatures).

*(Side note: This is why I'm a fan of Lisp-N, where each definition deserves its own symbol "space." I'm not a fan of dividing functions and variables into two spaces (as in classic Lisp-2) because both are just assignments of an object to a variable.)*

**What Are Macros Good For Here?** Definitions, NOT assignments. You might want to create a new type of description or specify its structure. For example, if only `defstruct` existed, you could create your own (inefficient but not terrible) `defclass` by writing a macro.

Another example: You might want to simplify variable declarations with type specifications:

```lisp
(let ((x 37))
  (declare (type fixnum x))
  (+ x 42))
```

Too verbose? You could create a macro to make this more concise. While at it, why not throw in `multiple-value-bind`? But be cautious—without careful design, you could end up with a mess.

Two things to note:

1. **Avoid Using Macros:** You can often avoid macros by registering internal objects manually. However, this exposes the internals to the user, which could lead to confusion.
2. **Single Definition-Creating Macro:** If your macro registers multiple things, make sure they're all strictly required—users should have to use them.

### 1.2 Control Flow Mechanisms

Examples include pattern matching with a `match` macro or parallelism approaches. If `case` didn't exist, you'd need to write one (although a good `match` could cover that). 

However, this is mostly theoretical, as there aren't many significantly distinct control-flow expressions. Ideally, a language should already include all the known ones. But, since humanity's knowledge grows, a truly self-modifying language **must** have a way to create new control flow expressions, just in case we discover something new.

*(Just don't think you're discovering new ones weekly because you can't be bothered to write an extra `if`.)*

### 1.3 DSL for Syntax Transformations

Different syntaxes for representing common data are inevitable. For instance, JSON, HTML, and others. You might want to write it down yourself and then get an object to work with.

Two options:

1. **Reader Macros:** Create a data structure at the read stage. You receive a stream of characters, which you can use however you want. This allows you to create a new "literal" syntax for anything. Handy, but it always creates a literal, i.e., an immutable value.
   
2. **Regular Functions:** Use a regular function with a string containing the data. If you need full compile-time evaluation, you can assign it to an `eval-when (:compile)` variable.

### 1.4 A Special Case

This section is dedicated to a specific type of macro, which we'll discuss below.

## 2. When You Probably Shouldn't, But Might Have To

### 2.1 Managing Resources

This is the famous "`with-` style macros" situation. The problem is well-known: you want to open a file, do your thing, and close the file—even if something goes wrong.

Some languages (you know which ones) handle this with their memory management model. But in languages with non-deterministic memory management, you have to manage it explicitly. 

Two well-known explicit methods are `defer` and `with-`. Since you can't realistically write a `defer` macro in CL, we'll stick with `with-`. 

In a perfect world, you'd have deterministic mechanisms for cleaning up resources immediately after a scope ends. In the real world, you write a `with-` macro, which typically expands into `unwind-protect`.

### 2.2 Modification Macros

This type of macro works around CL's all-is-reference copy semantics. While I’m not a fan, it's there for a reason. Use it, but remember that `setf` in many codebases (and in the standard) doesn't do any value transformations by default—it simply calls the setter.

A specific case is list modification macros. You can't create a function analogous to `push` and `pop` for plain CL lists. Therefore, you're forced to use macros, and if you need a custom one, you'll have to write it too. The design of those lists is terrible, but unavoidable.

This also applies to section 3.4.

Another similar case is the "generalized reference" concept, sometimes implemented via `symbol-macrolet`. Two things to consider:

1. **Semantics:** This is the only way to achieve something similar to actual reference, at least within the scope.
2. **Value:** Things can get murky when you generalize too much. I recommend explicit transformations for clarity (a known non-symbol-macro case is `setf subseq`—there's a function for that).

### 2.3 Generating Boilerplate

This often appears as a top-level `macrolet` with multiple calls to the defined local macros.

Example from the SBCL codebase:

```lisp
(macrolet ((def (op doc)
             `(defun ,op (number &rest more-numbers)
                ,doc
                (declare (explicit-check))
                (let ((n1 number))
                  (declare (real n1))
                  (do-rest-arg ((n2 i) more-numbers 0 t)
                    (if (,op n1 n2)
                        (setf n1 n2)
                        (return (do-rest-arg ((n) more-numbers (1+ i))
                                  (the real n))))))))) ; for effect
  (def <  "Return T if its arguments are in strictly increasing order, NIL otherwise.")
  (def >  "Return T if its arguments are in strictly decreasing order, NIL otherwise.")
  (def <= "Return T if arguments are in strictly non-decreasing order, NIL otherwise.")
  (def >= "Return T if arguments are in strictly non-increasing order, NIL otherwise."))
```

Now, a rewrite that avoids code generation:

```lisp
(defun op (op number &rest more-numbers)
  (declare (explicit-check))
  (let ((n1 number))
    (declare (real n1))
    (do-rest-arg ((n2 i) more-numbers 0 t)
      (if (funcall op n1 n2)
          (setf n1 n2)
          (return (do-rest-arg ((n) more-numbers (1+ i))
                    (the real n)))))))

(defun < (number &rest more-numbers)
 "Return T if its arguments are in strictly increasing order, NIL otherwise."
 (apply op #'< number more-numbers))

(defun > (number &rest more-numbers)
 "Return T if its arguments are in strictly decreasing order, NIL otherwise."
 (apply op #'> number more-numbers))

(defun <= (number &rest more-numbers)
 "Return T if arguments are in strictly non-decreasing order, NIL otherwise."
 (apply op #'<= number more-numbers))

(defun >= (number &rest more-numbers)
 "Return T if arguments are in strictly non-increasing order, NIL otherwise."
 (apply op #'>= number more-numbers))
```

This version is slightly longer but much clearer and easier to extend. Any optimization concerns should be addressed to the compiler developers (this is pretty easy to optimize).

## 3. When You Really Shouldn't Write Macros

Here's a whole bunch of stuff.

### 3.1 Looping

No. Just no.

…Okay, fine, here's why:

I've seen far too many looping macros (we're talking double digits here). The CL standard includes at least four, some doing the same thing as others but worse. That's just insane.

Here's the deal: there's really only one loop—the infinite loop with custom breaks, or the `tagbody` form with tags. Everything else is a variation of that. So what's the problem? Once you write a looping macro, it becomes almost impossible for the compiler, other people, or

 future you to figure out what's going on. Sometimes your loop is simple, but other times, it's a mess of nested loops, returns, and who knows what else.

There's a much safer and more readable alternative—the iterator protocol. A great example is Rust's standard iterators. They provide a uniform polymorphic interface for anything you might want to iterate over. If you have something new, you can extend it and have your own iterator that works well with existing ones.

**Benefits:**

1. **No New Syntax:** No need to design your own or understand someone else's.
2. **Easy Type Inference:** (Not so important for CL, but handy occasionally) You can easily tell when it stops and what it returns.
3. **Simpler to Combine:** You can easily combine things to change behavior without tracking internal values.
4. **Extensible and Polymorphic:** CL macros can't be polymorphic in a dynamic language.

### 3.2 Convenience "Shortcut" Macros

"Oh, I keep writing a lot of this:

```lisp
(defun foo (a b c)
  (bar a b c)
  ...)
```

Why not make a macro to simplify it?"

If I need to explain why this is a bad idea, you probably shouldn't be writing code at all.

### 3.3 Optimizations

Sometimes, you might want to perform pre-calculations at compile time or create a function that's an exact copy of a standard one but with more type info. This is when people write things like:

```lisp
(defmacro fix+ (a b)
  `(the fixnum (+ (the fixnum ,a) (the fixnum ,b))))
```

This is ... not great. You're changing the semantics from a function to a macro, making it hard to map/reduce that thing. Error signaling is also out of the question since the macro disappears at runtime. There are other problems, like the lack of strict evaluation rules, which means you have to implement them yourself using `once-only` and gensyms.

Common Lisp has a mechanism for this: `compiler-macros`. This is what you should use in this scenario. You can think of this as section 1.4. 

While compiler-macros still require manual evaluation, they at least retain function semantics.

### 3.4 Local Macros

With the exception of 2.2, don't write local macros. Creating a local "loopsy" is about as close to an eldritch horror as you can get. Shortcut macros have already been discussed in 3.2. There's no reason to write a local macro instead of a function, except if it's a dirty macro (which captures variables from the future environment). But dirty macros make the code completely unreadable—way over the line. Optimization concerns aren't valid here either; it's the compiler's job to inline functions and constant-fold things.

Would it be better to have `define-compiler-macrolet`? Maybe. But thankfully, we don't have it, so there's no need to discuss it.
