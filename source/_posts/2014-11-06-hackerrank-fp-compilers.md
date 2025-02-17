---
layout: post
title: Hackerrank FP Compilers题目
author: MaskRay
tags: [hackerrank,fp,ocaml]
---

## Generate String from Regex

```ocaml
let is_lower c =
  'a' <= c && c <= 'z'

type ast =
  | Lit of char
  | Star of ast
  | Cat of ast*ast
  | Or of ast*ast

module Parser = struct
  type state = Ret of ast | Cont of char list * ast list

  let parse re =
    let isp_tbl = Hashtbl.create 10 in
    let icp_tbl = Hashtbl.create 10 in
    List.iter (fun (c,x) -> Hashtbl.replace isp_tbl c x)
      ['\x00',0; '(',1; '|',3; '%',5; '*',7; ')',7];
    List.iter (fun (c,x) -> Hashtbl.replace icp_tbl c x)
      ['\x00',0; ')',1; '|',2; '%',4; '*',6; '(',6];

    let n = String.length re in
    let rec go ops vs i =
      let implicit_cat = 0 < i && i < n && (re.[i] = '(' || is_lower re.[i]) &&
        (re.[i-1] <> '(' && re.[i-1] <> '|') in
      let incoming implicit_cat ops vs =
        let ic = if implicit_cat then '%' else if i = n then '\x00' else re.[i] in
        if is_lower ic then
          Cont (ops, Lit ic::vs)
        else
          let icp = Hashtbl.find icp_tbl ic in
          let rec pop_go (op::ops' as ops) vs =
            if Hashtbl.find isp_tbl op > icp then
              match op with
              | '*' ->
                  let x::vs' = vs in
                  pop_go ops' (Star x::vs')
              | '%' ->
                  let y::x::vs' = vs in
                  pop_go ops' (Cat (x,y)::vs')
              | '|' ->
                  let y::x::vs' = vs in
                  pop_go ops' (Or (x,y)::vs')
              | c ->
                  Printf.printf "+ c: %c %c\n" op ic;
                  ops ,vs
            else
              ops, vs
          in
          let (op::ops' as ops), vs = pop_go ops vs in
          if Hashtbl.find isp_tbl op = icp then (
            if ic = '\x00' then
              Ret (List.hd vs)
            else
              Cont (ops', vs)
          ) else
            Cont (ic::ops, vs)
      in
      match incoming implicit_cat ops vs with
      | Ret _ as ret -> ret
      | Cont (ops,vs) ->
          if not implicit_cat then
            go ops vs (i+1)
          else
            let Cont (ops,vs) = incoming false ops vs in
            go ops vs (i+1)
    in
    let Ret ret = go ['\x00'] [] 0 in
    ret
end

let generate len =
  let rec go = function
    | Lit lit ->
        let a = Array.make (len+1) "#" in
        a.(1) <- String.make 1 lit;
        a
    | Star x ->
        let b = go x in
        let a = Array.make (len+1) "#" in
        a.(0) <- "";
        for i = 1 to len do
          let rec fill j =
            if j > 0 then
              if b.(j) <> "#" && a.(i-j) <> "#" then
                a.(i) <- a.(i-j) ^ b.(j)
              else
                fill (j-1)
          in
          fill i
        done;
        a
    | Cat (x,y) ->
        let b = go x and c = go y in
        let a = Array.make (len+1) "#" in
        for i = 0 to len do
          for j = 0 to len-i do
            if a.(i+j) = "#" && b.(i) <> "#" && c.(j) <> "#" then
              a.(i+j) <- b.(i) ^ c.(j)
          done
        done;
        a
    | Or (x,y) ->
        let b = go x in
        let c = go y in
        Array.init (len+1) (fun i -> if b.(i) <> "#" then b.(i) else c.(i))
  in
  go

let () =
  let len = read_int () in
  let re = read_line () in
  let ast = Parser.parse re in
  let a = generate len ast in
  let s = a.(len) in
  print_endline (if s <> "#" then s else "NIL")
```

## While Language

生成AST後進入求值階段，可以用state monad處理求值部分。

