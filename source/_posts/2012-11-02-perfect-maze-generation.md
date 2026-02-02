---
layout: post
title: 完美迷宮生成算法
author: MaskRay
tags: [algorithm, maze]
---

## Perfect maze

Perfect maze又稱standard maze，指沒有迴路，沒有不可達區域的迷宮。用圖論的語言來說，就是可以用spanning tree表示的迷宮，保證迷宮中任意兩個格子間都有唯一的路徑。本文旨在探討如何隨機生成棋盤狀perfect maze。迷宮格子的鄰居的定義採用[von Neumann neighborhood](http://en.wikipedia.org/wiki/Von_Neumann_neighborhood)，即水平豎直方向相鄰的四個格子。

## 變形Kruskal算法

說得通俗點，就是“隨機拆牆”。

<!-- more -->

一開始假設所有牆都在。如果當前不滿足“任意兩個格子間都有唯一路徑”，那麼就隨機選擇一堵牆，要求牆兩端的格子不連通，即它們對應的兩個頂點不在一個connected component裏。

把這堵牆拆掉，即在後牆兩端的格子對應的頂點間建立一條邊，此時connected component數就減少一了。不斷地找牆、拆牆，最後就能形成一個看上去挺隨機的迷宮。

### 僞代碼

    W = 所有牆
    T = {}
    number_of_components = N*N
    while number_of_components > 1
      (u, v) = W.pop
      if component(u) != component(v)
        T << (u, v)
        merge component(u) and component(v)

最後`T`集合就是spanning tree中的邊，即所需拆除的牆。

### 實現

考慮轉化爲圖論模型後我們需要的操作，只有兩個，即：

- 判斷兩個元素是否在一個集合中(兩個頂點是否在同一個connected component裏)，
- 合併兩個集合(合併兩個connected component)。

對於這個問圖，有經典的disjoin-set forest算法可以在近似線性的時間複雜度裏解決這個問題。

## Aldous-Broder算法

很簡單，隨機選擇一個格子作爲起點，每次隨機選擇一個鄰居格子走過去，如果目標格子不與當前格子連通則打通它們之間的牆。

### Haskell實現

以下代碼改編自某個Haskell的recursive backtracking算法：

```haskell
import Control.Applicative
import Control.Monad
import Control.Monad.Cont
import Control.Monad.ST
import Data.Array
import Data.Array.ST
import Data.STRef
import System.Random
import Debug.Trace

data Maze = Maze { rightWalls, belowWalls :: Array (Int, Int) Bool }

rand :: Random a => (a, a) -> STRef s StdGen -> ST s a
rand range g = do
  (a, g') <- liftM (randomR range) $ readSTRef g
  a <$ writeSTRef g g'

maze :: Int -> Int -> StdGen -> ST s Maze
maze w h gen = do
  let mk = newArray ((0,0), (w-1,h-1)) :: Bool -> ST s (STArray s (Int, Int) Bool)
  visited <- mk False
  right <- mk True
  bottom <- mk True
  gen <- newSTRef gen
  let
    walk 1 (x,y) = return ()
    walk c u@(x,y) = do
      writeArray visited u True
      let ns = [(x-1,y) | x > 0] ++ [(x+1,y) | x+1 < w] ++ [(x,y-1) | y > 0] ++ [(x,y+1) | y+1 < h] :: [(Int,Int)]
      i <- rand (0, length ns - 1) gen
      let v@(x', y') = ns !! i
          wall = if x == x' then bottom else right
          g = (min x x', min y y')
      hasWall <- readArray wall g
      seen <- readArray visited v
      if seen
         then
           walk c v
         else
           writeArray wall g False >> walk (c-1) v
  (,) <$> rand (0, w-1) gen <*> rand (0, h-1) gen >>= walk (w*h)
  Maze <$> freeze right <*> freeze bottom

  maze :: Int -> Int -> StdGen -> ST s Maze
  maze w h gen = do
    let mk = newArray ((0,0), (w-1,h-1)) :: Bool -> ST s (STArray s (Int, Int) Bool)
    visited <- mk False
    right <- mk True
    bottom <- mk True
    gen <- newSTRef gen
    let
      walk 1 gen (x,y) = return ()
      walk c gen u@(x,y) = do
        writeArray visited u True
        let ns = [(x-1,y) | x > 0] ++ [(x+1,y) | x+1 < w] ++ [(x,y-1) | y > 0] ++ [(x,y+1) | y+1 < h] :: [(Int,Int)]
        i <- rand (0, length ns - 1) gen
        let v@(x', y') = ns !! i
            wall = if x == x' then bottom else right
            g = (min x x', min y y')
        hasWall <- readArray wall g
        seen <- readArray visited v
        if seen
           then
             walk c gen v
           else
             writeArray wall g False >> walk (c-1) gen v
    (,) <$> rand (0, w-1) gen <*> rand (0, h-1) gen >>= walk (w*h) gen
    Maze <$> freeze right <*> freeze bottom

printMaze :: Maze -> IO ()
printMaze (Maze right bottom) = do
  putStrLn $ concat (replicate (maxX + 1) "._") ++ "."
  forM_ [0..maxY] $ \y -> do
    putStr "|"
    forM_ [0..maxX] $ \x -> do
      putStr $ if bottom ! (x, y) then "_" else " "
      putStr $ if right ! (x, y) then "|" else "."
    putChar '\n'
  where
    (maxX, maxY) = snd $ bounds right

main = getStdGen >>= stToIO . maze 12 13 >>= printMaze
```

該算法的優點是能等概率地生成每一棵spanning tree。

## Recursive backtracking算法

上述隨機拆牆使用到的Kruskal算法是求minimum spanning tree問題最廣爲人知的兩個算法之一，另一個就是Prim算法，那麼後者能不能解決這個問題呢？有種類似變形Prim的算法。

和變形Kruskal算法一樣，一開始假設所有牆都存在，僞代碼如下：

    S = {(0,0)}
    T = {}
    while visited.size != n * n
      u = S.pop
      for v in neighbors(u).random_shuffle
        if component(v) != component(u)
          T << (u, v)
          S << v

最後`T`集合就是spanning tree中的邊，即所需拆除的牆。

如果經典的Prim算法實現看作是維護了一個priority queue來維護connected component中的點，那麼這裏`S`可以用比priority queue簡單的stack或queue來實現。原因是我們求的不是minimum spanning tree，所有頂點的權值可以看作是相同的；所有邊的權值也可以看作是相同的。

如果用stack代替priority queue，那麼我們就甚至無需顯示維護一個stack了，直接使用程序函數調用時系統維護的棧。

- 隨機選擇一個頂點作爲起點
- 對於當前點的每一個鄰點，如果這個鄰點和當前點不在同一個connected component，
  那麼就在它們之間建立一條邊，即把它們之間的牆拆掉，並且遞歸調用該步驟，當前點設置爲這個鄰點。

### Haskell實現

下面實現來自[http://megasnippets.com/source-codes/haskell/maze_generation]()

```haskell
import Control.Monad
import Control.Monad.ST
import Data.Array
import Data.Array.ST
import Data.STRef
import System.Random

rand :: Random a => (a, a) -> STRef s StdGen -> ST s a
rand range gen = do
    (a, g) <- liftM (randomR range) $ readSTRef gen
    gen `writeSTRef` g
    return a

data Maze = Maze {rightWalls, belowWalls :: Array (Int, Int) Bool}

maze :: Int -> Int -> StdGen -> ST s Maze
maze width height gen = do
    visited <- mazeArray False
    rWalls <- mazeArray True
    bWalls <- mazeArray True
    gen <- newSTRef gen
    liftM2 (,) (rand (0, maxX) gen) (rand (0, maxY) gen) >>=
        visit gen visited rWalls bWalls
    liftM2 Maze (freeze rWalls) (freeze bWalls)
  where visit gen visited rWalls bWalls here = do
            writeArray visited here True
            let ns = neighbors here
            i <- rand (0, length ns - 1) gen
            forM_ (ns !! i : take i ns ++ drop (i + 1) ns) $ \there -> do
                seen <- readArray visited there
                unless seen $ do
                    removeWall here there
                    visit gen visited rWalls bWalls there
          where removeWall (x1, y1) (x2, y2) = writeArray
                    (if x1 == x2 then bWalls else rWalls)
                    (min x1 x2, min y1 y2)
                    False

        neighbors (x, y) =
            (if x == 0    then [] else [(x - 1, y    )]) ++
            (if x == maxX then [] else [(x + 1, y    )]) ++
            (if y == 0    then [] else [(x,     y - 1)]) ++
            (if y == maxY then [] else [(x,     y + 1)])

        maxX = width - 1
        maxY = height - 1

        mazeArray = newArray ((0, 0), (maxX, maxY))
            :: Bool -> ST s (STArray s (Int, Int) Bool)

printMaze :: Maze -> IO ()
printMaze (Maze rWalls bWalls) = do
    putStrLn $ '+' : (concat $ replicate (maxX + 1) "---+")
    forM_ [0 .. maxY] $ \y -> do
        putStr "|"
        forM_ [0 .. maxX] $ \x -> do
            putStr "   "
            putStr $ if rWalls ! (x, y) then "|" else " "
        putStrLn ""
        forM_ [0 .. maxX] $ \x -> do
            putStr "+"
            putStr $ if bWalls ! (x, y) then "---" else "   "
        putStrLn "+"
  where maxX = fst (snd $ bounds rWalls)
        maxY = snd (snd $ bounds rWalls)

main = getStdGen >>= stToIO . maze 11 8 >>= printMaze
```

### C實現

```c
##include <stdbool.h>
##include <stdio.h>
##include <stdlib.h>
##include <string.h>

##define N 92
bool R[N][N], D[N][N], v[N][N];
int m, n;

void dfs(int r, int c)
{
  int d = rand() % 4, dd = rand()%2 ? 1 : 3;
  v[r][c] = true;
  for (int i = 0; i < 4; i++) {
    int rr = r + (int[]){-1,0,1,0}[d],
        cc = c + (int[]){0,-1,0,1}[d];
    if ((unsigned)rr < m && (unsigned)cc < n && ! v[rr][cc]) {
      if (d % 2)
        R[r][c - (d == 1)] = true;
      else
        D[r - (d == 0)][c] = true;
      dfs(rr, cc);
    }
    d = (d + dd) % 4;
  }
}

int main()
{
  scanf("%d%d", &m, &n);
  dfs(0, 0);
  for (int c = 0; c < n; c++)
    printf("._");
  printf(".\n");
  for (int r = 0; r < m; r++) {
    printf("|");
    for (int c = 0; c < n; c++) {
      putchar(D[r][c] ? ' ' : '_');
      putchar(R[r][c] ? '.' : '|');
    }
    printf("\n");
  }
}
```

## Eller算法(以行爲單位的隨機拆牆)

對於直接輸出的算法來說，上述算法額外數據結構的空間複雜度都是和迷宮大小同階的，其實我們只需要和迷宮寬(或高)同階的額外空間就夠了。

這個算法我所瞭解的出處是某屆IOCCC的一個作品。<https://tromp.github.io/maze.html> (was: <http://homepages.cwi.nl/~tromp/maze.html>)給出了該作品的描述，後文我給出的實現也借鑑了鏈接中給出的代碼。

先考慮第一行，隨機拆掉一些格子的右牆。維護若干雙線循環鏈表，每個鏈表表示這行連通的格子。所以拆牆後，如果第一行有`x`堵牆，那麼就有`x-1`條鏈表。之後要決定哪些格子和下面一行相連。

    ._._._._._._.
    |_. |_._. | |

在上圖中，有2堵牆，有三個格子被決定和下面一行相連。最右邊的格子沒有和左邊相連，但是隻要它的下牆被拆除，那麼它最終還是會和所有格子連通。因此這堵下牆的拆除是必需的。

接下來要處理第二行了，依舊是決定哪些格子的右牆須要被拆除。但要注意如果格子`i`的右牆即格子`i`和`i+1`之間的牆被拆除了，那麼必須保證在上面各行格子`i`和`i+1`不連通。到這裏爲什麼要使用鏈表維護連通性就明顯了，我們需要快速判斷格子`i`和`i+1`在當前行之前有沒有連通來決定它們倆之間的牆能否被拆除，用union-find算法當然可行，但鏈表更方便。

在決定完拆除第二行一些格子的右牆後，考慮哪些格子需要和下一行(即第三行)相連，假設我們的決定是這樣的：

    ._._._._._._.
    |_. |_._. | |
    | |_._._. . |

這裏注意到第二行最左邊的格子，它的右牆沒有被拆除，所以它的下牆必須被拆除以保證最終它會和其他格子連通。

然後考慮第三行哪些格子的右牆要被拆除：

    ._._._._._._.
    |_. |_._. | |
    | |_._._. . |
    |_._. | |_| |

注意上圖第三行最後兩個格子，它們之間有牆，這堵牆不能被拆除，理由是第二行時它們已經連通了。

之後幾行如法炮製：

    ._._._._._._.
    |_. |_._. | |
    | |_._._. . |
    |_._. | |_| |
    |_._._. | | |
    | | ._._. | |
    | |_._._| ._|

然後開始處理最後一行。最後一行的任務有點特殊，因爲下牆不能被拆除(迷宮邊界)，需要拆除若干右牆使得最後一行的幾個連通分支並在一起。發現不得不寫成這樣：

    ._._._._._._.
    |_. |_._. | |
    | |_._._. . |
    |_._. | |_| |
    |_._._. | | |
    | | ._._. | |
    | |_._._| ._|
    |_._._._._._|

### Disjoin-set forest的刪除元素操作

之前的例子中我們發現每一行我們做了兩步決策，拆右牆和拆下牆。有些時候右牆不能被拆除，是因爲牆兩邊的格子在之前行已經被連通了。拆右牆就相當於合併兩個集合。

右牆決策產生之後，有些下牆是必須拆的，即當它自成一個集合時。如果不拆下牆，那麼在下一行，這個格子就和其他不連通了，所以得自成一個集合。

推敲我們需要實現的操作：並、查、從集合中刪除一個元素，如果只有連兩個操作，那麼很簡單，直接套用 disjoin-set forest 就好了。難點就在於如何支持“從集合中刪除一個元素”這一操作。刪除其實也是更新，引入時間因素後，就可以用一條更新記錄來代替一個刪除操作，但要注意把被刪除的記錄置爲無效狀態。刪除一個元素，可以看作：引入一個新頂點，改變格子指向頂點的映射。

![刪除元素](/static/maze-generation.png)

如圖，在格子0刪除下牆前沒有虛線邊以及虛線頂點3。刪除格子0的下牆後，需要讓格子0對應的元素(頂點0)自成一個集合。做法是把藍線(格子指向頂點的映射)換成紅色虛線。儘管有頂點1和2在disjoin-set forest中已經指向頂點3，但是我們做的這個刪除操作不會影響到它們。

### 用linked list實現disjoin sets

上一節通過改造disjoin-set forest使其支持刪除元素的操作，但是改造後空間複雜度就沒有吸引力了，因爲所有產生的頂點數目可能是迷宮大小級別。

仔細研究我們用到的操作：

- 合併：只有相鄰格子所對應集合會合併；
- 比較兩個格子是否在一個集合中：只有相鄰格子會被比較。

用一個鏈表表示一個connected component，並且讓鏈表的元素按編號排序。那麼對於格子`i`，查找其對應的鏈表中節點的後繼是否對應格子`i+1`就能知道格子`i`和`i+1`是否在同一個connected component。對於合併操作，以及讓一個格子自成一個集合的操作，鏈表也都能出色地完成任務。

#### 第一行

再來看：

    ._._._._._._.
    |_. |_._. | |

行間有兩堵牆，需要三個鏈表：

    L[0] = 1, L[1] = 0; R[0] = 1, R[1] = 0
    L[2] = 4, L[3] = 2, L[4] = 3; R[2] = 3, R[3] = 4, R[4] = 2
    L[5] = 5; R[5] = 5

#### 第二行

現在考慮第二行，上面的鏈表狀態代表的迷宮如下：

    ._._._._._._.
    |_. |_._. | |
    | | | | | | |

決定拆除一些牆後：

    ._._._._._._.
    |_. |_._. | |
    | | . . . . |

相應地，鏈表狀態變成：

    L[0] = 0; R[0] = 0
    L[1] = 5, L[2] = 1, L[3] = 2, L[4] = 3, L[5] = 4; R[1] = 2, R[2] = 3, R[3] = 4, R[4] = 5, R[5] = 1

接着決定拆掉一些下牆：

    ._._._._._._.
    |_. |_._. | |
    | |_._._. . |

那麼格子1 、2、3都得自成一個集合，鏈表狀態變成：

    L[0] = 0; R[0] = 0
    L[1] = 1; R[1] = 2
    L[2] = 2; R[2] = 2
    L[3] = 3; R[3] = 3
    L[4] = 5, L[5] = 4; R[4] = 5, R[5] = 4

#### 第三行

目前第三行尚未考慮拆牆，由上面的鏈表狀態得出的連通信息：格子0、1、2、3互不連通，格子4、5連通，恰好能反應下圖：

    ._._._._._._.
    |_. |_._. | |
    | |_._._. . |
    | | | | | | |

### OCaml實現

```ocaml
let eller m n =
  let l = Array.create (n+1) 0 in
  let r = Array.create (n+1) 0 in
  for i = 0 to n-1 do
    print_string "._";
    l.(i) <- i;
    r.(i) <- i
  done;
  l.(n) <- n-1;
  print_string ".\n|";
  for y = 0 to m-2 do
    for x = 0 to n-1 do
      let w = l.(x+1) in
      let pat1 =
        if x <> w && Random.int 3 <> 0 then begin
          r.(w) <- r.(x);
          l.(r.(w)) <- w;
          r.(x) <- x+1;
          l.(x+1) <- x;
          '.'
        end else
          '|'
        in
      let pat0 =
        if x <> l.(x) && Random.int 3 <> 0 then begin
          l.(r.(x)) <- l.(x);
          r.(l.(x)) <- r.(x);
          l.(x) <- x;
          r.(x) <- x;
          '_'
        end else
          ' '
        in
      print_char pat0;
      print_char pat1
    done;
    print_string "\n|"
  done;

  for x = 0 to n-1 do
    let w = l.(x+1) in
    let pat1 =
      if x <> w && (x == l.(x) || Random.int 3 <> 0) then begin
        r.(w) <- r.(x);
        l.(r.(w)) <- w;
        r.(x) <- x+1;
        l.(x+1) <- x;
        '.'
      end else
        '|'
      in
        l.(r.(x)) <- l.(x);
        r.(l.(x)) <- r.(x);
        l.(x) <- x;
        r.(x) <- x;
    print_char '_';
    print_char pat1
  done;;

Scanf.scanf "%d %d" eller
```

### C實現

```c
##include <stdio.h>
##include <stdlib.h>

##define N 92
int L[N], R[N];

int main()
{
  char pat[3] = {};
  int m, n;
  scanf("%d%d", &m, &n);
  for (int i = 0; i < n; i++) {
    printf("._");
    L[i] = R[i] = i;
  }
  L[n] = n - 1;

  printf(".\n|");
  while (m--) {
    for (int i = 0; i < n; i++) {
      int j = L[i + 1];
      if (i != j && rand() % 3) {
        L[R[j] = R[i]] = j;
        L[R[i] = i + 1] = i;
        pat[1] = '.';
      } else
        pat[1] = '|';
      if (i != L[i] && rand() % 3) {
        L[R[i]] = L[i];
        R[L[i]] = R[i];
        L[i] = R[i] = i;
        pat[0] = '_';
      } else
        pat[0] = ' ';
      printf(pat);
    }
    printf("\n|");
  }

  pat[0] = '_';
  for (int i = 0; i < n; i++) {
    int j = L[i + 1];
    if (i != j && (i == L[i] || rand() % 3)) {
      L[R[j] = R[i]] = j;
      L[R[i] = i + 1] = i;
      pat[1] = '.';
    } else
      pat[1] = '|';
    L[R[i]] = L[i];
    R[L[i]] = R[i];
    L[i] = R[i] = i;
    printf(pat);
  }
}
```

## 推薦閱讀

[Maze Generation: Eller's Algorithm](//weblog.jamisbuck.org/2010/12/29/maze-generation-eller-s-algorithm)給出了關於Eller算法的詳細描述。

[Maze Classification](http://www.astrolog.org/labyrnth/algrithm.htm)給出了非常詳細的迷宮生成算法和解法列表。
