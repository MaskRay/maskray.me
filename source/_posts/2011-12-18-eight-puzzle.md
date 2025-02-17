---
layout: post
title: 八數碼
author: MaskRay
tags: [algorithm, haskell, search]
---

八數碼問題，已訪問狀態採用 `factorial number system` 表示，未訪問的未使用（簡化代碼）
實現了 `Breadth First Search` 和 `Heuristic Search` 兩種算法。
帶上命令行選項 -g 能輸出 `Graphviz` 的 `dot` 格式的狀態樹。
比較滿意的地方是把兩種搜索算法的共同部分抽象出來了，寫成了單獨的 `search` 函數。

<!-- more -->

```haskell
{-# LANGUAGE CPP, FlexibleInstances, TypeSynonymInstances, ViewPatterns #-}
{-
Input "123485760"
stands for

1 2 3
4 8 5
7 6 0

Add -g to generate code for graphviz
 -}
import Control.Monad
import Data.List
import Data.Maybe
import Data.Function
import qualified Data.Sequence as Seq
import qualified Data.Map as M
import qualified Data.MultiSet as SS
import System.Environment
import System.IO

type State = [Int]
target = [1..8]++[0] :: State
target' = fromEnum target

factorials = 1 : scanl1 (*) [1..]

instance Enum State where
  fromEnum a = (\(_,_,acc) -> acc) $ foldr (\x (i,l,acc) ->
      (i+1,x:l,acc+(factorials!!i)*length (filter (<x) l))) (0,[],0) a
  toEnum acc = unfoldr (\(i,l,acc) ->
      if i < 0 then Nothing
      else let (q,r) = acc `divMod` (factorials !! i)
               x = l !! q
           in Just (x, (i-1,delete x l,r))
          ) (8,[0..8],acc)

moves :: State -> [State]
moves s = [ map (\x -> if x == 0 then s!!pos' else if x == s!!pos' then 0 else x) s
          | d <- [-1,3,1,-3]
          , not $ pos `mod` 3 == 0 && d == (-1)
          , not $ pos `mod` 3 == 2 && d == 1
          , let pos' = pos + d
          , not $ pos' < 0 || pos' >= 9
          ]
  where
    pos = fromJust $ findIndex (==0) s

solve :: (State -> M.Map Int Int) -> State -> IO ()
solve strategy src = do
    let ss = if fromEnum src == target' then M.singleton 0 (-1) else strategy src
    if odd (inverse (delete 0 src) - inverse (delete 0 target))
       then hPutStrLn stderr "no solution"
       else getArgs >>= \args -> if (elem "-g" args)
            then do
                putStrLn "digraph {"
                forM_ (nub $ M.keys ss) $ \s ->
                    putStrLn $ show s ++ " [shape=record" ++
                        (if s == fromEnum src
                         then ",style=filled,color=orange"
                         else if s == fromEnum target
                              then ",style=filled,color=orchid"
                              else "") ++ ",label=\""++label s++"\"];"
                forM_ (filter ((/=fromEnum src) . fst) $ M.toList ss) $ \(s,p) ->
                    putStrLn $ show p ++ "->" ++ show s ++ ";"
                putStrLn "}"
            else
                hPutStrLn stderr $ "minimum steps: " ++ show (pathLen (fromEnum target) ss)
  where
    label = intercalate "|" . map (('{':).(++"}") . intersperse '|' . concatMap show . map snd) . transpose . groupBy ((/=) `on` fst) . zip (cycle [1..3]) . (toEnum :: Int -> State)
    pathLen s m | s == fromEnum src = 0
                | otherwise = 1 + pathLen (fromJust $ M.lookup s m) m
    inverse = snd . foldr (\x (l,acc) -> (x:l,acc+length(filter(<x)l))) ([],0)

search :: (t -> (s, t)) -> (s -> State) -> ((s, t) -> [State] -> t) -> t -> M.Map Int Int -> M.Map Int Int
search extract transform merge open closed
    | isJust $ find (==target') suc' = closed'
    | otherwise = search extract transform merge (merge (h,open') suc) closed'
  where
    (h,open') = extract open
    suc = filter (not . flip M.member closed . fromEnum) . moves $ transform h
    suc' = map fromEnum suc
    closed' = M.union closed . M.fromList . zip suc' . repeat . fromEnum $ transform h

bfs :: State -> M.Map Int Int
bfs src = search extract id merge (Seq.singleton src) $ M.singleton (fromEnum src) (-1)
  where
    extract = (\(h Seq.:< t) -> (h, t)) . Seq.viewl
    merge (h,open') suc = open' Seq.>< Seq.fromList suc

astar :: State -> M.Map Int Int
astar src = search extract snd merge (SS.singleton (heuristic src, src)) $ M.singleton (fromEnum src) (-1)
  where
    extract = fromJust . SS.minView
    merge ((c,p),open') suc = SS.union open' $ SS.fromList $ map (\q -> (c - heuristic p + 1 + heuristic q, q)) suc
    heuristic = sum . map (\(x,y) -> distance x (y-1)) . filter ((/=0) . snd) . zip [0..]
      where
        distance p q = abs (x1-x2) + abs (y1-y2)
          where
            (x1,y1) = p `divMod` 3
            (x2,y2) = q `divMod` 3

main = do
    line <- getLine
##ifdef BFS
    solve bfs $ map (read . return) line
##else
    solve astar $ map (read . return) line
##endif
```
