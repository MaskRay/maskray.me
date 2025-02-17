---
layout: post
title: Haskell實現的Splay樹
author: MaskRay
tags: [haskell, data structure]
---

<!-- more -->

三週的軍訓總算挺過去了，這裏的網絡條件比想象中要糟糕不少。
其實有很多要說，還是等到“十一長假”回家了再慢慢說吧。

廢話不多說了，這是一個用 `Haskell` 實現的 `Top-down Splay tree`：

```haskell
module SplayTree (
  SplayTree,
  splay,
  insert,
  delete,
  empty,
  ) where

data SplayTree a = Nil | Node a (SplayTree a) (SplayTree a)
            deriving (Eq, Show)

splay :: (Ord a) => (a -> Ordering) -> SplayTree a -> SplayTree a
splay comp t = walk t Nil Nil
  where
    walk Nil _ _ = Nil
    walk t@(Node nx l r) lspine rspine =
      case comp nx of
        LT -> case l of
          Nil -> final t lspine rspine
          Node nl a b -> if comp nl == LT && a /= Nil then walk a lspine (Node nl rspine (Node nx b r))
                         else walk l lspine (Node nx rspine r)
        GT -> case r of
          Nil -> final t lspine rspine

          Node nr c d -> if comp nr == GT && d /= Nil then walk d (Node nr (Node nx l c) lspine) rspine
                         else walk r (Node nx l lspine) rspine
        EQ -> final t lspine rspine

    final g@(Node x l r) lspine rspine = Node x (lfinal l lspine) (rfinal r rspine)
    lfinal l Nil = l
    lfinal l (Node y a b) = lfinal (Node y a l) b
    rfinal r Nil = r
    rfinal r (Node y a b) = rfinal (Node y r b) a

insert :: (Ord a) => a -> SplayTree a -> SplayTree a
insert key Nil = Node key Nil Nil
insert key t =
  let t'@(Node nx l r) = splay (compare key) t
  in if key < nx then Node key l (Node nx Nil r)
     else Node key (Node nx l Nil) r

delete :: (Ord a) => a -> SplayTree a -> SplayTree a
delete key Nil = Nil
delete key t =
  let t'@(Node nx l r) = splay (compare key) t
  in case compare key nx of
    EQ -> if l == Nil then r
          else (\(Node nl a _) -> Node nl a r) $ splay (const GT) l
    _ -> t'

empty = Nil

-- Test.QuickCheck

prop_insert_delete :: [Int] -> Bool
prop_insert_delete xs = foldr delete (foldr insert empty xs) xs == Nil
```