```ocaml
module Str = struct
  let implode l =
    let s = String.make (List.length l) ' ' in
    let rec go i = function
      | [] -> s
      | h::t -> s.[i] <- h; go (i+1) t
    in
    go 0 l

  let explode s =
    let rec go acc i =
      if i = String.length s then
        List.rev acc
      else
        go (s.[i]::acc) (i+1)
    in
    go [] 0
end

module LazyList = struct
  type 'a node = Nil  | Cons of 'a * 'a t and 'a t = 'a node Lazy.t
  let empty = lazy Nil
  let singleton x = lazy (Cons (x, empty))
  let cons h t = lazy (Cons (h, t))
  let force = Lazy.force
  let rec map f l = lazy (
    match force l with
    | Nil -> Nil
    | Cons (h, t) -> Cons (f h, map f t)
    )
  let rec append l1 l2 = lazy (
    match force l1 with
    | Nil -> force l2
    | Cons (h, t) -> Cons (h, append t l2)
    )
  let rec concat ll = lazy (
    match force ll with
    | Nil -> Nil
    | Cons (h, t) -> append h (concat t) |> force
    )
  let is_empty l =
    match force l with
    | Nil -> true
    | Cons _ -> false
end

module ParserCombinators = struct
  type input = { s: string; pos: int }
  type 'a t = input -> ('a * input) LazyList.node Lazy.t

  let unit x s = LazyList.singleton (x, s)
  let zero = unit []
  let (>>=) (type a) (type b) (x : a t) (f : a -> b t) s = LazyList.map (fun (a,s') -> f a s') (x s) |> LazyList.concat
  let (>>) x y = x >>= fun _ -> y
  let (<<) x y = x >>= fun b -> y >> unit b
  let (<|>) x y s = let r = x s in if LazyList.is_empty r then y s else r
  let fail s = LazyList.empty
  let (<$>) f x = x >>= fun b -> unit (f b)
  let (<$) b x = x >> unit b
  let (<*>) f x = f >>= fun g -> x >>= fun b -> unit (g b)
  let (<**>) x f = x >>= fun b -> f >>= fun g -> unit (g b)

  let rec many x =
    let go = x >>= fun b ->
      many x >>= fun bs ->
      unit (b::bs)
    in
    go <|> zero

  let many1 x =
    x >>= fun b ->
    many x >>= fun bs ->
    unit (b::bs)

  let next s =
    if s.pos = String.length s.s then
      None
    else
      Some (s.s.[s.pos], { s with pos = s.pos + 1 })

  let pred f = (fun s -> match next s with
    | None -> LazyList.empty
    | Some x -> LazyList.singleton x) >>= fun b ->
        if f b then unit b else fail

  let char c = pred (fun c' -> c = c')
  let str s =
    let rec go = function
      | [] -> zero
      | h::t -> char h >>= fun b -> go t >>= fun bs -> unit (b::bs)
    in
    Str.explode s |> go
  let digit = pred (fun c -> '0' <= c && c <= '9')
  let lower = pred (fun c -> 'a' <= c && c <= 'z')
  let drop x = x >> zero
  let space = pred (fun c -> c = ' ' || c = '\t' || c = '\r' || c = '\n')
  let token x = x << drop (many space)
  let char_ c = char c |> token
  let str_ s = str s |> token

  let sep_by x sep =
    let go = x >>= fun b ->
      many (drop sep >> x) >>= fun bs ->
      unit (b::bs)
    in
    go <|> zero

  let chainl1 x op =
    let rec go a =
      (op >>= fun f ->
      x >>= fun b ->
      go (f a b)) <|> unit a
    in
    x >>= go

  let parse x s =
    match LazyList.force ((drop (many space) >> x) { s; pos = 0 }) with
    | LazyList.Nil -> None
    | LazyList.Cons ((x, _), _) -> Some x
end

type aexpr =
  | Num of int64
  | Var of string
  | Add of aexpr * aexpr
  | Sub of aexpr * aexpr
  | Mul of aexpr * aexpr
  | Div of aexpr * aexpr

type bexpr =
  | Bool of bool
  | Not of bexpr
  | And of bexpr * bexpr
  | Or of bexpr * bexpr
  | Lt of aexpr * aexpr
  | Gt of aexpr * aexpr

type stmt =
  | Chain of stmt list
  | Assign of string * aexpr
  | If of bexpr * stmt * stmt
  | While of bexpr * stmt

module Parser = struct
  include ParserCombinators

  let add x y = Add (x,y)
  let sub x y = Sub (x,y)
  let mul x y = Mul (x,y)
  let div x y = Div (x,y)
  let num l = Num (Str.implode l |> Int64.of_string)
  let var l = Var (Str.implode l)
  let chain xs = Chain xs
  let if_ b t e = If (b,t,e)
  let while_ b t = While (b,t)
  let assign x y = Assign (x,y)
  let and_ x y = And (x,y)
  let or_ x y = Or (x,y)
  let lt x y = Lt (x,y)
  let gt x y = Gt (x,y)

  let rec aexpr =
    let rec e0 = lazy ((num <$> token (many1 digit)) <|> (var <$> token (many1 lower)) <|> (fun s -> (char_ '(' >> Lazy.force e2 << char_ ')') s))
    and e1 = lazy (chainl1 (Lazy.force e0) ((mul <$ char_ '*') <|> (div <$ char_ '/')))
    and e2 = lazy (chainl1 (Lazy.force e1) ((add <$ char_ '+') <|> (sub <$ char_ '-')))
    in
    Lazy.force e2

  let rec bexpr =
    let rec e0 = lazy ((Bool false <$ str_ "false") <|> (Bool true <$ str_ "true") <|>
      (fun s -> (char_ '(' >> Lazy.force e3 << char_ ')') s)
      )
    and e1 = lazy ((aexpr <**> ((lt <$ char_ '<') <|> (gt <$ char_ '>')) <*> aexpr) <|> Lazy.force e0)
    and e2 = lazy (chainl1 (Lazy.force e1) ((and_ <$ str_ "and")))
    and e3 = lazy (chainl1 (Lazy.force e2) ((or_ <$ str_ "or")))
    in
    Lazy.force e3

  let rec stmt =
    let rec sassign = lazy (Str.implode <$> token (many1 lower) <**> (assign <$ str_ ":=") <*> aexpr)
    and sif = lazy ((if_ <$ str_ "if") <*> bexpr << str_ "then" <*> Lazy.force sb << str_ "else" <*> Lazy.force sb)
    and sb = lazy (char_ '{' >> (fun s -> Lazy.force s1 s) << char_ '}')
    and swhile = lazy ((while_ <$ str_ "while") <*> bexpr << str_ "do" <*> Lazy.force sb)
    and s0 = lazy (Lazy.force sif <|> Lazy.force swhile <|> Lazy.force sb <|> Lazy.force sassign)
    and s1 = lazy (chain <$> sep_by (Lazy.force s0) (char_ ';'))
    in
    Lazy.force s1
end

module Eval = struct
  type store = (string, int64) Hashtbl.t
  type 'a t = store -> 'a

  let unit a s = a
  let (>>=) x f s = f (x s) s
  let (>>) x y = x >>= fun _ -> y
  let (<$>) f x s = x s |> f
  let rec map_ f = function
    | [] -> unit ()
    | h::t -> f h >> map_ f t
  let liftM2 f x y s =
    let x' = x s in
    let y' = y s in
    f x' y'

  let rec eval_a = function
    | Num n -> unit n
    | Var v -> fun tbl -> Hashtbl.find tbl v
    | Add (x,y) -> liftM2 Int64.add (eval_a x) (eval_a y)
    | Sub (x,y) -> liftM2 Int64.sub (eval_a x) (eval_a y)
    | Mul (x,y) -> liftM2 Int64.mul (eval_a x) (eval_a y)
    | Div (x,y) -> liftM2 Int64.div (eval_a x) (eval_a y)

  let rec eval_b = function
    | Bool b -> unit b
    | Not x -> not <$> eval_b x
    | And (x,y) -> liftM2 (&&) (eval_b x) (eval_b y)
    | Or (x,y) -> liftM2 (||) (eval_b x) (eval_b y)
    | Lt (x,y) -> liftM2 (<) (eval_a x) (eval_a y)
    | Gt (x,y) -> liftM2 (>) (eval_a x) (eval_a y)

  let rec eval_s = function
    | Assign (v,x) -> fun tbl -> Hashtbl.replace tbl v (eval_a x tbl)
    | Chain xs -> map_ eval_s xs
    | If (b,t,e) -> eval_b b >>= fun b' -> eval_s (if b' then t else e)
    | While (b,t) as w -> eval_b b >>= fun b' -> if b' then eval_s t >> eval_s w else unit ()

  let interpret ast =
    let tbl = Hashtbl.create 13 in
    eval_s ast tbl;
    Hashtbl.fold (fun k v xs -> (k,v)::xs) tbl [] |> List.sort compare |> List.iter (fun (k,v) ->
      Printf.printf "%s %Ld\n" k v
    )
end

let read _ =
  let rec go acc =
    try
      let s = read_line () in
      go (s::acc)
    with End_of_file ->
      String.concat "\n" (List.rev acc)
  in
  go []

let () =
  match Parser.parse Parser.stmt (read 0) with
  | None -> ()
  | Some ast -> Eval.interpret ast
```

