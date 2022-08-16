---
layout: post
title: Primality Testing in OCaml - Part I
---

This is the first in a series of posts about implementing primality tests in the [OCaml programming language](https://ocaml.org). I first stumbled upon OCaml by accident when I was reading about the [Coq proof assistant](https://coq.inria.fr), and it quickly became one of my favorite general-purpose programming languages.

<!--more-->

One of the exercises in OCaml's [list of 99 problems](https://ocaml.org/problems) is to test the primality of a given integer. This problem got me thinking about different primality tests and how easy or difficult it would be to implement them in a functional style in vanilla OCaml. In this post I will talk about implementing a naïve primality test: an [Euler seive](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes#Euler's_sieve). In future posts I plan to work through some more sophisticated tests and compare their performance.

### The idea:

The idea behind the Euler seive is very simple: let \\(n\\) be a positive integer. If \\(n\\) is composite, then it must have a factor less than or equal to \\(\sqrt{n}\\), which means it must have a prime factor less than or equal to \\(\sqrt{n}\\). So, consider the set of integers

\\[\\{2,3,4,\ldots, \lfloor{\sqrt{n}}\rfloor\\}.\\]

\\(n\\) is composite if and only it has a divisor in this set. So beginning with the smallest element of the set, test whether or not it divides \\(n\\). If it does, then \\(n\\) is composite. If it does not, remove that element _and all of its multiples_ from the set, and repeat. At each step you either find a divisor of \\(n\\) or you strictly reduce the number of elements remaining in the set. If you manage to reach the empty set, then \\(n\\) is prime.

### A worked example:

Say we wanted to test the primality of \\(409\\). Its square root rounds down to \\(20\\), so we need to test whether \\(409\\) has a divisor in the set

\\[\\{2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20\\}.\\]

Since \\(2\\) does not divide \\(409\\), we remove the multiples of \\(2\\) from the set and try again, with the set of candidate divisors reduced to:

\\[\\{3,5,7,9,11,13,15,17,19\\}\\]

Since \\(3\\) does not divide \\(409\\), we now remove all multiples of \\(3\\) from the set, leaving

\\[\\{5,7,11,13,17,19\\}.\\]

Further iterations of this process will end up only removing the smallest element from the set. We end up testing whether \\(409\\) is divisible by \\(2,3,5,7,11,13,17,19\\), and in each case finding that it is not. Since \\(409\\) doesn't have a prime factor less than or equal to \\(\sqrt{20}\\), it must be prime.

### An OCaml implementation:

The Euler Seive as described above is very amenable to being implemented in a recursive and functional style. The only hiccup is that OCaml doesn't have a built-in utility for creating a list of all integers between given bounds. We will want this not only as a component of our algorithm, but also for testing it. So we'll create a `range` utility as a standalone function and also choose a convenient infix operator.

```ocaml
  let range a b =
    let rec range_aux a b lst = match b - a with
      | d when d < 0 -> []
      | 0 -> a :: lst
      | _ -> range_aux a (b-1) (b :: lst) in
    range_aux a b []

  let (--) a b = range a b
```

It is important to point out that a slightly more direct implementation might feel more natural here:

```ocaml
  let rec range a b = match b - a with
    | d when d < 0 -> []
    | 0 -> [a]
    | _ -> a :: range (a + 1) b
```

However, this second `range` function is not suitable for our purposes, because is not [tail recursive](https://en.wikipedia.org/wiki/Tail_call). Using this second `range` function to create a list of length around \\(10^6\\) will overflow the stack. But our original (and tail-recursive) implementation can handle ranges several orders of magnitude longer before it starts to slow down for other reasons.

With our `range` function in hand, we're basically ready to implement the Euler seive in OCaml: Here's the quick summary. Given an integer \\(n\\):

1. If \\(n < 2\\) then it's not prime
2. But if \\(n \geqslant 2\\), make the list from \\(2\\) to \\(\lfloor\sqrt{n}\\rfloor\\). If \\(n\\) has a proper divisor, then it must have a proper divisor in this list.
3. If the smallest integer in that list divides \\(n\\), then \\(n\\) is not prime. If the list is empty, then \\(n\\) is prime.
4. But if neither of those cases hold, then remove the smallest element of the list together with all of its multiples, and repeat steps 3-4.

My OCaml implementation ended up looking like this:

```ocaml
  let is_prime n = match n with
    | n when n < 2 -> false
    | n -> let candidate_divisors =
             n
             |> float_of_int
             |> Float.sqrt
             |> Float.floor
             |> int_of_float
             |> range 2 in
           let rec seive div_list m = match div_list with
             | [] -> true
             | p :: qs -> if m mod p = 0
                          then false
                          else
                            let not_p_mult q = q mod p <> 0 in
                            let ds = List.filter not_p_mult qs in
                            seive ds m in
           seive candidate_divisors n;;
```

I'll save the more exhaustive benchmarking for a future post, where I'll measure the performance of this `is_prime` function against some newer implementations. But for now it's nice to at least spot check the correctness of the function.  For example, the list of primes less than \\(500\\) can be found with `1 -- 500 |> List.filter is_prime`, which gives

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

and these are indeed the \\(95\\) primes less than \\(500\\). We could also test the primality of a large prime like `is_prime 1_000_000_009`, or a challenging semiprime like `is_prime 10_003_800_361`. This second number is a worst-case composite number for the Euler sieve, since it's the square of a large prime.

As one more spot check of correctness we might use our `is_prime` function to implement the [prime counting function](https://en.wikipedia.org/wiki/Prime-counting_function). Usually denoted \\(\pi(x)\\), it is defined to be the number of primes less than or equal to \\(x\\).

```ocaml
  let prime_pi n = 1 -- n
                   |> List.filter is_prime
                   |> List.length
```

Asking for `prime_pi 1_000_000` (accurately) gives `78498`, so we can be reasonably assured of the correctness of our code (Although this took about a minute to run on my system. If we were actually interested in the prime counting function itself, this would not be the best way to implement it).

