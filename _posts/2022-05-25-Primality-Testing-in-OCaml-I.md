---
layout: post
title: Primality Testing in OCaml - Part I
---

This is the first in a series of posts about implementing primality tests in the [OCaml programming language](https://ocaml.org). I first stumbled upon OCaml by accident when I was reading about the [Coq proof assistant](https://coq.inria.fr), and it has since become one of my favorite general-purpose programming languages.

<!--more-->

One of the exercises in OCaml's [list of 99 problems](https://ocaml.org/problems) is to test the primality of a given integer. This problem got me thinking about different primality tests and how easy or difficult it would be to implement them in a functional style in vanilla OCaml. In this post I will talk about implementing one of the most naïve [primality tests](https://en.wikipedia.org/wiki/Primality_test) out there: [trial division](https://en.wikipedia.org/wiki/Trial_division). In future posts I will to work through some more sophisticated tests and compare their performance.

### The idea

The idea behind trial division is very simple: let \\(n\\) be a positive integer. If \\(n\\) is composite, then it must have a proper divisor less than or equal to \\(\sqrt{n}\\). So consider the set of integers

\\[\\{2,3,4,\ldots, \lfloor{\sqrt{n}}\rfloor\\}.\\]

\\(n\\) is composite if and only it has a divisor in this set.

### First implementation

Let's start by implementing this test directly, putting off any optimizations or refinements for the moment. The only thing we need before writing the primality test itself is a `range` function to generate the list of integers between specified bounds.

```ocaml
  let range a b =
    let rec range_aux a b lst = match b - a with
      | d when d < 0 -> []
      | 0 -> a :: lst
      | _ -> range_aux a (b-1) (b :: lst) in
    range_aux a b []

  let (--) a b = range a b
```

It is important to point out that you can write a shorter, more idiomatic range function like this:

```ocaml
  let rec range a b = match b - a with
    | d when d < 0 -> []
    | _ -> a :: range (a + 1) b
```

However, this second `range` function is not suitable for our purposes, because is not [tail recursive](https://en.wikipedia.org/wiki/Tail_call). Using this second `range` function to create a list of length around \\(10^6\\) will overflow the stack. But our original (and tail-recursive) implementation can handle ranges a few orders of magnitude longer before it starts to slow down for other reasons. With our `range` function in hand, we can implement our trial division primality test.

```ocaml
  let is_prime n =
    if n < 2
    then false
    else n
         |> float_of_int
         |> Float.sqrt
         |> Float.floor
         |> int_of_float
         |> (--) 2
         |> List.map ((mod) n)
         |> List.mem 0
         |> not
```

And just like that, we have a working primality test. It will stack overflow if `n` has around 12 digits, most likely because `List.map` is [not tail recursive](https://v2.ocaml.org/api/List.html). But it's an easy-to-implement primality test and it runs decently quickly on single inputs from within the range it can handle.

### Small optimizations

There are two clear improvements that can be made to the trial division algorithm.

1. There is no need to test whether \\(n\\) is divisible by *all* integers less than or equal to \\(\sqrt{n}\\). We only need to test divisibility until we find a divisor. At that point we know \\(n\\) is composite and can stop. If \\(n\\) is composite with a small divisor, this will save some time. Though if \\(n\\) is prime or a perfect square, it saves no time.

2. There is no need to test divisibility by even integers. We can just test if \\(n\\) is even at the outset. If it's not, we can do trial division with the odd integers less than or equal to \\(\sqrt{n}\\). This cuts the size of the list of potential divisors in half.

In fact, (2) can be extended a bit further. Instead of first checking if \\(n\\) is even, we first check whether it's divisible by \\(2\\) or \\(3\\). If it's not, then the following fact will simplify our search for divisors.

> If \\(n > 3\\) is not divisible by \\(2\\) or \\(3\\), then \\(n\\) must have a divisor which is less than or equal to \\(\sqrt{n}\\) and which is of the form \\(6k \pm 1\\) for some integer \\(k\\).

This is because any number that's not of the form \\(6k \pm 1\\) is divisible by \\(2\\) or \\(3\\). Since \\(n\\) isn't divisible by \\(2\\) or \\(3\\), it can't have a divisor that's divisible by \\(2\\) or \\(3\\). This lets us throw out two thirds of the set of potential divisors.

Re-implementing trial division with these two small optimizations will require just a tiny bit of setup, mostly for convenience. First, we'd like a function for the boolean-valued binary relation "divides". This is usually written with the vertical bar or "pipe" symbol \\(\|\\).

> We say that \\(a\\) divides \\(b\\), written \\(a \mid b\\), if there exists an integer \\(k\\) satisfying \\(a k = b\\).

Unfortunately we can't use the `|` operator in OCaml for this purpose. We'll settle for `|?`

```ocaml
    let divides a b = b mod a = 0;;

    let ( |? ) = divides;;
```

Also, given a predicate `pred : 'a -> bool` and a list `lst : 'a list`, we want to know if `lst` has an element `x` for which `p x` is `true`.

```ocaml
    let rec has_element_with pred lst = match lst with
      | [] -> false
      | x :: xs -> (pred x) || (has_element_with pred xs);;
```

Because the or operator `||` is evaluated [lazily](https://en.wikipedia.org/wiki/Lazy_evaluation), if the first element of `lst` satisfies the predicate, the function returns `true` and does not consider the tail of the list. With our `divides` operator and our lazy replacement for `List.mem` in hand, we can now write our slightly-improved trial division primality test.

```ocaml
  let is_prime n = match n with
    | n when (n = 2) || (n = 3) -> true
    | n when (n < 2) || (2 |? n) || (3 |? n) -> false
    | n -> let p = fun i -> (6*i - 1 |? n) || (6*i + 1 |? n) in
           n
           |> float_of_int
           |> Float.sqrt
           |> Float.floor
           |> int_of_float
           |> (+) 1
           |> (fun m -> m / 6)
           |> (--) 1
           |> has_element_with p
           |> not
```

It's easy enough to spot-check correctness with something like

```ocaml
  List.filter is_prime (1--500)
```

which gives

```ocaml
- : int list =
[2; 3; 5; 7; 11; 13; 17; 19; 23; 29; 31; 37; 41; 43; 47; 53; 59; 61;
 67; 71; 73; 79; 83; 89; 97; 101; 103; 107; 109; 113; 127; 131; 137;
 139; 149; 151; 157; 163; 167; 173; 179; 181; 191; 193; 197; 199; 211;
 223; 227; 229; 233; 239; 241; 251; 257; 263; 269; 271; 277; 281; 283;
 293; 307; 311; 313; 317; 331; 337; 347; 349; 353; 359; 367; 373; 379;
 383; 389; 397; 401; 409; 419; 421; 431; 433; 439; 443; 449; 457; 461;
 463; 467; 479; 487; 491; 499]
```

and these are indeed the \\(95\\) primes below \\(500\\). We could also test the primality of a large prime like `is_prime 1_000_000_009`, or a large semiprime like `is_prime 10_003_800_361`. This second number is a worst-case composite number for trial division. Since it's the square of a large prime, we don't find a divisor until the very end of our search.

We might also use our `is_prime` function to implement the [prime counting function](https://en.wikipedia.org/wiki/Prime-counting_function). Usually denoted \\(\pi(x)\\), it is defined to be the number of primes less than or equal to \\(x\\).

```ocaml
  let prime_pi n = 1 -- n
                   |> List.filter is_prime
                   |> List.length
```

Asking for `prime_pi 1_000_000` gives the correct value of `78498`, so we can be reasonably assured of the correctness of our code (Though if we were actually interested in the prime counting function itself, this would not be the best way to implement it).

Finally, we'll briefly compare our first naïve implementation to the version with optimizations, by using both of them to test primality of the first \\(10^7\\) integers. The naïve implementation runs in `964.78` seconds, while the version with lazy evaluation and the \\(6k \pm 1\\) heuristic runs in `31.58` seconds, a roughly \\(30\\) times speedup (mostly because of the lazy evaluation). We'll put off the more serious benchmarking until we have more sophisticated primality tests to compare against.
