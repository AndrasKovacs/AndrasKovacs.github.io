---
title: 'Lightweight resilient recursive parsing'
date: 2026-09-29
permalink: /posts/2026/09/blog-post-2/
published: true
tags:
---

Resilient parsing means that we try to parse as much as possible of the source code, possibly
producing multiple parsing errors. Clearly, this is useful in numerous IDE features like
auto-completion, and can be a nice improvement on interactive user experience.

In this post I explain a library design which is, as far as I know, new. I plan to use this in
[Pterodactyl](https://www.jonmsterling.com/019E/).

## 1. Overview

- The API is a modest extension on top of combinator libraries like [nom](https://docs.rs/nom/latest/nom/) or [flatparse](https://github.com/AndrasKovacs/flatparse). At the lowest level, it adds two extra combinators:
    - One specifies that a parser **must be** followed by a token of a certain kind.
    - The other specifies that a parser **may be** followed by a token of a certain kind.
- It has minimal runtime overhead during error-free parsing. Note though that this holds
  in an optimized "production" implementation but not in the current demo.
- It uses **greedy error recovery**: whenever the parser would normally throw an unrecoverable
  error, it skips forward to the *earliest possible* position where parsing can resume. This may not
  produce an optimal result in any sense, but fancier solutions seem to be more costly or may
  require pre-processing the entire parser grammar. Subjectively, the recovery behavior feels pretty
  good in practice.
- We purely parse, and don't *repair* the source code in any way, i.e. we don't conceptually insert
  extra tokens that make the input better-formed.
- Users have to make modest changes to parser logic in order to take advantage of error recovery,
  mostly when implementing left-associative operators.
- I use Haskell in this post and in the demo implementation. [Here's the code](https://github.com/AndrasKovacs/resilient-parser-demo).

## 2. Example of usage and behavior

In this section I give an example of a parser that uses the library. I only go into the
implementation details in the next section.

First, let's look at the basic functionality. We have the type `Parser` which happens to be a
`Monad`, an `Alternative` and a `MonadFail`:

```haskell
empty :: Parser a
(<|>) :: Parser a -> Parser a -> Parser a
pure  :: a -> Parser a
(>>=) :: Parser a -> (a -> Parser b) -> Parser b
fail  :: String -> Parser a
```

The error handling follows [nom](https://docs.rs/nom/latest/nom/) and [flatparse](https://github.com/AndrasKovacs/flatparse), where recoverable failure is distinguished from irrecoverable failure. The former is `empty` and the latter is `fail` here.

- Using `<|>`, we can arbitrarily backtrack from `empty`.
- We can't backtrack from `fail`; it's immediately propagated by `<|>`.

This can be contrasted to [Parsec](https://hackage.haskell.org/package/parsec)-inspired libraries,
where different failures are not distinguished, but we can only backtrack from parsing that has not
consumed any input tokens. If we want to backtrack from deeper parsing, we need to wrap the parser
in the `try` combinator. If the grammar happens to be LL(1), we get correct behavior without any
`try`, and also get some non-trivial error reporting out of the box. However, I prefer the "two
failures" setup for production use:
- It is significantly faster, because it contains no machinery for out-of-the-box error reporting.
- Truly nice error messages require careful curation, so if we're aiming for that, we need to
  manually label parsers with errors anyway.

We make some simplifications in the actual parsing API.
- We can only parse strings, i.e. `Char` lists, and assume that every `Char` is a token. In production, we'd want to have a separate lexing phase and do resilient parsing on token sequences.
- Error messages are plain strings and we don't do fancy processing and accumulation of expected items. Fancy logics are very much compatible with the current setup, if someone wants that.

We have a class for embedding parse errors into data types:
```haskell
class EmbedError a where
  mkError :: Maybe a -> Error -> a
```
If the `Maybe a` is a `Just`, that means that we have an error node in a tree to the right of
a non-error node. If it's `Nothing`, it's just a plain error node. We never need to put
an `Error` on the left of a subtree; we'll see that this is ensured by the recovery setup.

A source position is represented simply as a string. We can convert it to an integer position by
taking its length (counting backwards from the end).

An `Error` is a message together with **three** positions.

```haskell
type Msg   = String
type Pos   = String
data Error = Error Msg Pos Pos Pos
```
Why three? In the middle, we have the position where the parser actually got stuck. On the left and right, we have the positions spanning the *error node* in the output. All of these are important: the error position tells the user where to make an edit, while the error node span signals the extent of the *hole* that's carried further into the typechecking phase (because we want to report multiple parsing and type checking errors at the same time!). For example, if the input in the demo parser is
```
λ f. λ g. (λ x.
```
the "highlighting" output that we get is

```
λ f. λ g. (λ x.
          -----^
```
where `^` is the error position and the `-`-s together with `^` mark the error node span,
and the AST output looks like this:
```
Lam
  (Ident 'f')
  (Lam
     (Ident 'g')
     (TmError Nothing ...))
```

Anyway, let's also assume:
```haskell
satisfy' :: (Char -> Bool) -> Parser Char
char'    :: Char -> Parser ()
```
The `'` is my personal naming convention for *backtrackable* parsers, i.e.
parsers that do interesting work before hitting any `fail`. This includes parsers
which never `fail`, like the above two.

Basic lexing:

```haskell
import Data.Char

ws'    = () <$ many (satisfy' isSpace)
sym' c = char' c <* ws'
name'  = satisfy' (\c -> isLower c && c /= 'λ') <* ws'
```
The new combinators used in the current example:

```haskell
infixl 6 <!
(<!) :: EmbedError a => Show a => Parser a -> Char -> Parser a

infixl 6 <?
(<?) :: EmbedError a => Parser a -> Char -> Parser (Either a a)

infixl 6 <|
(<|) :: EmbedError a => Parser a -> Parser a
```

- `p <! c` should be used whenever `p` must be followed by `c`. It runs `p` and consumes `c`.
- `p <? c` should be used whenever `p` may be followed by `c`, and the `Either` in the result
  indicates whether `c` was present and consumed (`Left` means that `c` was consumed).
- `(p <|)` should be used whenever we have no local knowledge about the tokens that might follow
  `p`. For example, bodies of lambda expressions are not bracketed by anything on the right, hence
  should be wrapped in `(<|)`.

We produce the following AST.

```haskell
data Ident_ e
  = Ident Char -- lowercase Char
  | IdentError (Maybe (Ident_ e)) e
  deriving (Show, Foldable)

data Tm_ e
  = Var (Ident_ e)
  | App (Tm_ e) (Tm_ e)
  | Plus (Tm_ e) (Tm_ e)           -- right associative +
  | Mul (Tm_ e) (Tm_ e)            -- left associative *
  | List [Tm_ e]                   -- [t1, t2, ...]
  | Let (Ident_ e) (Tm_ e) (Tm_ e) -- L x = t; u
  | Lam (Ident_ e) (Tm_ e)         -- λ x. t or \x. t
  | TmError (Maybe (Tm_ e)) e
  deriving (Show, Foldable)

type Tm = Tm_ Error
type Ident = Ident_ Error

instance EmbedError Tm    where mkError = TmError
instance EmbedError Ident where mkError = IdentError
```
I use parameterization by `e` to derive the `Foldable` instances that traverse
errors. The actual parser is the following.

```haskell
ident' = Ident <$> name'
ident  = ident' <|> fail "identifier"

atom' = (Var <$> ident')
    <|> (sym' '(' *> tm <! ')')
    <|> (sym' '[' *> pure (List []) <* sym' ']')
    <|> (sym' '[' *> (List <$> nonEmptyList) <! ']')

atom = atom' <|> fail "atomic expression"

nonEmptyList =
  (tm <? ',') >>= \case
    Left t  -> (t:) <$> nonEmptyList
    Right t -> pure [t]

goSpine t =
      (do u <- (atom' <|); goSpine (App t u))
  <|> pure t

spine = goSpine =<< atom

-- left-associative
goMul t =
  (spine <? '*') >>= \case
    Left u  -> goMul (Mul t u)
    Right u -> pure (Mul t u)

mul =
  (spine <? '*') >>= \case
    Left t  -> goMul t
    Right t -> pure t

-- right-associative
plus =
  (mul <? '+') >>= \case
    Left t  -> Plus t <$> plus
    Right t -> pure t

tm :: Parser Tm
tm =
      (Lam <$> ((sym' 'λ' <|> sym' '\\') *> ident <! '.') <*> (tm <|))
  <|> (Let <$> (sym' 'L' *> ident <! '=') <*> (tm <! ';') <*> (tm <|))
  <|> plus
```
This is quite similar to what one would write using a Parsec-style library. We don't
use any higher-level list combinators like `sepBy1`, but they could be defined here
as well in a more resilient way. The main changes from the most naive style are in the
handling of left-associative multiplication and lists.

- For multiplication, we need to duplicate some logic in order to use `<?` as much as possible.
- For lists, we use the somewhat standard trick of handling the non-empty case separately, avoiding
  the use of `many` and the backtracking induced by it.
- Note `(atom' <|)` in `goSpine`: here the possible following tokens can be only computed globally,
  so we don't assume anything. This is a limitation of the API, but I think that it's a reasonable
  trade-off. We'll shortly see how it behaves.

### Examples

The demo implementation lets us either print a textual error highlighting or print
the resulting AST which may contain embedded errors. Let's start with an error-free
example:
```
L f = λ g. λ x. [g x, x + x]; f
```
The AST output is
```haskell
Let
  (Ident 'f')
  (Lam
     (Ident 'g')
     (Lam
        (Ident 'x')
        (List
           [ App (Var (Ident 'g')) (Var (Ident 'x'))
           , Plus (Var (Ident 'x')) (Var (Ident 'x'))
           ])))
  (Var (Ident 'f'))
```
The highlighting output prints the source back, and if we have some error, its position and
enclosing node span is printed on the line below. Taking the input
```
[..., x + x]
```
the highlighting is
```
[..., x + x]
 ^--
```
and the corresponding AST is
```haskell
List [ TmError Nothing "atomic expression" , Plus (Var (Ident 'x')) (Var (Ident 'x')) ]
```
(I only show the message inside `Error` in `TmError`). What happens here:

1. In `tm <? ','` in `nonEmptyList`, `tm` fails when hitting `.`. At the point of failure, we know
that we can possibly recover by skipping ahead to `]`, `+`, `*` or `,`. `fail` skips forward to the
nearest recovery character, which is `,` in this case, then throws an exception.

2. The exception gets caught at `tm <? ','`, which consumes `,` and converts the exception into an error node. We continue parsing normally. The span of the error node is computed at this point too, since we know a) the starting position of `tm` b) the position where `tm` failed c) the position to which we skipped ahead.

As an extra feature, I also keep track of *trailing whitespace* in the error span. So we get
the following output:
```
[...     , x + x]
 ^--
```
Without tracking trailing whitespaces, we'd get
```
[...     , x + x]
 ^-------
```
The following illustrates the limitation I mentioned in application spines:
```
f a b . c d
      ^----
```
So, a parsing failure in a function argument skips the rest of the arguments. The way it works:

1. We consume `f a b` successfully but stop at `.` because it's not an atom.
2. The top-level parser runner sees that we have not consumed all input, so it packages
   `f a b` together with the leftover junk into a `Just` error node.

So, the AST output is

```haskell
TmError (Just (App (App (Var (Ident 'f')) (Var (Ident 'a'))) (Var (Ident 'b'))))
        "expected end of input"
```
Of course, this works inside a larger term as well:
```
L x = f a b . c d; x
            ^----
```
In this case, it is `tm <! ';'` that runs `tm` successfully but finds that `.` is missing. So it skips forward, and `.` happens to be the leftmost recovery token, so it packages the `tm` result into a `Just` error node.

A different scenario:
```
L x = f a b (c d ; x
            -----^
```
Here, it is the `atom'` parsing in `goSpine` that fails, because we go under an opening parenthesis and don't find the matching one. So what `atom' <|` does is to simply catch the failure and convert it to an error node.

Could we skip over ill-formed function arguments, parsing later arguments?

- Well-bracketed function arguments can be already skipped over. For example, in `f a (..) b` we can skip over `(..)`.
- The demo does not have a separate lexer, but if we have one, we can convert lexical errors to error tokens, and then it becomes easy to skip over error tokens.
- Skipping over non-well-bracketed function arguments containing non-lexical errors is not really supported by our API.
  But note that in our demo grammar, such cases would be highly ambiguous. For example, we could repair `f a (c d` either to `f a (c) d` or `f a (c d)`.

Generally speaking, *whitespace-separated operators* introduce ambiguity which makes error recovery
less effective. Token-separated operators work very nicely in comparison.

Consider the following.
```
* +
^ ^^
```
This isn't super spectacular, but the AST output is quite instructive:
```haskell
Plus
  (Mul
     (TmError Nothing "atomic expression")
     (TmError Nothing "atomic expression"))
  (TmError Nothing "atomic expression")
```
So we have three errors, because the input could be repaired to `a * b + c` for some `a, b, c`, and
note that the precedences of `*` and `+` are correctly handled.

Let's look at an example for *greedy recovery*:
```
(f [x, y), z])
   -----^^----
```
We get the first error when hitting an unexpected `)`. We recover to `)`, meaning
that we just stay in place, `[x y` becomes an error node and we return an `f` application.
But now we have some leftover junk which turns into another error in the top-level runner.

Here the recovery chooses skipping ahead to `)` because it comes first. Another option could have
been to skip ahead to `,` instead, which would have produced just a single error span for `)`.

Could we do better in general? The extreme solution is to try to parse from every recovery point and
pick the result with the overall least amount of parse errors. This is actually not that bad using
*multi-shot delimited continuations*, which are
[available](https://ghc-proposals.readthedocs.io/en/latest/proposals/0313-delimited-continuation-primops.html)
in GHC today. This way, we don't get overheads in error-free parsing. But error recovery can easily
blow up still, so it needs some extra pruning/optimization, and it's not clear to me if the extra
complexity is worth it.

Here's another interesting example:
```
f ((g ?)
  ------^
```
This illustrates that we *don't track nested parse errors*. On the inside, `(g ?)` gets parsed to a partial term, but then we have a missing `)` on the outside, so the AST of `(g ?)` gets discarded and replaced with a fresh error.

Tracking nested errors is actually not difficult: at every `p <! c`, if `p` succeeds but `c` does
not follow, we collect the list of errors occurring in the result of `p` and attach it to the fresh
error. This means that every error is now a tree containing possibly nested spans. It would be a fun
feature, but again I'm not sure if it's super useful and I didn't want to bother displaying nested
spans.

Finally, here's a bigger example just for illustration:
```
L e a b = λ f. λ y. f y y ( ?  ?  y;
    ^--                   ---------^
L b = [x, y, z, ?? + x * ?? + x, ((), k];
                ^-       ^-      ---^
L f = λ x. λ y  ? ;
           -----^
[a, b, c, λλλλλ]
          -^---
```

## 3. Implementation

Let's proceed to the dirty details. There's nothing very surprising up until the `Alternative` instance
definition. We have `Empty` for recoverable failure and `Fail` for unrecoverable failure. The extra thing so far is just a `Set Char` passed as reader environment.
```haskell
type Recovery = Set Char

data Res a
  = OK a String
  | Fail Msg Pos Pos Pos
  | Empty
  deriving (Show, Functor)

newtype Parser a = Parser {runParser :: Recovery -> String -> Res a}
  deriving Functor

instance Applicative Parser where
  pure a = Parser \r s -> OK a s
  (<*>) = ap

instance Monad Parser where
  return = pure
  Parser f >>= g = Parser \r s -> case f r s of
    OK a s            -> runParser (g a) r s
    Fail msg s s' s'' -> Fail msg s s' s''
    Empty             -> Empty

instance Alternative Parser where
  empty = Parser \_ _ -> Empty
  Parser f <|> Parser g = Parser \r s -> case f r s of
    Empty -> g r s
    res   -> res
```

Things get more interesting in the `MonadFail` instance.

```haskell
recover :: Recovery -> String -> (String, String)
recover r s = go s s where
  go stripped (c:s)
    | Set.member c r = (stripped, c:s)
    | isSpace c      = go stripped s
    | otherwise      = go s s
  go stripped [] = (stripped, [])

instance MonadFail Parser where
  fail msg = Parser \r s0 -> case recover r s0 of
    (s1, s2) -> Fail msg s0 s1 s2
```

`recover` produces two strings. The second one is obtained by skipping forward until we hit a
character in the recovery set. The first string corresponds to the position that strips all
whitespace characters just before the recovery position. For example
```
recover (singleton 'x') "a b   x"
```
returns
```
("   x", "x")
```
Note that `fail` returns the following three positions:

- The error position `s0`
- The whitespace-stripped end position of the error span `s1`.
- The recovery position `s2`.

This triple does not have the same meaning as the triple in `Error`!

Also recall `EmbedError`:
```haskell
class EmbedError a where
  mkError :: Maybe a -> Error -> a
```
The low-level API consists of the following three functions:
```haskell
mustFollow    :: EmbedError a => [Char] -> Parser a -> Parser (a, Char)
mayFollow     :: EmbedError a => [Char] -> Parser a -> Parser (Either (a, Char) a)
eofMustFollow :: EmbedError a => Parser a -> Parser a
```
`eofMustFollow` is really just for running the top-level parser,
converting stray `Fail`-s and `Empty`-s to error values.

`mustFollow` and `mayFollow` generalize `(<!)` and `(<?)` respectively, taking
a list of recovery characters. The lists are actually treated as sets here; they
are lists so that users can write list syntax instead of the noisier `Set` syntax.
So, `mustFollow` and `mayFollow` both return the character that actually followed the
parser.

Let's look at `mustFollow`.
```haskell
mustFollow :: EmbedError a => [Char] -> Parser a -> Parser (a, Char)
mustFollow cs p = Parser \r s0 ->
  let cSet = Set.fromList cs in
  let r'   = r <> cSet in

  -- we run p under the extended recovery set
  case runParser p r' s0 of

    OK a s1
      | c:s2 <- s1, Set.member c cSet ->
        -- if p succeeds and a valid Char follows, return normally
        OK (a, c) s2

        -- otherwise try to find a valid Char by skipping forward
      | (s2, s3) <- recover r' s1 ->
        case s3 of
          -- if we find it, annotate the value with an error on the right
          c:s4 | Set.member c cSet ->
            OK (mkError (Just a) (Error cs s1 s1 s2), c) s4
          -- if we don't find it, fail. This will be handled
          -- somewhere up in the call stack. Note: this is the point
          -- where we'd need to extract errors from "a" in order to
          -- track nested errors.
          s3 ->
            Fail ("expected one of " ++ show cs) s1 s2 s3

    Fail msg s1 s2 s3
      -- if p fails and a valid Char follows, return an error value
      | c:s4 <- s3, Set.member c cSet ->
        OK (mkError Nothing (Error cs s0 s1 s2), c) s4

      -- otherwise propagate the Err
      | otherwise ->
        Fail msg s1 s2 s3

    -- Empty is always directly propagated by error recovery
    Empty -> Empty
```

If your impression is that this is quite subtle, I agree! It took me around four attempts to arrive
at this version, and the positions and spans are also easy to get wrong. This definition seems kind
of *forced*, but in any case I'd be glad to see suggestions for alternative behaviors and combinator
sets.

For some higher-level illustration, assume that we only use `(<!)` for recovery in a parser. In this
case, we're always parsing under a stack of closing brackets that will have to be consumed. If we
hit a `fail`, we skip to the nearest closing bracket, throw a `Fail`, and catch it at the innermost
`(<!)` call that expects the closing bracket that we found. There are two kinds of greediness at play:

1. Skipping over the least number of tokens on failure.
2. Catching the failure at the innermost stack frame, which means dropping the least
   number of frames and producing the greatest number of non-error AST nodes.

Let's proceed to `mayFollow`.
```haskell
mayFollow :: EmbedError a => [Char] -> Parser a -> Parser (Either (a, Char) a)
mayFollow cs p = Parser \r s0 ->
  let cSet = Set.fromList cs in
  let r'   = r <> cSet in

  -- run p under the extended recovery set
  case runParser p r' s0 of
    OK a s1
      | c:s2 <- s1, Set.member c cSet ->
        -- if p succeeds and a valid Char follows, return Left
        OK (Left (a, c)) s2
      | otherwise ->
        -- otherwise return a Right
        OK (Right a) s1
    Fail msg s1 s2 s3
      | c:s4 <- s3, Set.member c cSet ->
        -- if p fails and a valid Char follows, return Left of an error value
        OK (Left (mkError Nothing (Error msg s0 s1 s2), c)) s4
      | otherwise ->
        -- otherwise return Right of an error value
        OK (Right (mkError Nothing (Error msg s0 s1 s2))) s3
    Empty -> Empty
```
This looks a bit simpler! Notice that it succeeds in every case except the `Empty` one.
So, all failures are caught and converted to error values. This is fine, because
the following character set is optional, and we can continue execution at the point
of `mayFollow` regardless of the following character.

In fact, `(<|)` is defined as `mayFollow` with an empty set:

```haskell
(<|) :: EmbedError a => Parser a -> Parser a
(<|) p = either fst id <$> (mayFollow [] p)
```

What happens if we needlessly wrap parsers in `(<|)`? As far as I understand, nothing too horrible;
it's just unnecessary runtime overhead.

Finally, I think that `eofMustFollow` doesn't need much commentary:

```haskell
eofMustFollow :: EmbedError a => Parser a -> Parser a
eofMustFollow p = Parser \r s0 -> case runParser p r s0 of
  OK a "" -> OK a ""
  OK a s1 -> case recover mempty s1 of
    (s2, _) -> OK (mkError (Just a) (Error "expected end of input" s1 s1 s2)) ""
  Fail msg s1 s2 "" -> OK (mkError Nothing (Error msg s0 s1 s2)) ""
  Fail msg s1 s2 s3 -> case recover mempty s3 of
    (s4, _) -> OK (mkError Nothing (Error msg s0 s2 s4)) ""
  Empty -> OK (mkError Nothing (Error "unknown parse error" s0 s0 "")) ""
```

Overall, there are two things that I'm not quite happy with in this implementation:

- It seems that a `fail` is being inlined into `mustFollow`, in the `OK` branch with
  a missing following character. This suggests that `mustFollow` should be factored
  into simpler combinators. But my attempts to do this haven't succeeded so far.
- `eofMustFollow` should be likewise expressible from other simpler combinators. I didn't
  try hard to remedy this and I don't think that it's a big deal.

## Performance

I don't have any benchmarks, I just write a bit about performance here. The main objective is to
minimize overhead on error-free parsing. On that front we can do really well. In Pterodactyl I'm
writing a highly optimized version which looks like this:

- There's a separate lexing phase that produces tokens. A token consists of a tag ("kind") and a span.
- Since there are not many different kinds, a set of kinds is just a 64-bit bitset.
- The recovery combinators are inlined (of course), but all the failing branches are outlined
  explicitly, in order to decrease code size. So recovery operators are not much bigger than
  plain monadic binding.

Recovery would be a bit more expensive in the presence of *user-defined mixfix operators*.
We would like to have the same kind of recovery there as well. Naturally, such operators
don't fit into a 64-bit bitset, so we need some real set structure. In Pterodactyl, mixfix
custom operators will be only resolved during elaboration, so normal parsing can still use
bitsets.

## Related work

I did a moderate amount of literature search but did not find the feature set described here.  If I
missed something and I'm just reinventing a wheel here, please shout at me! Anyway, two works were
the closest:

First, [Syntax Error Recovery in Parsing Expression Grammars](https://arxiv.org/abs/1806.11150) by
Medeiros and Mascarenhas. Similarly to our setup, this starts with PEG parsers extended with
irrecoverable labeled errors. Then, it makes it possible to associate to each labeled error (`fail`
in our case) a small parser that's used for recovery. Recovery parsers are static and have to be
written by hand or computed from the grammar. For LL(1) grammars we can compute good quality
recovery parsers, but of course fancier grammars need manual tuning. In contrast, our design doesn't
require hand-written recovery parsers, instead it uses a dynamic approximation of follow sets, and
the user's obligation is to set up those follow sets to be accurate enough.

Second, [Deterministic, Error-Correcting Combinator
Parsers](https://www.cs.tufts.edu/~nr/cs257/archive/doaitse-swierstra/LL1.pdf) by Swierstra and
Duponcheel. This requires no extra annotations, but it's is restricted to LL(1) grammars and has
significant runtime overhead. The "noSkip" set that's tracked here is reminiscent of our recovery
set, but to be honest I would need to look harder at this work to understand the differences.
