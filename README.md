# CSCI 3366 Programming Languages

### Spring 2018

**R. Muller**

------

### Lecture Notes: Week 9

#### This Week

1. The miniC Compiler: Storage for Variables; Call Stack Layout; Codengen
2. Untyped Lambda Calculus 1: Syntax & β-reduction
3. Untyped Lambda Calculus 2: Normal-forms; Combinators

------

## 1. The miniC Compiler: Storage for Variables; Call Stack Layout; Codegen

The final phase of the miniC compiler is primarily concerned with storage allocation for variables used in a functions. This includes 1. parameters passed to the functions, 2. local variables explicitly declared in the function and 3. temporary storage locations introduced by the compiler in earlier compilation phases.

You should review the `Quads` and `Mips` modules to ensure that you understand the source and target languages of the translation performed by the `Codegen` module.

> NB: miniC has no heap allocation and the miniC compiler has no register allocator, all storage for variables is allocated on the run-time stack.

When a function is called, its activation record or call frame is constructed on the run-time stack. Part of the activation record is constructed by the calling procedure, the "caller", and part of it is constructed by the function being called, the "callee". The structure is described in several course documents:

```
       +----------------------------+
A      |      Caller-Save Regs      |                 higher addresses
A      +----------------------------+
A      |       Value of Arg 1       |                 
A      +--           :            --+
A      |             :              |
A      +--                        --+
A      |       Value of Arg n       |
A      +----------------------------+
A      |  Caller's Return Address   |
       +----------------------------+
B      |   Caller's Frame Pointer   | <-- fp
B      +----------------------------+
B      |      Local Variable 1      |
B      +--            :           --+
B      |              :             |
B      +--                        --+
B      |      Local Variable k      |
B      +----------------------------+
B      |   Callee Save Registers    |                 lower addresses
       +----------------------------+
                                      <-- sp
```

The portions of the call frame annotated with A on the left are built by the caller and the portions annotated with a B on the left are built by the callee. Note that since there is no register allocator, there are no caller-save or callee-save registers.

#### Accessing Storage

The MIPS architecture uses a special register, the frame pointer `$fp` , which is expected to contain the address of a fixed position in the (middle of) a call frame. Storage for all identifiers are accessed via indirect addressing off of the frame pointer. For example, `-4($fp)` would access the 4-byte word of memory allocated for local variable 1, while `8($fp)` would access storage for argument n.

#### The Codegen Environment

The code generator uses an environment for two purposes:

1. To record implementations of built-in operators, and
2. To record storage offsets for identifiers.

The base code generator environment is defined in the `Dynamicbasis` module. Values in the environment are of type `Dynamicbasis.t`:

```ocaml
type t = CodeGenerator of (Opnd.t list -> Mips.Codestream.t)
       | Offset of int
```

You should review the code in `dynamicbasis.ml` to better understand the implementations of the various primitive operators in miniC. For example, the code generator for the simple `+` operator is

```ocaml
(* + *)
(function
  | [rd; rs; rt] -> fIL [toInstr(None, Opn.Add {rd; rs; rt}, None)]
  | _ -> failwith "Dynamicbasis: bad call of the + codegen.")
```

In the above, `fIL` is an abbreviation of the `Mips.fromInstructionList` function.

The codegen harness code has one pre-defined utility function for looking up code generators and one pre-defined utility function for looking up the storage offset for an identifier. These functions are called `lookupCodeGenerator` and `lookupOffset` respectively.

The harness code has a stub for the function `makeEnv`. The purpose of this function is to extend the pre-built dynamic basis by associating appropriate storage offsets for identifiers appearing in a given `Quads` procedure.

## 2. Untyped Lambda Calculus 1: Syntax & β-reduction

The development of set theory allows us to understand mathematical functions as sets of input/output pairs. 

```
fact = {(0, 1), (1, 1), (2, 2), (3, 6), (4, 24), (5, 120), ... }
```

