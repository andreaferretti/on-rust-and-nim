I was recently contacted from a person I had never met before, asking for some comments on Nim and Rust. Since I have taken a little time to answer, I tought I might as well publish my thoughts.

Please, note that I do not intend to bash Rust, and in fact I am looking forward to their 1.0 release. Also, links are to the 0.12 documentation, because if I recall correctly that was the version I was using when the events took place, but the substance remains the same with the latest releases.

The original email
==================

I hope you don't mind if I contact you directly, and ignore if you're offended, but I saw your post on the parasail email list and looked at your [KMeans benchmark](https://github.com/andreaferretti/kmeans).

In particular, I was interested in your statement that rust was hard and that you found Nim much easier. I'm interested in both and did some additional searching and found [this](http://hookrace.net/blog/conclusion-on-nim/) from [Dennis Felsing](https://github.com/def-) and he too likes Nim a lot.

I was wondering if you could expand on your dislike of Rust?

Again, feel free to ignore this email.

My answer
=========

Hi,

I will be happy to expand. First, I should make clear that I do not "dislike" Rust. It is just that, while I appreciate some ideas in theory, in practice i have found it hard to use.

TL;DR:

* Rust has good theoretical ideas, but they do not seem to translate to a very usable language.
* Nim is more rough in the edges, some features interact in unexpected ways, and the language in general in bigger, but it is far easier, and in general it feels much more practical.

I would choose Nim for projects where I can afford a GC, and evaluate case by case for other projects.

General remarks
---------------

As you probably know, Rust has a compile-time mechanism that guarantees that you make an appropriate use of pointers, avoiding such errors as: use of uninitialized pointers, double free, use after free and so on. Essentially, it does this through the use of compiler-enforced smart pointers, such as unique pointers or shared pointers. To achieve this, the notion of a type is expanded in such a way that each pointer has a type and also a lifetime, and the compiler checks that the use of lifetimes is correct. I will not expand, as many resources on Rust explain the mechanism in detail.

Now, Nim does not have anything like this. In Nim, you either use the garbage collector, or you manually manage pointers, with the associated possibility of errors. That is, while you can program at a low level in Nim, you do not have the safety net provided by Rust. This is no more or no less safe than managing pointers in C, C++ or D.

I believe that much of the difficulty I found in Rust was due to this mechanism, and another part was probably due to the fact that Nim has a much cleaner syntax, which you can easily check by comparing the two implementation, say, of Point: [Nim](https://github.com/andreaferretti/kmeans/blob/master/nim/algo.nim) (first 18 lines) vs [Rust](https://github.com/andreaferretti/kmeans/blob/master/rust/src/point/mod.rs)

Writing K-means in Rust
-----------------------

Now, what were in practice the stumbling blocks that I found?

1) The algorithm uses a hash map indexed by points. Now, how do I put a point into a hasmap? My first attempt used

http://doc.rust-lang.org/0.12.0/std/collections/struct.HashMap.html

But this lead me to the problem of hashing points. This in turn, would require me to hash each coordinate, but Double in Rust does not implement the trait Hash. Now, there are good reasons for doing so, due to the subtleties of floating point  arithmetic, but they did not apply in my case.

2) So I had to find an alternative solution. Let us use

http://doc.rust-lang.org/0.12.0/std/collections/struct.TreeMap.html

Unfortunately, the operation `find` on a TreeMap returns an immutable reference to the value (even though the TreeMap is mutable!), but need to mutate it. With a little [help online](http://stackoverflow.com/questions/26378178/trying-to-dereference-pointer) I was able to make it work, but the solution proposed was kind of hacky and certainly not very clean (look at the answer!). It also involved some copying.

3) Then I tried to parse JSON to read the points. The [obvious attempt](http://stackoverflow.com/questions/26336281/read-json-in-rust) did not work and the proposed solution involved a 11 line definition to define how to parse a point.

4) At this point I went back trying to use an actual `HashMap`. I wanted to write a `Hash` trait for `Point`, but this required me to define a method that had a `std::hash::sip::SipState` in its signature. I really did not want to bother understanding what that even is, so the thing stayed slow for a while.

5) Then with the help of one of Rust mantainers [here](http://codereview.stackexchange.com/questions/67577/k-means-in-rust) I was finally able to get it working as it is now. Notice the use of `mem::transmute` that I don't think I would have been able to find myself.

Writing K-means in Nim
----------------------

In Nim, I was able to obtain an implementation (garbage-collected) of about the same running time in the very first morning I had ever heard of Nim, essentially by literal translation of the Python version. It turned out I had made an unnecessary copy, and once that was cleared out, the Nim version was faster (!).

What I do not like in Rust
--------------------------

So, while I believe that Rust has good principles, I also think that languages should have a decent ergonomics, and Rust fails at that on many fronts:

* the whole language is verbose: compare [these 10 lines](https://github.com/andreaferretti/kmeans/blob/master/rust/src/point/mod.rs#L30)
with [this single line](https://github.com/andreaferretti/kmeans/blob/master/nim/algo.nim#L10)
* the API of the standard library is overcomplicated (look at `SipHash` above, or the mutability issue on `TreeMap::find`, or the hoops to jump to parse a JSON file)
* cargo itself requires a byzantine structure to distinguish between lib and executable: look at the directory structure of the Rust example
* the documentation is also quite poor. Look at the [documentation for Hash](http://doc.rust-lang.org/0.12.0/std/hash/trait.Hash.html) Is it clear to you what you have to implement? What is that type parameter `S`?

There are other small things that turned out not to be a problem, but were nevertheless overcomplicated. Look at [this line](https://github.com/andreaferretti/kmeans/blob/master/rust/src/algo/mod.rs#L45)

Why do I have to convert the vector to iterator and back, instead of being able to map it directly? I guess this is to leave the possibility to map into another container type (say, from a `Vector` to a `List`), but adding a map function on Vector would have not prevented this more sophisticated use.

What I like in Nim
------------------

There are also a lot of things that Nim gives you and Rust does not have, such as

* simpler macros and even simpler templates
* a lot of control on how to interoperate with C, including calling conventions
* an effect system
* a limited form of dependent types through static types and range types
* the possibility to add macro transformations as hints for the compiler for optimization
* dynamic binding, when needed

and probably more

What I do not like in Nim
-------------------------

What are the pain points of Nim? There are not many, but I will mention:

* the language in general feels big, and I constantly have the sensation that some independent features may have [unexpected interactions](http://forum.nim-lang.org/t/796)
* the error reporting from the compiler is often difficult to understand
* there are much less libraries available

Conclusion
----------

Now, don't take my words: there may be many inaccuracies above, as I am a beginner both with Rust and Nim. If anything, they prove my point better: even after going through a lot of hoops, I still do not have clear many things about Rust.

I hope this helps you!
Best,
Andrea