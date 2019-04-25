---
layout: post
title:  "简单的 Parsec 实现"
date:   2019-4-23 00:00:00 +0000
categories:
  - post
tags:
  - Haskell
  - Parsec
---

安装 [`markdown-unlit`](http://hackage.haskell.org/package/markdown-unlit) 并使用 `-pgmL markdown-unlit` 选项，可在 `ghci` 中载入并运行本文（需将后缀名改为 `.lhs`）。

_这里是一堆扩展和导入_

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE FunctionalDependencies #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE MultiWayIf #-}

module MinimalParsec where

import Data.Char
import Data.Either
import Data.Monoid
import Control.Arrow
import Control.Monad
import Control.Applicative
import Control.Monad.Trans
import Control.Monad.State
import Control.Monad.Writer
import Control.Monad.Except
import Control.Monad.Identity
import qualified Control.Monad.Fail as Fail
```

_`Control.Monad.Fail` 在 GHC 8.0 引入用于淘汰 `Monad` 类型类的 `fail` 方法。以下实现若要在旧版本 GHC 使用，可能需要额外定义 `fail` 方法。_

本文的 Parsec 仅作为 Parser Combinator 的缩写，并非特指 [Parsec](http://hackage.haskell.org/package/parsec) 库。常见的 Parsec 库有 [Parsec](http://hackage.haskell.org/package/parsec)，[Attoparsec](http://hackage.haskell.org/package/attoparsec)，[ReadP](http://hackage.haskell.org/package/base-4.12.0.0/docs/Text-ParserCombinators-ReadP.html)，[Megaparsec](http://hackage.haskell.org/package/megaparsec) 等等。在这些库不可用的时候，实现一个基础的 Parsec 库也是很好的 Haskell 练习。比如爱丁堡大学的 [Informatics 1 - Introduction to Computation](http://www.drps.ed.ac.uk/18-19/dpt/cxinfr08025.htm) 的 [Monads 部分](https://www.learn.ed.ac.uk/bbcswebdav/pid-3432293-dt-content-rid-7167241_1/courses/INFR080252018-9SV1SEM1/lectures/fp/lect20.pdf) 就使用了 Parser Monad 作为例子，而这也可以作为进一步实现更完善的 Parsec 库的基础。

UoE 的课件中引用了一个押韵的定义：

>_A parser for things_  
>_Is a function from strings_  
>_To lists of pairs_  
>_Of things and strings_  
>      _— Graham Hutton_

该定义出现于 Graham Hutton 所著的 Programming in Haskell [[1]](#1)，用 Haskell 代码表示即为：

```haskell
type HuttonParsec a = String -> [(String, a)]
```

这是一个经典的定义，更详细的相关内容可以参考 UoE 的课件。然而，这也是一个简单且特化的定义：它只能处理 `String`，也就是 `[Char]` 类型的输入，而我们希望能够处理任意类型的 token 序列；它表示了一个非确定的 parser，也就是它会以列表的形式返回多个 parsing 结果，诚然这里的列表也起到了表示可能失败的 parsing 的作用，但实际中，比起非确定性，我们更关心 parsing 是否成功以及如果成功，结果是什么；最后，除了非确定性，我们可能需要管理更丰富的副作用以及副作用的组合，同时最大程度复用代码，这个定义缺乏了对应的抽象。

首先是处理任意 token 序列的问题。在 [[2]](#2) 中 token 被作为类型参数 `a` 抽象出去，所以作为输入的是泛化了的 `[a]`。这里我们做进一步的抽象——形似于 [Parsec](http://hackage.haskell.org/package/parsec) 中的 `Stream` 类型类定义——对于一个 token 序列 `s`，我们只需要知道如何**连续地**从这个序列中取出类型为 `t` 的 token（`s -> (t, s)`），以及是否已经取完（`s -> Maybe (t, s)`）。列表类型自然地成为了 `Stream` 类型类的实例。

```haskell
-- | Stream of tokens
class Stream s t | s -> t where
  uncons :: s -> Maybe (t, s)

instance Stream [a] a where
  uncons [] = Nothing
  uncons (x : xs) = Just (x, xs)
```

再来看关于副作用管理的部分。`HuttonParsec` 的定义强烈提示了它与 `StateT` 单子变换器有关，因为它具有类似于 `s -> m (a, s)` 的形式，而单子变换器也正是之前提出的问题的一种解决办法。其实在 [[3]](#3) 中就已经指出，`HuttonParsec` 可以被如下定义所替代。

```haskell
-- @newtype StateT s m a = StateT { runStateT :: s -> m (a,s) }@
type HuttonParsecStateT a = StateT String [] a
```

这样的好处是我们免费获得了所有为 `StateT` 定义的类型类实例——包括 `Alternative`/`MonadPlus` 这样在 Parsec 实现中非常重要的类型类（因为底层单子 `[]` 是 `MonadPlus` 的实例）。于是，在下面的代码中，我们更进一步，将 token 序列类型和底层单子都抽象出来，这时我们的定义就和 `StateT` 没什么区别了。为了与已有的 `StateT` 区分，我们用 `newtype` 做包裹，同时使用 `GeneralizedNewtypeDeriving` 扩展来免费获得各种需要的类型类实例。

```haskell
-- | Backtracking Parser
-- @s -> m (a, s)@
newtype BTParsecT s m a = BTParsecT (StateT s m a)
  deriving (Functor, Applicative, Monad, MonadTrans, MonadPlus, Alternative, Fail.MonadFail, MonadState s)
```

剩下的事情便是让 `BTParsecT` 真正可用。除了常规的 `runBTParsecT` 和 `evalBTParsecT`，我们还需要补充生成针对单个 token 的 parser 的 `itemBT`，以及表示期望得到文件尾的 `eofBT`。需要注意的是，因为希望尽量精简，此处以及下文中的 Parsec 实现都没有留存位置数据，所以报错能提供的信息十分有限。

```haskell
runBTParsecT :: BTParsecT s m a -> s -> m (a, s)
runBTParsecT (BTParsecT p) = runStateT p

evalBTParsecT :: (Monad m) => BTParsecT s m a -> s -> m a
evalBTParsecT p = fmap fst . runBTParsecT p

itemBT :: (Fail.MonadFail m, Stream s t) => (t -> Maybe a) -> BTParsecT s m a
itemBT f = do
  s <- get
  case uncons s of
    Nothing -> Fail.fail "unexpected end of input"
    Just (t, s') -> case f t of
      Nothing -> Fail.fail "mismatched token"
      Just a -> put s' >> return a

eofBT :: (Fail.MonadFail m, Stream s t) => BTParsecT s m ()
eofBT = do
  s <- get
  case uncons s of
    Nothing -> return ()
    _ -> Fail.fail "unexpected token"
```

下面给出一些例子。

```haskell
-- | Sample parser combinators
charBT :: (Fail.MonadFail m, Stream s Char) => Char -> BTParsecT s m Char
charBT c = itemBT (\t -> if t == c then Just c else Nothing)

stringBT :: (Fail.MonadFail m, Stream s Char) => String -> BTParsecT s m String
stringBT = mapM charBT

digitBT :: (Fail.MonadFail m, Stream s Char) => BTParsecT s m Char
digitBT = itemBT (\t -> if isDigit t then Just t else Nothing)

letterBT :: (Fail.MonadFail m, Stream s Char) => BTParsecT s m Char
letterBT = itemBT (\t -> if isLetter t then Just t else Nothing)

spcBT :: (Fail.MonadFail m, Stream s Char) => BTParsecT s m Char
spcBT = itemBT (\t -> if isSpace t then Just t else Nothing)

sepByBT :: (Fail.MonadFail m, MonadPlus m, Stream s Char) => BTParsecT s m a -> BTParsecT s m b -> BTParsecT s m [a]
sepByBT a b = liftA2 (:) a (many (b *> a)) <|> pure []
```

其中，`sepByBT a b` 意为被 `b` 隔开的一系列 `a`，比如 `sepByBT digit (char ',')` 可将 `"1,2,3"` 解析为 `['1', '2', '3']`。`sepByBT` 中的 `many` 由 `Alternative` 类型类提供，意为零次或多次应用某个 parser；`<|>` 亦由 `Alternative` 类型类提供，在此处起到选择的作用，即 `a <|> b` 先尝试 `a`，若成功则返回 `a`（的值），失败则尝试 `b`。

到此为止，一切看起来都很顺利。然而，如果我们将 `sepByBT` 和 `Text.Parsec` 中的 [`sepBy`](http://hackage.haskell.org/package/parsec-3.1.13.0/docs/src/Text.Parsec.Combinator.html#sepBy) 比较，就会发现它们的表现是不同的。

```
> parse (sepBy digit (char ',')) "" "1,"
Left (line 1, column 3):
unexpected end of input
expecting digit

> evalBTParsecT (sepByBT digitBT (charBT ',')) "1,"
"1"
```

导致这个区别的原因是，`Text.Parsec` 是默认 predictive，或者说 {$LL(1)$} 的（[UoE 相关课件](https://www.inf.ed.ac.uk/teaching/courses/ct/18-19/slides/5-parsing.pdf)），在使用 `try` 等组合子时则可以获得任意的 lookahead；而 `BTParsecT` 总是 full backtracking，或者说 {$LL(\infty)$} 的 [[4]](#4)。针对此处的例子，主要的区别体现在 `sepBy` 和 `sepByBT` 都使用了的 `many` 中的 `<|>`。`sepBy` 和 `many` 的简化版定义如下。

```
sepBy p sep = liftA2 (:) p (many (sep *> p)) <|> pure []

many p = liftA2 (:) p (many p) <|> pure []
```

_`Text.Parsec` 中原本的 `sepBy` 使用了单子风格，此处为了与 `sepByBT` 对应而改写为 Applicative 风格。容易观察到，`sepBy` 和 `sepByBT` 的定义是一致的。_

对于 `sepBy digit (char ',')`，在成功消耗了 `'1'` 之后，会执行 `many (char ',' *> digit)`，也就是：

```
liftA2 (:) (char ',' *> digit) (many (char ',' *> digit)) <|> pure []
```

此处，`char ',' *> digit` 成功消耗了 `','`，随后期望得到 `digit`，却发现已经到了序列尾，于是产生了错误信息。同时，由于已经消耗了一个 token（被 `char ','` 消耗的 `','`），按照 `Text.Parsec` 中 `<|>` 的定义，其右侧的 `pure []` 不再被尝试。而对于 `sepByBT` 来说，在 `charBT ',' *> digitBT` 消耗了 `','` 并失败之后，`<|>` 会**恢复**到输入被消耗前的位置尝试执行 `pure []`。由于 `pure []` 永远成功并返回 `[]`，我们便得到了上面的结果。

若用 `Text.Parsec` 的行为进行类比，`BTParsecT` 相当于到处都加上了 `try`，而 `try` 是很贵的 [[5]](#5)。[[4]](#4) 也指出，full backtracking 可能会导致严重的空间泄露，同时也会让提供精准的错误信息变得困难。为了解决这些问题，[[4]](#4) 提供了一种默认 predictive 的 Parsec 实现思路——实际上，[[4]](#4) 就是 `Text.Parsec` 的原型。上文提到过，我们并不关心详细的错误信息，所以我们参考 [[4]](#4) 中的基础实现即可。[[4]](#4) 中基础的 `Parser` 类型定义如下。

```haskell
type Parser a = String -> Consumed a

data Consumed a = Consumed (Reply a) | Empty (Reply a)

data Reply a = Ok a String | Error
```

注意到 `Consumed a` 同构于 `(Reply a, Bool)`，且 `Consumed a` 满足以下规则 [[4]](#4)：

p | q | (p >>= q)
--- | --- | ---
Empty | Empty | Empty
Empty | Consumed | Consumed
Consumed | Empty | Consumed
Consumed | Consumed | Consumed

如果把 `Empty` 换成 `False`，`Consumed` 换成 `True`，我们会发现这个规则和 `Bool` 关于 `||` 形成的 `Any` 幺半群是一致的（当然，换一种定义方式，`All` 也可以），而这提示“记录是否消耗了输入”这个副作用可以交给 `Writer Any` 单子来管理。同时，注意到 `Reply a` 同构于 `Maybe (a, String)`，那么 `Parser a` 就同构于 `String -> (Maybe (a, String), Any)`，使用单子变换器重写便是 `StateT String (MaybeT (Writer Any)) a`。随后，将 `String` 抽象出去，用 `ExceptT String` 换掉 `MaybeT` 以提供基本的错误信息，加入底层单子，新的 Parsec 类型定义便成型了。

```haskell
-- | Predictive Parser
-- @s -> m (Either String (a, s), Any)@
newtype PDParsecT s m a = PDParsecT (StateT s (ExceptT String (WriterT Any m)) a)
  deriving (Functor, Applicative, Monad, MonadState s, MonadWriter Any)

runPDParsecT :: PDParsecT s m a -> s -> m (Either String (a, s), Any)
runPDParsecT (PDParsecT p) = runWriterT . runExceptT . runStateT p 

evalPDParsecT :: (Monad m) => PDParsecT s m a -> s -> m (Either String a)
evalPDParsecT p = fmap (either (Left . id) (Right . fst) . fst) . runPDParsecT p
```

可以看到，我们同样使用了 `GeneralizedNewtypeDeriving` 扩展来免费获得一些类型类的实例，但不同于 `BTParsecT`，此处若仍然自动生成 `Alternative` 等类型类的实例，我们将无法得到期望的 predictive 特性（[为什么？](http://hackage.haskell.org/package/transformers-0.5.6.2/docs/src/Control.Monad.Trans.Except.html#ExceptT)），所以我们需要自己提供符合要求的实现。需要注意的是，[[4]](#4) 中的基础 Parser 实现因为不携带任何错误信息，所以在实现 `pA <|> pB` 时对于 `pA` 失败但没有消耗输入的情况，直接执行 `pB` 即可；但对于此处使用了 `ExceptT` 的情况，为了保证 `empty` 是 `<|>` 的单位元，我们还需要对错误信息是否为空进行检查，否则返回空错误信息的 `pB` 可能会覆盖带有非空错误信息的 `pA`。

```haskell
isConsumed = getAny . snd
isFailed = isLeft . fst
isSucc = isRight . fst

instance (Monad m) => Alternative (PDParsecT s m) where
  empty = PDParsecT empty
  pA <|> pB = PDParsecT . StateT $ \s -> ExceptT . WriterT $ do
    rA <- runPDParsecT pA s
    if isConsumed rA then return rA else do
      rB <- runPDParsecT pB s
      return $ if | isConsumed rB -> rB
                  | isSucc rA -> rA
                  | otherwise -> case fst rB of {Left "" -> rA; _ -> rB}

instance (Monad m) => MonadPlus (PDParsecT s m) where
  mzero = empty
  mplus = (<|>)
  
instance (Monad m) => Fail.MonadFail (PDParsecT s m) where
  fail = PDParsecT . StateT . const . ExceptT . return . Left

instance MonadTrans (PDParsecT s) where
  lift m = PDParsecT . StateT $ \s -> ExceptT . WriterT $ do
    a <- m
    return (Right (a, s), mempty)
```

`itemPD` 和 `eofPD` 的定义与 `itemBT` 和 `eofBT` 大同小异。

```haskell
itemPD :: (Monad m, Stream s t) => (t -> Maybe a) -> PDParsecT s m a
itemPD f = do
  s <- get
  case uncons s of
    Nothing -> Fail.fail "unexpected end of input"
    Just (t, s') -> case f t of
      Nothing -> Fail.fail "mismatched token"
      Just a -> put s' >> tell (Any True) >> return a

eofPD :: (Monad m, Stream s t) => PDParsecT s m ()
eofPD = do
  s <- get
  case uncons s of
    Nothing -> return ()
    _ -> Fail.fail "unexpected token"
```

虽然默认 predictive，我们还是会有需要任意长的 lookahead，或者说需要 backtracking 的时候 [[4]](#4)。和 `Text.Parsec` 的 `try` 同理，`tryPD p` 会在 `p` 失败时伪装成没有消耗任何输入的样子——更详细的解释可参考 [[4]](#4) 的 3.4 节。

```haskell
tryPD :: (Monad m) => PDParsecT s m a -> PDParsecT s m a
tryPD p = PDParsecT . StateT $ \s -> ExceptT . WriterT $ do
  r @ (e, _) <- runPDParsecT p s
  return (if isLeft e then (e, Any False) else r)
```

下面给出一些例子。

```haskell
-- | Sample parser combinators
charPD :: (Monad m, Stream s Char) => Char -> PDParsecT s m Char
charPD c = itemPD (\t -> if t == c then Just c else Nothing)

stringPD :: (Monad m, Stream s Char) => String -> PDParsecT s m String
stringPD = mapM charPD

digitPD :: (Monad m, Stream s Char) => PDParsecT s m Char
digitPD = itemPD (\t -> if isDigit t then Just t else Nothing)

letterPD :: (Monad m, Stream s Char) => PDParsecT s m Char
letterPD = itemPD (\t -> if isLetter t then Just t else Nothing)

spcPD :: (Monad m, Stream s Char) => PDParsecT s m Char
spcPD = itemPD (\t -> if isSpace t then Just t else Nothing)

sepByPD :: (Monad m, Stream s Char) => PDParsecT s m a -> PDParsecT s m b -> PDParsecT s m [a]
sepByPD a b = liftA2 (:) a (many (b *> a)) <|> pure []
```

最后，我们使用 `PDParsecT` 来实现以下 EBNF 描述的语法的 parser。以下语法参考了 [Tiny Three-Pass Compiler](https://www.codewars.com/kata/tiny-three-pass-compiler)。

```
expression := term | expression "+" term | expression "-" term

term := factor | term "*" factor | term "/" factor

factor := number | variable | "(" expression ")"

variable := letter {letter}

letter := "A" | "a" | "B" | "b" | ... | "Z" | "z"

number := digit {digit}

digit := "0" | "1" | "2" | ... | "9"
```

```haskell
chainl1PD :: (Monad m) => PDParsecT s m a -> PDParsecT s m (a -> a -> a) -> PDParsecT s m a
chainl1PD p op = p >>= rest
  where rest x = (op <*> pure x <*> p >>= rest) <|> pure x

someSpcPD :: (Monad m, Stream s Char) => PDParsecT s m ()
someSpcPD = () <$ some spcPD

manySpcPD :: (Monad m, Stream s Char) => PDParsecT s m ()
manySpcPD = () <$ many spcPD

data Expr = Imm Int
          | Var String 
          | Add Expr Expr 
          | Sub Expr Expr 
          | Mul Expr Expr 
          | Div Expr Expr
          deriving (Show, Eq)

numberPD :: (Monad m, Stream s Char) => PDParsecT s m Expr
numberPD = (Imm . read) <$> some digitPD

variablePD :: (Monad m, Stream s Char) => PDParsecT s m Expr
variablePD = Var <$> some letterPD

factorPD :: (Monad m, Stream s Char) => PDParsecT s m Expr
factorPD = numberPD <|> variablePD <|> (charPD '(' *> manySpcPD *> exprPD <* manySpcPD <* charPD ')')

termPD :: (Monad m, Stream s Char) => PDParsecT s m Expr
termPD = chainl1PD (factorPD <* manySpcPD) (((Mul <$ charPD '*') <|> (Div <$ charPD '/')) <* manySpcPD)

exprPD :: (Monad m, Stream s Char) => PDParsecT s m Expr
exprPD = chainl1PD (termPD <* manySpcPD) (((Add <$ charPD '+') <|> (Sub <$ charPD '-')) <* manySpcPD)
```

简单地测试一下。

```
> evalPDParsecT exprPD "1  +xyz *  3/    5-(6 + 2)" :: Identity (Either String Expr)
Identity (Right (Sub (Add (Imm 1) (Div (Mul (Var "xyz") (Imm 3)) (Imm 5))) (Add (Imm 6) (Imm 2))))
```

##### [1]
HUTTON, G. _Programming in Haskell_, 2nd ed. Cambridge University Press, New York, NY, USA, 2016, ch. 13, pp. 177–195.

[INFO](http://www.cs.nott.ac.uk/~pszgmh/pih.html)

##### [2]
HUTTON, G. Higher-order functions for parsing. _Journal of Functional Programming 2_, 3 (1992), 323–343.

[PDF](http://www.cs.nott.ac.uk/~pszgmh/parsing.pdf)

##### [3]
HUTTON, G., AND MEIJER, E. Monadic parser combinators. Tech. Rep. NOTTCS-TR-96-4, Department of Computer Science, University of Nottingham, 1996.

[PDF](http://www.cs.nott.ac.uk/~pszgmh/monparsing.pdf)

##### [4]
LEIJEN, D., AND MEIJER, E. Parsec: Direct style monadic parser combinators for the real world. Tech. Rep. UU-CS-2001-35, Departement of Computer Science, Universiteit Utrecht, July 2001.

[PDF](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/parsec-paper-letter.pdf)

##### [5]
O’SULLIVAN, B., GOERZEN, J., AND STEWART, D. _Real World Haskell_, 1st ed. O’Reilly Media, Inc., 2008, ch. 16, p. 402.

[PDF](http://pv.bstu.ru/flp/RealWorldHaskell.pdf)

[HTML](http://book.realworldhaskell.org/read/using-parsec.html)

[中文](http://cnhaskell.com/chp/16.html#id5)