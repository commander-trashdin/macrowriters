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

*(Just don't think you're discovering new ones on a weekly basis because you can't be bothered to write an extra `if`.)*

### 1.3 DSL for Syntax Transformations

Different syntaxes for representing common data are inevitable. For instance, JSON, HTML, and others. You might want to write it down yourself and then get an object to work with.

Two options:

1. **Reader Macros:** Create a data structure at the read stage. You receive a stream of characters, which you can use however you want. This allows you to create a new "literal" syntax for anything. Handy, but it always creates a literal, i.e., an immutable value -- keep that in mind.
   
2. **Regular Functions:** Use a regular function with a string containing the data. If you need full compile-time evaluation, you can assign it to an `eval-when (:compile)` variable. If you want that inside a function, you will have to use a thing from 1.4.

### 1.4 A Special Case

This section is dedicated to a specific type of macro, which we'll discuss below.

## 2. When You Probably Shouldn't, But Might Have To

### 2.1 Managing Resources

This one is rather famous, usually referred to as "`with-` style macros". 
The problem of managing resources is a well-known one. You want to open a file, do your thing, close the file. You also want to close the file if something goes wrong, no matter what. Thine file shall be closed, lest vile little computer goblins invade its sacred inner temple and pilfer thy precious data.
Some languages (you know which ones) integrated that in their memory management model. They have system that (wait, you actually don't know which languages I'm talking about??? Cmon..) perform cleanups at certain fixed points, and this is also where the streams are closed, last goodbyes are said. Usually that happens (okay, fine, the languages are Rust and C++, obviously duh) after some scope is closed.

However, these mechanisms are absent in languages with non-deterministic memory management models. While there are ways to hide them under the hood, such as in Haskell, it is rarely used (in Haskell it only works because of how rigid the control flow there is, so in classical Haskell the compiler always knows what and when to do best. They still need explicit functions for opening/closing files if they want to use Control.Monad or such).

Two well known explicit ways are `defer` and `with-`. The former makes you specify _what_ should happen, and the latter makes you specify what we are _dealing_ with. You cannot realisitcally write a `defer` macro in CL (or Scheme, or even Racket) without rewriting _a lot_ of syntax, so we'll stick with `with-`. 

In a perfect world, you would have access to deterministic mechanisms, that would allow you to say "this thing here only exists inside the scope, after which it should be cleaned immediately, here's a custom cleanup mechanism". Or maybe something else, I'm not smart enought to predict the future.
In real world, you just write a `with-` macro, that (usually) expands into `unwind-protect`.

### 2.2 Modification Macros

This is a macro system that creates a workaround for CL all-is-reference copy semantics. I am not a fan of that semantics (which is why this is in 2, not 1), but that system is there for a reason (some things are impossible otherwise, and they should be possible), so use it. Keep in mind that in many codebases as well as in the standard `setf` doesn't do any value transformations on it's own and simple calls the setter (which then can do things if you _really_ want that). That is probably a good status-quo state of things and you don't want to break it. 

A specific case is list modification macros. You can't create a function analogous to `push` and `pop` for plain CL lists. Therefore, you're forced to use macros, and if you need a custom one, you'll have to write it too. The design of those lists is really horrible, but also it is there, so this is unavoidable. This also applies to section 3.4.

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

This version is _slightly_ longer, but also hides no details about what is going on and can be extended in a different file easily. Any optimisation concerns should go to the compiler devs (but also this is rather trivially optimisable).


## 3. When You Really Shouldn't Write Macros

Here's a whole bunch of stuff.

### 3.1 Looping

No. Just no.

…Okay, fine, here's why:

I've seen far too many looping macros (we're talking double digits here and I never even looked for them specfically). The CL standard includes at least four, some doing the same thing as others but worse. That's just insane.

Here's the deal: there's really only one loop — the infinite loop with custom breaks, or the `tagbody` form with tags. Everything else is a variation of that. So what's the problem? Once you write a looping macro, it becomes almost impossible for the compiler, other people, or

future you to figure out what's going on. Sometimes your loop is simple, but other times it is more complex (and even way more complex), and might have a loop inside, might have a couple...Does it return? In CL it sure does. What does it return? Only Machine God knows.

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

Why don't I just make a macro that is basically `(bar a b c)`, and insert it, that would be easier"

If I need to explain why this is a bad idea, you probably shouldn't be writing code at all.

### 3.3 Optimizations

Sometimes, you might want to perform pre-calculations at compile time or create a function that's an exact copy of a standard one but with more type info. This is when people write things like:

```lisp
(defmacro fix+ (a b)
  `(the fixnum (+ (the fixnum ,a) (the fixnum ,b))))
```

This is ... not great. You're changing the semantics from a function to a macro, making it hard to map/reduce that thing. Error signaling is also out of the question since the macro disappears at runtime. There are other problems, like the lack of strict evaluation rules, which means you have to implement them yourself using `once-only` and gensyms.

Common Lisp has a mechanism for this: `compiler-macros`. This is what you should use in this scenario. This is what section 1.4 is for. 

To be fair, they still force you to do manual evaluation properly, but at least you retain the function semantics.

### 3.4 Local Macros

With the exception of 2.2, don't write local macros. Creating a local "loopsy" is about as close to an eldritch horror as you can get. Shortcut macros have already been discussed in 3.2. There's no reason to write a local macro instead of a function, except if it's a dirty macro (which captures variables from the future environment). But dirty macros make the code completely unreadable—way over the line. Optimization concerns aren't valid here either; it's the compiler's job to inline functions and constant-fold things.

Would it be better to have `define-compiler-macrolet`? Maybe. But thankfully, we don't have it, so there's no need to discuss it.