## Down With Abstractions

https://www.hackerrank.com/challenges/down-with-abstractions

要求把lambda calculus轉化爲SKIBC組合子，[Wikipedia](http://en.wikipedia.org/wiki/Combinatory_logic#Completeness_of_the_S-K_basis)給出了一個算法進行這種轉換。我嘗試用De Bruijn index表示的lambda calculus實現。算法中有一個子程序是判斷某個子calculus內是否有一個變量爲free的，如果用樸素的實現方式(遍歷取出所有變量，一一判斷是否存在free的)，會增大時間複雜度。如果calculus的形態不發生改變，那麼可以Inversion of Control：對於每個lambda abstraction找自由變量的要求，轉換爲把所有變量涉及的lambda abstraction標記出來。考慮到算子的形態會發生改變，我最終採用對於每個子calculus設置一個persistent heap來表示。

下面是實現代碼，其中還結合state monad和lazy list實現了一個parser monad，用parsing expression grammar的方式來處理。
mutually recursive函數，目前理解得尚不清楚，用了一個比較麻煩的方式tying-the-knot：各function都表示爲lazy的，這樣所有的引用都需要表示爲`Lazy.force referenced`的形式，另外對於引用鏈上所有back reference的地方都要使用eta expansion。如果不這麼做會報運行時錯誤。

```ocaml
module Str = struct
  let implode l =
    let s = String.make (List.length l) ' ' in
    let rec go i = function
      | [] -> s
      | h::t -> s.[i] <- h; go (i+1) t
    in
    go 0 l

  let explode s =
    let rec go acc i =
      if i = String.length s then
        List.rev acc
      else
        go (s.[i]::acc) (i+1)
    in
    go [] 0
end

module LazyList = struct
  type 'a node = Nil  | Cons of 'a * 'a t and 'a t = 'a node Lazy.t
  let empty = lazy Nil
  let singleton x = lazy (Cons (x, empty))
  let cons h t = lazy (Cons (h, t))
  let force = Lazy.force
  let rec map f l = lazy (
    match force l with
    | Nil -> Nil
    | Cons (h, t) -> Cons (f h, map f t)
    )
  let rec append l1 l2 = lazy (
    match force l1 with
    | Nil -> force l2
    | Cons (h, t) -> Cons (h, append t l2)
    )
  let rec concat ll = lazy (
    match force ll with
    | Nil -> Nil
    | Cons (h, t) -> append h (concat t) |> force
    )
  let is_empty l =
    match force l with
    | Nil -> true
    | Cons _ -> false
end

module ParserCombinators(S : sig type store end) = struct
  type store = S.store
  type input = { s: string; pos: int; store: store }
  type 'a t = input -> ('a * input) LazyList.node Lazy.t

  let unit x s = LazyList.singleton (x, s)
  let zero = unit []
  let (>>=) (type a) (type b) (x : a t) (f : a -> b t) s = LazyList.map (fun (a,s') -> f a s') (x s) |> LazyList.concat
  let (>>) x y = x >>= fun _ -> y
  let (<<) x y = x >>= fun b -> y >> unit b
  let (<|>) x y s = let r = x s in if LazyList.is_empty r then y s else r
  let fail s = LazyList.empty
  let (<$>) f x = x >>= fun b -> unit (f b)
  let (<$) b x = x >> unit b
  let (<*>) f x = f >>= fun g -> x >>= fun b -> unit (g b)
  let (<**>) x f = x >>= fun b -> f >>= fun g -> unit (g b)

  let rec many x =
    let go = x >>= fun b ->
      many x >>= fun bs ->
      unit (b::bs)
    in
    go <|> zero

  let many1 x =
    x >>= fun b ->
    many x >>= fun bs ->
    unit (b::bs)

  let next s =
    if s.pos = String.length s.s then
      None
    else
      Some (s.s.[s.pos], { s with pos = s.pos + 1 })

  let pred f = (fun s -> match next s with
    | None -> LazyList.empty
    | Some x -> LazyList.singleton x) >>= fun b ->
        if f b then unit b else fail

  let char c = pred (fun c' -> c = c')
  let str s =
    let rec go = function
      | [] -> zero
      | h::t -> char h >>= fun b -> go t >>= fun bs -> unit (b::bs)
    in
    Str.explode s |> go
  let drop x = x >> zero
  let space = pred (fun c -> c = ' ' || c = '\t' || c = '\r' || c = '\n')
  let ident = Str.implode <$> many1 (pred (fun c -> 'a' <= c && c <= 'z' || 'A' <= c && c <= 'Z' || '0' <= c && c <= '9' || c = '_'))

  let token x = x << drop (many space)
  let char_ c = char c |> token
  let ident_ = token ident

  let (>>::) x y =
    x >>= fun a ->
    y >>= fun b ->
    unit (a::b)

  let sep_by x sep =
    let go = x >>= fun b ->
      many (drop sep >> x) >>= fun bs ->
      unit (b::bs)
    in
    go <|> zero

  let chainl1 x op =
    let rec go a =
      (op >>= fun f ->
      x >>= fun b ->
      go (f a b)) <|> unit a
    in
    x >>= go

  let rec chainr1 x op =
    let go a =
      (op >>= fun f ->
      chainr1 x op >>= fun b ->
      unit (f a b)) <|> unit a
    in
    x >>= go

  let parse x store s =
    match LazyList.force ((drop (many space) >> x) { s; pos = 0; store }) with
    | LazyList.Nil -> None
    | LazyList.Cons ((x, _), _) -> Some x
end

module LeftistHeap = struct
  type t = E | N of int * int * int * t * t

  let empty = E

  let is_empty = function
    | E -> true
    | _ -> false

  let singleton x = N (x,0,0,E,E)

  let rank = function
    | E -> -1
    | N (_,_,rk,_,_) -> rk

  let make x l r =
    if rank l >= rank r then
      N (x,0,rank r+1,l,r)
    else
      N (x,0,rank l+1,r,l)

  let add i = function
    | E -> E
    | N (x,d,rk,l,r) -> N (x+i,d+i,rk,l,r)

  let rec merge h1 h2 = match h1, h2 with
    | E, _ -> h2
    | _, E -> h1
    | N (x,d1,_,l1,r1), N (y,d2,_,l2,r2) ->
        if x < y then
          make x (add d1 l1) (merge (add d1 r1) h2)
        else
          make y (add d2 l2) (merge (add d2 r2) h1)

  let top = function
    | E -> invalid_arg "empty"
    | N (x,_,_,_,_) -> x

  let pop = function
    | E -> invalid_arg "empty"
    | N (_,d,_,l,r) -> merge (add d l) (add d r)

  let push h x = merge h (N (x,0,0,E,E))
end

type term =
  | Var of LeftistHeap.t * int
  | Abs of LeftistHeap.t * term
  | App of LeftistHeap.t * term * term
  | S
  | K
  | I
  | B
  | C

module List = struct
  include List

  let fold_left1 f xs =
    List.fold_left f (List.hd xs) (List.tl xs)

  let rec drop c = function
    | [] -> []
    | _::xs' as xs ->
        if c = 0 then
          xs
        else
          drop (c-1) xs'
end

let get_dh = function
  | Var (dh,_) -> dh
  | App (dh,_,_) -> dh
  | Abs (dh,_) -> dh
  | _ -> LeftistHeap.empty

let rec cut dh =
  if LeftistHeap.is_empty dh then
    dh
  else if LeftistHeap.top dh < 0 then
    cut (LeftistHeap.pop dh)
  else
    dh

(*let rec free l = function*)
  (*| Var (dh, c) -> c = l*)
  (*| App (dh,x,y) -> free l x || free l y*)
  (*| Abs (dh,x) -> free (l+1) x*)
  (*| _ -> false*)

let free x = match x with
  | Var (_,_)
  | App (_,_,_)
  | Abs (_,_) ->
      let dh = get_dh x in
      not (LeftistHeap.is_empty dh) && LeftistHeap.top dh = 0
  | _ -> false

let rec shift = function
  | Var (dh,c) as v -> Var (LeftistHeap.add (-1) dh |> cut, c-1)
  | App (dh,x,y) -> App (LeftistHeap.add (-1) dh |> cut, shift x, shift y)
  | Abs _ -> assert false
  | c -> c

let var x = Var (LeftistHeap.singleton x, x)

let abstract x = Abs (LeftistHeap.add (-1) (get_dh x) |> cut, x)

let app x y = App (LeftistHeap.merge (get_dh x) (get_dh y), x, y)

let rec nest lv t =
  if lv = 0 then
    t
  else
    nest (lv-1) (abstract t)

let print t =
  let rec go r = function
  | Var (_,c) ->
      Printf.printf "%d" c
  | Abs _ ->
      assert false
  | App (_,x,y) ->
      if r then print_char '(';
      go false x;
      go true y;
      if r then print_char ')'
  | S -> print_char 'S'
  | K -> print_char 'K'
  | I -> print_char 'I'
  | B -> print_char 'B'
  | C -> print_char 'C'
  in
  go false t

let rec tr =
  let e = LeftistHeap.empty in function
  | App (_,x,y) -> app (tr x) (tr y)
  | Abs (_, Var (_, 0)) -> I
  | Abs (_, App (_, x, Var (_, 0))) when not (free x) ->
      tr x |> shift
  | Abs (_, x) ->
      if not (free x) then
        app K (tr x |> shift)
      else (
        match x with
        | Abs (_, y) ->
            tr (abstract (tr x))
        | App (_,x,y) ->
            (*App (App (S, tr (Abs x)), tr (Abs y))*)
            let f1 = free x
            and f2 = free y in
            if not f1 then
              app (app B (tr x |> shift)) (tr (abstract y))
            else if not f2 then
              app (app C (tr (abstract x))) (tr y |> shift)
            else
              app (app S (tr (abstract x))) (tr (abstract y))
        | _ ->
            assert false
      )
  | t -> t

module Parser = struct
  include ParserCombinators(struct type store = string list end)

  let update f = fun s -> LazyList.singleton ((), { s with store = f s.store })

  let lookup v =
    let rec find i = function
      | [] -> failwith "not found"
      | v'::t ->
          if v = v' then
            i
          else
            find (i+1) t
    in
    fun s -> LazyList.singleton (find 0 s.store, s)

  let term =
    let rec pvar = lazy (ident_ >>= fun v -> var <$> lookup v)
    and pabstract = lazy (
      char_ '\\' >> many1 ident_ >>= fun vs ->
      let l = List.length vs in
      char_ '.' >>
      update (fun ctx -> List.fold_left (fun ctx v -> v::ctx) ctx vs) >>
      (nest l <$> Lazy.force term) <<
      update (List.drop l)
    )
    and term0 = lazy ((char_ '(' >> (fun s -> Lazy.force term s) << char_ ')' <|> Lazy.force pabstract <|> Lazy.force pvar))
    and term = lazy (List.fold_left1 app <$> many1 (Lazy.force term0))
    in
    Lazy.force term
end

let () =
  let n = read_int () in
  for i = 1 to n do
    let s = read_line () in
    match Parser.parse Parser.term [] s with
    | None -> ()
    | Some t -> tr t |> print; print_endline ""
  done
```

## Infer

https://www.hackerrank.com/challenges/infer

http://okmij.org/ftp/ML/generalization.html Didier Rémy提出的加速generalization的算法。


```ocaml
module Str = struct
  let implode l =
    let s = Bytes.make (List.length l) ' ' in
    List.iteri (fun i c -> Bytes.set s i c) l;
    s
end

module LazyList = struct
  type 'a node = Nil | Cons of 'a * 'a t
  and 'a t = 'a node Lazy.t
  let empty = lazy Nil
  let singleton x = lazy (Cons (x, empty))
  let force = Lazy.force
  let rec map f l = lazy (
    match force l with
    | Nil -> Nil
    | Cons (h, t) -> Cons (f h, map f t)
  )
  let rec append l1 l2 = lazy (
    match force l1 with
    | Nil -> force l2
    | Cons (h, t) -> Cons (h, append t l2)
  )
  let rec concat ll = lazy (
    match force ll with
    | Nil -> Nil
    | Cons (h, t) -> append h (concat t) |> force
  )
  let is_empty l = force l = Nil
end

module ParserCombinators = struct
  type input = { s: bytes; pos: int }
  type 'a t = input -> ('a * input) LazyList.t
  let unit a s = LazyList.singleton (a, s)
  let zero = unit []
  let (>>=) (type a) (type b) (x : a t) (f : a -> b t) s =
    LazyList.map (fun (a,s) -> f a s) (x s) |> LazyList.concat
  let (>>) x y = x >>= fun _ -> y
  let (<<) x y = x >>= fun a -> y >> unit a
  let (<|>) x y s = let r = x s in if LazyList.is_empty r then y s else r
  let (<$>) f x = x >>= fun a -> unit (f a)
  let rec many x = many1 x <|> zero
  and many1 x = x >>= fun b -> many x >>= fun bs -> unit (b::bs)
  let sep_by x sep =
    let go = x >>= fun b ->
      many (sep >> x) >>= fun bs ->
      unit (b::bs)
    in
    go <|> zero
  let pred f s =
    if s.pos = Bytes.length s.s then
      LazyList.empty
    else
      let c = s.s.[s.pos] in
      if f c then unit c { s with pos = s.pos + 1 } else LazyList.empty
  let ident = Str.implode <$> many1 (pred (fun c ->
    'a'<=c&&c<='z' || 'A'<=c&&c<='Z' || '0'<=c&&c<='9' || c = '_'))
  let space = pred (fun c -> c = ' ' || c = '\t' || c = '\r' || c = '\n')
  let token x = x >>= fun a -> many space >> unit a
  let char c = pred ((=) c)
  let str cs =
    let rec go i =
      if i = Bytes.length cs then
        zero
      else
        char cs.[i] >> go (i+1)
    in
    go 0
  let ident_ = token ident
  let char_ c = token (char c)
  let str_ cs = token (str cs)
end

type expr =
  | Var of string
  | Fun of string list * expr
  | App of expr * expr list
  | Let of string * expr * expr
type level = int
type typ =
  | TConst of string
  | TVar of tv ref
  | TArrow of typ list * typ * levels
  | TApp of typ * typ list * levels
and tv = Unbound of int * level | Link of typ
and levels = { mutable level_old : level; mutable level_new : level }
let gray_level = -1
let generic_level = 19921213

let rec djs_find = function
  | TVar ({contents = Link t} as tv) ->
      let t = djs_find t in
      tv := Link t;
      t
  | t -> t

let get_level t =
  match djs_find t with
  | TConst _ -> 0
  | TVar ({contents = Unbound (_, l)}) -> l
  | TApp (_, _, ls)
  | TArrow (_, _, ls) -> ls.level_new
  | _ -> assert false

module Parser = struct
  include ParserCombinators
  let force = Lazy.force
  let generic_ctr = ref 0
  let tapp f args =
    let l = List.fold_left (fun acc a ->
      max acc (get_level a)) (get_level f) args in
    TApp (f, args, { level_old = l; level_new = l })
  let tarrow args r =
    let l = List.fold_left (fun acc a ->
      max acc (get_level a)) (get_level r) args in
    TArrow (args, r, { level_old = l; level_new = l })
  let typ () =
    let univ = Hashtbl.create 0 in
    let rec parse_ident =
      ident_ >>= fun n ->
      unit @@
        try TVar (ref @@ Link (Hashtbl.find univ n))
        with Not_found -> TConst n
    and parse_tys s = sep_by parse_ty (char_ ',') s
    and parse_ty s = (
      let t1 =
        let rec bracket f =
          (char_ '[' >> parse_tys << char_ ']' >>= fun args ->
          bracket (tapp f args)) <|> unit f
        in
        parse_ident >>= bracket >>= fun f ->
        (str_ "->" >> (fun s -> parse_ty s) >>= fun r ->
        unit @@ tarrow [f] r) <|> unit f
      in
      let t2 =
        char_ '(' >> parse_tys << char_ ')' >>= fun args ->
        (str_ "->" >> parse_ty >>= fun r ->
        unit @@ tarrow args r) <|> unit (List.hd args)
      in
      t1 <|> t2
    ) s
    and parse_top s = (
      (str_ "forall[" >> many ident_ << char_ ']' >>= fun vs ->
      List.iteri (fun i v ->
        decr generic_ctr;
        Hashtbl.replace univ v (TVar (Unbound (!generic_ctr, generic_level) |> ref))) vs;
      parse_ty) <|> parse_ty
    ) s
    in
    parse_top
  let expr =
    let rec parse_let s = (
      str_ "let" >> ident_ >>= fun n ->
      char_ '=' >> parse_expr >>= fun e ->
      str_ "in" >>
      parse_expr >>= fun b ->
      unit @@ Let (n, e, b)
    ) s
    and parse_fun s = (
      str_ "fun" >> many ident_ >>= fun args ->
      str_ "->" >> parse_expr >>= fun b ->
      unit @@ Fun (args, b)
    ) s
    and parse_simple_expr s = (
      let first = (char_ '(' >> parse_expr << char_ ')') <|>
        (ident_ >>= fun n -> unit @@ Var n)
      in
      let rec go a =
        (char_ '(' >> sep_by parse_expr (char_ ',') << char_ ')' >>= fun b ->
        go @@ App (a, b)) <|> unit a
      in
      first >>= go
    ) s
    and parse_expr s = (
      parse_let <|>
      parse_fun <|>
      parse_simple_expr
    ) s
    in
    parse_expr
  let eof s =
    if s.pos = Bytes.length s.s then
      LazyList.singleton ((), s)
    else
      LazyList.empty
  let parse x s =
    match force ((many space >> x << eof) { s; pos = 0 }) with
    | LazyList.Nil -> None
    | LazyList.Cons ((x, _), _) -> Some x
end

exception Cycle
exception Fail
exception Length
let gensym_ctr = ref 0
let gensym () =
  let n = !gensym_ctr in
  incr gensym_ctr;
  n
let reset_gensym () = gensym_ctr := 0
let cur_level = ref 0
let reset_level () = cur_level := 0
let enter_level () = incr cur_level
let leave_level () = decr cur_level
let new_var () = TVar (ref (Unbound (gensym (), !cur_level)))
let new_app f args = TApp (f, args, { level_new = !cur_level; level_old = !cur_level })
let new_arrow args r = TArrow (args, r, { level_new = !cur_level; level_old = !cur_level })

let adj_q = ref []
let reset_adj_q () = adj_q := []
let force_adj_q () =
  let rec go l acc t =
    match djs_find t with
    | TVar ({contents = Unbound (n, l')} as tv) ->
        if l < l' then
          tv := Unbound (n, l);
        acc
    | TApp (_, _, ls)
    | TArrow (_, _, ls) as t ->
        if ls.level_new = gray_level then
          raise Cycle;
        if l < ls.level_new then
          ls.level_new <- l;
        one acc t
    | _ -> acc
  and one acc = function
    | TApp (r, args, ls)
    | TArrow (args, r, ls) as t ->
        if ls.level_old <= !cur_level then
          t::acc
        else if ls.level_old = ls.level_new then
          acc
        else (
          let lvl = ls.level_new in
          ls.level_new <- gray_level;
          let acc = List.fold_left (go lvl) acc args in
          let acc = go lvl acc r in
          ls.level_new <- lvl;
          ls.level_old <- lvl;
          acc
        )
    | _ -> assert false
  in
  adj_q := List.fold_left one [] !adj_q

let rec update_level l = function
  | TConst _ ->
      ()
  | TVar ({contents = Unbound (n, l')} as tv) ->
      if l < l' then
        tv := Unbound (n, l)
  | TApp (_, _, ls)
  | TArrow (_, _, ls) as t ->
      if ls.level_new = gray_level then
        raise Cycle;
      if l < ls.level_new then (
        if ls.level_new = ls.level_old then
          adj_q := t :: !adj_q;
        ls.level_new <- l
      )
  | _ -> assert false

let rec unify t1 t2 =
  let t1 = djs_find t1 in
  let t2 = djs_find t2 in
  if t1 != t2 then
    match t1, t2 with
    | TConst t1, TConst t2 when t1 = t2 ->
        ()
    | TVar ({contents = Unbound (_, l)} as tv), t'
    | t', TVar ({contents = Unbound (_, l)} as tv) ->
        update_level l t';
        tv := Link t'
    | TApp (r1, args1, l1), TApp (r2, args2, l2)
    | TArrow (args1, r1, l1), TArrow (args2, r2, l2) ->
        if l1.level_new = gray_level || l2.level_new = gray_level then
          raise Cycle;
        if List.length args1 <> List.length args2 then
          raise Length;
        let lvl = min l1.level_new l2.level_new in
        l1.level_new <- gray_level;
        l2.level_new <- gray_level;
        List.iter2 (unify_level lvl) args1 args2;
        unify_level lvl r1 r2;
        l1.level_new <- lvl;
        l2.level_new <- lvl
    | _ -> raise Fail

and unify_level l t1 t2 =
  let t1 = djs_find t1 in
  update_level l t1;
  unify t1 t2

let gen t =
  let rec go t =
    match djs_find t with
    | TConst _ ->
        ()
    | TVar ({contents = Unbound (n, l)} as tv) ->
        if l > !cur_level then
          tv := Unbound (n, generic_level)
    | TApp (r, args, ls)
    | TArrow (args, r, ls) ->
        if ls.level_new > !cur_level then (
          List.iter go args;
          go r;
          let lvl = List.fold_left (fun acc a -> max acc (get_level a)) (get_level r) args in
          ls.level_new <- lvl;
          ls.level_old <- lvl
        )
    | _ -> assert false
  in
  force_adj_q ();
  go t

let inst t =
  let subst = Hashtbl.create 0 in
  let rec go = function
    | TVar {contents = Unbound (n, l)} when l = generic_level ->
        (try
          Hashtbl.find subst n
        with Not_found ->
          let tv = new_var () in
          Hashtbl.replace subst n tv;
          tv)
    | TVar {contents = Link t} ->
        go t
    | TApp (f, args, ls) when ls.level_new = generic_level ->
        new_app (go f) (List.map go args)
    | TArrow (args, r, ls) when ls.level_new = generic_level ->
        new_arrow (List.map go args) (go r)
    | t -> t
  in
  go t

let rec typeof env e =
  let rec go = function
    | Var x -> Hashtbl.find env x |> inst
    | Fun (args, e) ->
        let ty_args = List.map (fun x -> new_var ()) args in
        List.iter2 (Hashtbl.add env) args ty_args;
        let ty_e = go e in
        let r = new_arrow ty_args ty_e in
        List.iter (Hashtbl.remove env) args;
        r
    | App (e, args) ->
        let ty_fun = go e in
        let ty_args = List.map go args in
        let ty_res = new_var () in
        unify ty_fun (new_arrow ty_args ty_res);
        ty_res
    | Let (x, e1, e2) ->
        enter_level ();
        let ty_e1 = go e1 in
        leave_level ();
        gen ty_e1;
        Hashtbl.add env x ty_e1;
        let r = go e2 in
        Hashtbl.remove env x;
        r
  in
  go e

let rec check_cycle = function
  | TVar {contents = Link t} ->
      check_cycle t
  | TApp (r, args, ls)
  | TArrow (args, r, ls) ->
      if ls.level_new = gray_level then
        raise Cycle;
      let lvl = ls.level_new in
      ls.level_new <- gray_level;
      List.iter check_cycle args;
      check_cycle r;
      ls.level_new <- lvl
  | _ -> ()

let rec top_typeof env e =
  reset_gensym ();
  reset_level ();
  reset_adj_q ();
  let t = typeof env e in
  check_cycle t;
  t

let rec show t =
  let open Printf in
  let id2name = Hashtbl.create 0 in
  let rec go t =
    match djs_find t with
    | TConst n -> n
    | TVar ({contents = Unbound (n, _)}) ->
        (try
          Hashtbl.find id2name n
        with _ ->
          let i = Hashtbl.length id2name in
          let name = Char.chr (Char.code 'a' + i) |> String.make 1 in
          Hashtbl.replace id2name n name;
          name)
    | TApp (f, args, _) ->
        let u = go f in
        let v = String.concat ", " (List.map go args) in
        sprintf "%s[%s]" u v
    | TArrow (args, r, _) ->
        let f = function
          | TArrow _ -> false
          | _ -> true
        in
        let u = String.concat ", " (List.map go args) in
        let v = go r in
        if List.length args = 1 && f @@ djs_find (List.hd args) then
          sprintf "%s -> %s" u v
        else
          sprintf "(%s) -> %s" u v
    | _ -> assert false
  in
  let s = go t in
  let l = Hashtbl.length id2name in
  if l > 0 then (
    let vs = Hashtbl.fold (fun _ v l -> v::l) id2name [] |> List.sort compare in
    sprintf "forall[%s] %s" (String.concat " " vs) s
  ) else
    s

let extract = function
  | Some x -> x
  | None -> assert false

let core =
  [ "head", "forall[a] list[a] -> a"
  ; "tail", "forall[a] list[a] -> list[a]"
  ; "nil", "forall[a] list[a]"
  ; "cons", "forall[a] (a, list[a]) -> list[a]"
  ; "cons_curry", "forall[a] a -> list[a] -> list[a]"
  ; "map", "forall[a b] (a -> b, list[a]) -> list[b]"
  ; "map_curry", "forall[a b] (a -> b) -> list[a] -> list[b]"
  ; "one", "int"
  ; "zero", "int"
  ; "succ", "int -> int"
  ; "plus", "(int, int) -> int"
  ; "eq", "forall[a] (a, a) -> bool"
  ; "eq_curry", "forall[a] a -> a -> bool"
  ; "not", "bool -> bool"
  ; "true", "bool"
  ; "false", "bool"
  ; "pair", "forall[a b] (a, b) -> pair[a, b]"
  ; "pair_curry", "forall[a b] a -> b -> pair[a, b]"
  ; "first", "forall[a b] pair[a, b] -> a"
  ; "second", "forall[a b] pair[a, b] -> b"
  ; "id", "forall[a] a -> a"
  ; "const", "forall[a b] a -> b -> a"
  ; "apply", "forall[a b] (a -> b, a) -> b"
  ; "apply_curry", "forall[a b] (a -> b) -> a -> b"
  ; "choose", "forall[a] (a, a) -> a"
  ; "choose_curry", "forall[a] a -> a -> a"
  ]

let core_env =
  let env = Hashtbl.create 0 in
  List.iter (fun (var, typ) ->
    let t = Parser.parse (Parser.typ ()) typ |> extract in
    Hashtbl.replace env var t
  ) core;
  env

let type_check line =
  let env = Hashtbl.copy core_env in
  Parser.parse Parser.expr line |> extract |> top_typeof env

let () =
  read_line () |> type_check |> show |> print_endline
```