The [untyped lambda calculus](<https://en.wikipedia.org/wiki/Lambda_calculus>), discovered by [Alonzo Church](<https://en.wikipedia.org/wiki/Alonzo_Church>) in 1932, is a theory of **functions as computation rules**. These are two complementary perspectives on functions. It turns out that lambda calculus is equivalent in computational power to a Turing Machine.

#### Syntax

Let Var = {x, y, z, …} be a set of variables.

```
M, N ::= x | (λx.M) | (M N)
```

The λ-terms are variables, functions (aka *abstractions*) and applications (calls) of functions. That is it.

+ The x in (λx.M), is a binding occurrence of the variable x, M is the body of the function. The variable x may or may not occur in M. In lambda calculus, all functions are anonymous, but they can be given names as in ((λf.M) (λx.N)), here, in M, the function (λx.N) goes by the name f. 
+ In (M N), M is in the function position and N is in the argument position, i.e., the function computed by M will be passed N as an input.

It's convenient to use abbreviations for the notation above.

1. Outer parenthesis will often be omitted. So ( λx.x) and (y z) might be written as λx.x and y z, resp. 

2. Application associates to the left, so

   ``` 
   M N P is shorthand for ((M N) P)
   ```

3. Nested function notation

   ```
   λxy.M is shorthand for (λx.(λy.M))
   ```

#### Combinators

A λ-term is *closed* if it contains no free variables. Closed λ-terms are called *combinators*.

Some combinators are especially well-known:

+ I = λx.x — the identity function
+ K = λxy.x — the function of two arguments (roughly speaking) that returns its first argument
+ S = λxyz.x z (y z) — the S combinator is a basic router.
+ ω = λx.x x — the ω combinator has self-application
+ ω' = λx.f (x x)
+ Ω = ω ω
+ Y = λf.ω' ω' — the paradoxical

##### Lambda Terms as Trees

It's often easiest to understand what is going on in a λ-term by viewing it as a tree, I, S, ω and Y are as follows

``` 
  I = λx          S = λx            ω = λx              Y = λf
       |               |                 |                   |
       x              λy                 @                   @
                       |                / \                /   \
                      λz               x   x             λx     λx
                       |                                  |      |
                       @                                  @      @
                     /   \                               / \    /  \
                    @     @                             f   @  f    @
                   / \   / \                               / \     / \
                  x   z y   z                             x   x   x   x
```

#### β-Reduction

A λ-term of the form (λx.M) N is called a *β-redex*. A function (λx.M) is applied to an argument N by replacing all free occurrences of x in M by N:

(λx.M) N —>β M[x:=N]

In lambda calculus we extend use this essential one-step β-reduction relation to define other relations:

1. reduction in any subterm, e.g., ((λx.z) y) v —> z v
2. multistep reduction —>>β and
3. conversion or equality =β.

We'll omit the details but see Barendregt for the details.

#### Normal Forms

A λ-term containing no β-redexes is called a β-normal-form (β-nf). All of the example combinators above are β-normal forms except Ω and Y. The term K I is not a β-normal form, but it *has* one, namely λy.I. 

>  It turns out that the first undecidable problem in computing was the question, for an arbitrary λ-term M, does M have a β-normal form. There is no decision procedure.

The terms Ω and Y are not normal forms and moreover, they do not have normal forms. In particular, Ω is a canonical infinite loop:

```
Ω -> Ω -> Ω -> ...
```

The discoverers of lambda calculus were troubled by Ω and went to some lengths to banish it.

## 3. Untyped Lambda Calculus 2: The Fixpoint Combinator

In lambda calculus, we can represent pretty much anything, natural numbers, integers, booleans, pairs, etc. It's an interesting topic but we'll leave it for another day, but see [Selinger](<https://github.com/BC-CSCI3366/s19/blob/master/resources/papers/Selinger.pdf>), section 3. 

For now, consider our old friend the factorial function

```
fact = λn.if n = 0 then 1 else n * fact(n - 1)
```

This makes sense intuitively but there is no provision in lambda calculus to use the symbol fact recursively in this manner. In order implement unbounded repetition, we can use the Y combinator. Let F be any λ-term. Then

```
Y F = (λf.(λx.f (x x)) (λx.f (x x))) F
    =β (λx.F (x x)) (λx.F (x x))
    =β F ((λx.F (x x)) (λx.F (x x)))
    =β F (F ((λx.F (x x)) (λx.F (x x)))
    = ...
    =β F (F (F (...)))
```

and in general, Y F = F (Y F). This is to say, that for any λ-term F, (Y F) is a [fixpoint](<https://en.wikipedia.org/wiki/Fixed_point_(mathematics)>) of F.

We can use Y to implement recursion as follows. Returning to the fact example, first we abstract with respect to fact:

```
F = λfact.λn.if n = 0 then 1 else n * fact(n - 1)
```

Then consider Y F 3.

```
Y F 3 = (Y F) 3
      = ((λf.(λx.f (x x)) (λx.f (x x))) F) 3
      =β (λx.F (x x)) (λx.F (x x)) 3
      =β F (((λx.F (x x)) (λx.F (x x))) 3
      = (λfact.λn.if n = 0 then 1 else n * fact(n - 1)) (((λx.F (x x)) (λx.F (x x))) 3
      =β (λn.if n = 0 then 1 else n * (((λx.F (x x)) (λx.F (x x)))(n - 1)) 3
      =β if 3 = 0 then 1 else 3 * (((λx.F (x x)) (λx.F (x x)))(3 - 1)
      = if false then 1 else 3 * (((λx.F (x x)) (λx.F (x x)))(3 - 1)
      = 3 * (((λx.F (x x)) (λx.F (x x)) 2)
      = ...
      = 3 * 2 * 1 * 1
      = 6
```

