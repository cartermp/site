---
title: How to use F# and BenchmarkDotNet
date: 2020-06-25
author:
  - Phillip Carter
tags:
  - fsharp

# Set how many table of contents levels to be showed on page.
geekblogToC: 3

# Set true to hide page or section from side menu (file-tree menu only).
geekblogHidden: false

# Add an anchor link to headlines.
geekblogAnchor: true
---

Every once in a while, you'll want to compare to performance of two or more routines, algorithms, types, etc. This is called [benchmarking](https://en.wikipedia.org/wiki/Benchmarking) and it's a lot of fun to do in F#. Here's how you do it.

## How to install BenchmarkDotNet

First, make sure you've got the latest [.NET SDK](https://dotnet.microsoft.com/download). Then create a new console app:

```console
dotnet new console -lang F# -o BasicExperiment
```

Navigate to the `BasicExperiment` directory and add the package:

```console
dotnet add package benchmarkdotnet
```

Now you can write code!

## Simple BenchmarkDotNet example

Here's a simple benchmark comparing a naiive implementation for `List.length` to the built-in one on a list of 1 million integers.

```fsharp
// Learn more about F# at http://docs.microsoft.com/dotnet/fsharp

open BenchmarkDotNet.Attributes
open BenchmarkDotNet.Running

module List =
    let inline myLength xs =
        let rec loop xs acc =
            match xs with
            | [] -> acc
            | _ :: tail ->
                loop tail (acc + 1)
        loop xs 0

[<MemoryDiagnoser>]
type LengthBench() =
    let lst = [ 1..1_000_000 ]

    [<Benchmark(Baseline=true)>]
    member _.BuiltIn() = List.length lst

    [<Benchmark>]
    member _.Custom() = List.myLength lst

[<EntryPoint>]
let main argv =
    BenchmarkRunner.Run<LengthBench>() |> ignore
    0 // return an integer exit code
```

The results looked like this when I ran it:

|  Method |     Mean |     Error |    StdDev | Ratio | RatioSD | Gen 0 | Gen 1 | Gen 2 | Allocated |
|-------- |---------:|----------:|----------:|------:|--------:|------:|------:|------:|----------:|
| BuiltIn | 2.857 ms | 0.0562 ms | 0.0788 ms |  1.00 |    0.00 |     - |     - |     - |         - |
|  Custom | 3.077 ms | 0.0608 ms | 0.1127 ms |  1.09 |    0.05 |     - |     - |     - |         - |

So it looks like my routine is in the same ballpark. Cool!

There are three important things here aside from the structure of the code:

1. The `Benchmark` attribute, which tells BenchmarkDotNet which methods are benchmarks
2. The `Baseline` property, which establishes the baseline results to compare against (in this case, the built-in `List.length` function)
3. The `Baseline` property, which establishes the baseline results to compare against (in this case, the built-in `List.length` function)
4. The `MemoryDiagnoser` attribute, which measures allocations for a given benchmark.

The fourth is particularly important for routines that aren't 100% CPU operations like computing the length of a list. You may find yourself in a situation where one routine is a little faster but allocates more memory. In some situations, less allocations might matter more than a few milliseconds of execution time, for example.

I personally prefer to cleanly separate out the routine I'm testing from the benchmark code. For larger pieces of code, I may move the implementation into a separate file.

Another consideration is looking into some of the Benchmark.NET APIs for setup and teardown. Some benchmarks have complicated initialization requirements that the sample doesn't cover. But before you go reaching for more machinery to help you test things, I'd recommend seeing what you can change so that you can stick with something simple and straightforward.

## What to look for in the results

The first thing to look for is pretty obvious: which benchmark has the faster execution time? The mean is usually good enough, but you should also check the standard deviation to see how much variance there is. Two algorithms could show similar mean execution times, but one may have a smaller standard deviation which could imply more consistent execution time.

The second thing to look for is initially straightforward: which benchmark allocates more memory? At least in .NET, the less memory you allocate, the better (usually). But this can then complicate matters. If one algorithm is 2x faster but allocates 2x as much memory, is it better? What about the inverse? The answers to these questions will depend on circumstances specific to your application. For example, is memory an issue because you have a long-running process where you'd like as little GC as possible? Or does it not matter because you have a process that just runs once and will likely never run out of memory?

## Don't bother trying to outsmart BenchmarkDotNet

You might notice some variance in overall results if you run the benchmarking program multiple times, where one algorithm goes back and forth between running marginally faster or slower than the other. This doesn't really mean much other than that the 2 (or more) routines are basically the same from a performance perspective.

You may be tempted to do something like run an algorithm many times for each benchmark just to "really be sure" it's being tested correctly. This is a waste of time, since BenchmarkDotNet will run each benchmark enough times so that it has statistically significant results. Additionally, artifically calling an algorithm many times just to "really be sure" about it means that you're not actually testing that algorithm anymore.

## Performance work is hard

Lastly, don't let the simplicity of a great tool like BenchmarkDotNet fool you into thinking that performance work is easy. Yes, if you can benchmark an algorithm and show that it is significantly faster than another one, you should probably just use the faster algorithm. But this kind of experimentation barely scratches the surface of what performance work (at least for F# and .NET applications) can actually entail. If you're interested in how deep it can go, there are some [great books on Amazon](https://www.amazon.com/s?k=.NET+performance&ref=nb_sb_noss_2) covering the topic.
