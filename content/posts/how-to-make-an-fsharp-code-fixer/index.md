---
title: How to make an F# code fixer
date: 2020-12-04
author:
  - Phillip Carter
tags:
  - fsharp
  - Visual Studio
resources:
  - name: yeet
    src: "images/wrap-expression-in-parentheses.png"
    title: Wrap Expression in Parentheses Quick Fix

# Set how many table of contents levels to be showed on page.
geekblogToC: 3

# Set true to hide page or section from side menu (file-tree menu only).
geekblogHidden: false

# Add an anchor link to headlines.
geekblogAnchor: true
---

Note: this post does not apply to Jetbrains Rider. Rider uses its own engine for representing F# syntax expressions and has its own strongly-typed API for traversing and manipulating F# expressions.

F# tooling in Visual Studio and Visual Studio Code supports a variety quick fixes for fixing an error in your code. Here's an example of one:

{{< img name="yeet" size="small" lazy=true >}}

Pretty neat, right? This post will walk through the essentials of implementing a quick fix like this in either Visual Studio or VSCode.

## The essential pieces of an editor Quick Fixer

Quick Fixes are pretty straightforward. They are comprised of 3 things:

1. An editing environment that can "listen" for specific diagnostics (tracked by ID) and allow you to plug into that engine
2. A "context" for a Quick Fix that crucially contains the span/range of text in a document corresponding to an error or warning
3. Some code that registers itself as a plugin for that diagnostic ID and/or message contents, and/or some other condition (more on that later)
4. Some code that performs logic that rewrites a small section of the user's code to fix an issue
And that's it! The lifecycle is pretty simple, too:

Periodically, an editing environment calls into the F# language service to process syntax and typecheck. This happens most often when you're typing (after a very short delay to account for the typing). When it's finished and there are syntax or typechecking errors, it raises appropriate diagnostics for the editing environment to report.

When this happens, any quick fix that is registered to "listen" to a particular diagnostic is made available to be triggered if and only if that diagnostic was raised. When the user does something like click a lightbulb in an editor or hit the right key command, all Quick Fixes that are available at that position are executed asynchronously, and the syntax transformation that they offer is also made available.

## Each editor has their own APIs

First things first: you can't just copy/paste a quick fix from Visual Studio into VSCode or vice/versa. Although a quick fix can share the same logic across editors, it must ultimately bind to the particular editor API that hosts it.

In the case of Visual Studio tooling for F#, the skeleton that wraps any custom logic generally looks like this:

```fsharp
namespace Microsoft.VisualStudio.FSharp.Editor

open System.Composition
open System.Threading
open System.Threading.Tasks

open Microsoft.CodeAnalysis.Text
open Microsoft.CodeAnalysis.CodeFixes
open Microsoft.CodeAnalysis.CodeActions

[<ExportCodeFixProvider(FSharpConstants.FSharpLanguageName, Name = "NAME HERE"); Shared>]
type internal FSharpYourQuickFixNameHereFixProvider() =
    inherit CodeFixProvider()

    // Any applicable diagnostic IDs go here
    let fixableDiagnosticIds = set ["FSXYZ"]

    override _.FixableDiagnosticIds = Seq.toImmutableArray fixableDiagnosticIds

    override this.RegisterCodeFixesAsync context : Task =
        async {
            // Title comes from a resource file
            let title = SR.WrapExpressionInParentheses()

            // Custom logic can be written or called here

            let applicableIDs =
                context.Diagnostics
                |> Seq.filter (fun x -> this.FixableDiagnosticIds.Contains x.Id)
                |> Seq.toImmutableArray

            context.RegisterCodeFix(
                CodeAction.Create(
                    title,
                    (fun (cancellationToken: CancellationToken) ->
                        async {
                            let! sourceText = context.Document.GetTextAsync(cancellationToken) |> Async.AwaitTask
                            return context.Document.WithText((* TODO - code that changes text *))
                        } |> RoslynHelpers.StartAsyncAsTask(cancellationToken)),
                    title),
                    applicableIDs)
        } |> RoslynHelpers.StartAsyncUnitAsTask(context.CancellationToken)
```

It may seem like there's a lot going on here, but most of it is just glue code to ensure that everything is asynchronous and cancellable and runs in the Roslyn workspace host inside of Visual Studio. They key pieces are there:

Configuring a set of applicable diagnostics for the code fix
Code that registers a quick fix for the applicable diagnostics (asynchronous and cancellable)
Spots in the code to enter in custom logic and logic for manipulating user code
In VSCode (technically FsAutocomplete), a quick fix skeleton might look similar to this:

```fsharp
let yourCustomeCodeFix (getFileLines: string -> Result<string [], _>): CodeFix =
    ifDiagnosticByCode
        (fun diagnostic codeActionParams ->
            match getFileLines (codeActionParams.TextDocument.GetFilePath()) with
            | Ok lines ->
                let erroringExpression = getText lines diagnostic.Range
                async.Return [ { Title = "your title here"
                                 File = codeActionParams.TextDocument
                                 SourceDiagnostic = Some diagnostic
                                 Edits =
                                    [| { Range = diagnostic.Range
                                         NewText = "" (* TODO - define new text *) } |]
                                 Kind = Fix } ]
            | Error _ -> async.Return [])
        (Set.ofList [ "DIAGNOSTIC-IDS-HERE" ])
```

Due to some nice helper functionality it's less code, but the basic pieces are all the same.

## Easy quick fixer example: just manipulating text

Sometimes, a quick fix can be trivial to implement because all you need to do is change an obviously incorrect span of text in a user's source code. The following example comes from a very common error:

```fsharp
let rng = System.Random()
let makeBigger x = x * 2
makeBigger rng.Next(5)
```

This code seems like it might be right, but the compiler complains because it thinks that the `(5)` is another argument being passed to `makeBigger`. It's a "classic" F# compiler error that is usually resolved by adding parentheses. So, why not make a Code Fix that adds the parentheses? As it turns out, that is trivial.

Here's how it is done in Visual Studio:

```fsharp
namespace Microsoft.VisualStudio.FSharp.Editor

open System.Composition
open System.Threading
open System.Threading.Tasks

open Microsoft.CodeAnalysis.Text
open Microsoft.CodeAnalysis.CodeFixes
open Microsoft.CodeAnalysis.CodeActions

[<ExportCodeFixProvider(FSharpConstants.FSharpLanguageName, Name = "AddParentheses"); Shared>]
type internal FSharpWrapExpressionInParenthesesFixProvider() =
    inherit CodeFixProvider()

    // FS0597 is the ID for the diagnostic that gets triggered
    let fixableDiagnosticIds = set ["FS0597"]

    override _.FixableDiagnosticIds = Seq.toImmutableArray fixableDiagnosticIds

    override this.RegisterCodeFixesAsync context : Task =
        async {
            // Title comes from a resource file
            let title = SR.WrapExpressionInParentheses()

            let applicableIDs =
                context.Diagnostics
                |> Seq.filter (fun x -> this.FixableDiagnosticIds.Contains x.Id)
                |> Seq.toImmutableArray

            // This will wrap a range of text in parentheses
            let getChangedText (sourceText: SourceText) =
                sourceText.WithChanges(TextChange(TextSpan(context.Span.Start, 0), "("))
                          .WithChanges(TextChange(TextSpan(context.Span.End, 0), ")"))

            context.RegisterCodeFix(
                CodeAction.Create(
                    title,
                    (fun (cancellationToken: CancellationToken) ->
                        async {
                            let! sourceText = context.Document.GetTextAsync(cancellationToken) |> Async.AwaitTask
                            return context.Document.WithText(getChangedText sourceText)
                        } |> RoslynHelpers.StartAsyncAsTask(cancellationToken)),
                    title),
                    applicableIDs)
        } |> RoslynHelpers.StartAsyncUnitAsTask(context.CancellationToken)
```

Because the diagnostic itself has a range that encapsulates the entire troublesome expression, all we need to do is wrap parentheses around that range in a document.

The same quick fix in VSCode looks like this:

```fsharp
/// a codefix that parenthesizes a member expression that needs it
let parenthesizeExpression (getFileLines: string -> Result<string [], _>): CodeFix =
  ifDiagnosticByCode
    (fun diagnostic codeActionParams ->
      match getFileLines (codeActionParams.TextDocument.GetFilePath()) with
      | Ok lines ->
          let erroringExpression = getText lines diagnostic.Range
          async.Return [ { Title = "Wrap expression in parentheses"
                           File = codeActionParams.TextDocument
                           SourceDiagnostic = Some diagnostic
                           Edits =
                               [| { Range = diagnostic.Range
                                    // Using a string interpolation to supply new text
                                    NewText = $"(%s{erroringExpression})" } |]
                                    Kind = Fix } ]
      | Error _ -> async.Return [])
    (Set.ofList [ "597" ])
```

This kind of easy quick fix can be written becase we have all the information we need right there. However, not every quick fix can be written so easily.

## Harder quick fixer example: scanning the text in a document

Sometimes the error range for a diagnostic isn't enough information to inform a quick fix. But not all is lost! Sometimes all you have to do is scan through a document until you find something that gives you the information you need.

Consider the following error:

```fsharp
[<EntryPoint>]
let main argv =
    // 'argv -1' is an error
    // The range of the error, however, is only 'argv'
    for x = 0 to argv -1 do
        printfn "uuuhhhhh"
```

The compiler will complain because it things you're calling argv as a function and passing `-1` to it. This can happen because `-` is both a binary and a unary operator, and the F# parser parses `-1` as a negation on 1, and the entire text of `-1` as a value being passed to argv. Since argv is not a function, this is obviously not correct.

Because the compiler error's range corresponds to argv, we don't actually have enough information to know that we can place a space between the `-` and the `1`. In fact, based on the error range being only for argv, we don't even know where in source the `-1` is! So we'll not only need to find its location, but also ensure that the next construct that comes after argv is indeed a `-`.

Luckily, this can be done as a recursive function or loop. Here's an example of scanning forward past the span corresponding to the diagnostic using Visual Studio APIs:

```fsharp
let pos = context.Span.End + 1

let nextNonWhitespaceText =
    let rec loop str pos =
        if not (String.IsNullOrWhiteSpace(str)) then
            str
        else
            loop (sourceText.GetSubText(TextSpan(pos + 1, 1))) (pos + 1)
    loop (sourceText.GetSubText(TextSpan(pos, 1))) pos
```

This will grab a span of text that's exactly one character long, check it, and keep going until it's not whitespace. We can then check that nextNonWhitespaceText is equal to `-`. If it is, we can trigger a code fix! Here's how the entire code fixer can look:

```fsharp
namespace Microsoft.VisualStudio.FSharp.Editor

open System
open System.Composition
open System.Threading.Tasks

open Microsoft.CodeAnalysis.Text
open Microsoft.CodeAnalysis.CodeFixes

[<ExportCodeFixProvider(FSharpConstants.FSharpLanguageName, Name = "ChangePrefixNegationToInfixSubtraction"); Shared>]
type internal FSharpChangePrefixNegationToInfixSubtractionodeFixProvider() =
    inherit CodeFixProvider()

    let fixableDiagnosticIds = set ["FS0003"]

    override _.FixableDiagnosticIds = Seq.toImmutableArray fixableDiagnosticIds

    override _.RegisterCodeFixesAsync context : Task =
        asyncMaybe {
            let diagnostics =
                context.Diagnostics
                |> Seq.filter (fun x -> fixableDiagnosticIds |> Set.contains x.Id)
                |> Seq.toImmutableArray

            let! sourceText = context.Document.GetTextAsync(context.CancellationToken)

            // End of 'argv', in the case of the example above
            let pos = context.Span.End + 1

            // This won't ever actually happen, but it's good to check
            do! Option.guard (pos < sourceText.Length)

            let nextNonWhitespaceText =
                let rec loop str pos =
                    if not (String.IsNullOrWhiteSpace(str)) then
                        str
                    else
                        loop (sourceText.GetSubText(TextSpan(pos + 1, 1))) (pos + 1)
                loop (sourceText.GetSubText(TextSpan(pos, 1))) pos

            // Bail if this isn't a negation
            do! Option.guard (nextNonWhitespaceText = "-")

            let title = SR.ChangePrefixNegationToInfixSubtraction()

            let codeFix =
                CodeFixHelpers.createTextChangeCodeFix(
                    title,
                    context,
                    (fun () -> asyncMaybe.Return [| TextChange(TextSpan(pos, 1), "- ") |]))

            context.RegisterCodeFix(codeFix, diagnostics)
        }
        |> Async.Ignore
        |> RoslynHelpers.StartAsyncUnitAsTask(context.CancellationToken)
```

Note that the API calls are slightly different here. There is a helper defined called `createTextChangeCodeFix` that can be used, unlike in the previous example.

## Harder quick fixe example: checking the syntax tree

Now things get a little more challenging. In the previous two examples, we could either work directly with a span of text in a document and change it, or scan the document to find what we need. But what if that's not enough? In some cases, you need to answer a more complicated question that corresponds to the actual struture of F# source code. Consider the following incorrect code:

```fsharp
let f (x: bool) (y: bool) =
    !x && !y
```

Someone without much F# (or OCaml) experience might thing that this is a boolean `not` operation. However, it is not! The `!` operator is used to dereference a [Reference Cell](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/reference-cells). A correct fix would be to use the `not` operator:

```fsharp
let f (x: bool) (y: bool) =
    not x && not y
```

The diagnostic triggers on both `x` and `y` but it does not contain the text or position of !. Although it's possible to scan in a document to find the `!`, there's actually a much better approach: using the F# syntax tree APIs. Instead of relying on potentially error-prone custom scanning code, checking if a span of text is contained in a deference call (using `!`) will always be correct.

This can be trivially accomplished with a type extension on `FSharpParseFileResults`:

```fsharp
open FSharp.Compiler
open FSharp.Compiler.Text
open FSharp.Compiler.Range
open FSharp.Compiler.SourceCodeServices

[<AutoOpen>]
module ParseTreeExtensions =
    type FSharpParseFileResults with
        member scope.TryRangeOfRefCellDereferenceContainingPos expressionPos =
            match scope.ParseTree with
            | Some input ->
                AstTraversal.Traverse(expressionPos, input, { new AstTraversal.AstVisitorBase<_>() with
                    member _.VisitExpr(_, _, defaultTraverse, expr) =
                        match expr with
                        | SynExpr.App(_, false, SynExpr.Ident funcIdent, expr, _) ->
                            if funcIdent.idText = "op_Dereference" && rangeContainsPos expr.Range expressionPos then
                                Some funcIdent.idRange
                            else
                                None
                        | _ -> defaultTraverse expr })
            | None -> None
```

The F# compiler services contain, among other things, a syntax tree visitor that has some default behavior you can override. You still need to implement `VisitExpr`, which is the exact one we're going to work with here.

If it looks complicated, don't worry! It's really not too bad. There is just a bit of terminology to understand:

* "Range" and "Pos", such as in the `rangeContainsPos` call, refer to a range of text in a document (a line/column pair) and a position (a line and a column)
* `SynExpr.App` refers to a function application. All function applications contain a function expression and a argument expression of type `SynExpr`
* `SynExpr.Ident` refers to an identifer in a syntax tree. It has a name (`idText`) and a range (`idRange`)

In this case, the expression `!x` (or any of variant including arbitrary nesting of parentheses or whitespace) is just a `SynExpr.App` where the function expression is a `SynExpr.Ident` with `idText` of `op_Dereference`. So we just need to check that a given position is contained in the range of the argument that is being applied to `!`.

So, how do we call this? That's where a given editor API comes into play. In the case of Visual Studio, we need to convert from a Roslyn-based span of text to an F# compiler-based range of text (note the difference in terminology). Even though they both refer to the same thing, they have slightly different ways of representing the data.

We also need to parse a document to get access to an instance of `FSharpParseFileResults`. So if we refer back to the skeleton source code, the custom logic here is:

1. Parse a document
2. Convert the code fix context's span of text into an F# range
3. Call our extension to `FSharpParseFileResults`
4. Apply a code fix to the `!` if it exists

Here's the full code snippet of the fixer:

```fsharp
namespace Microsoft.VisualStudio.FSharp.Editor

open System.Composition
open System.Threading.Tasks

open Microsoft.CodeAnalysis.Text
open Microsoft.CodeAnalysis.CodeFixes

[<ExportCodeFixProvider(FSharpConstants.FSharpLanguageName, Name = "ChangeRefCellDerefToNotExpression"); Shared>]
type internal FSharpChangeRefCellDerefToNotExpressionCodeFixProvider
    [<ImportingConstructor>]
    (
        checkerProvider: FSharpCheckerProvider,
        projectInfoManager: FSharpProjectOptionsManager
    ) =
    inherit CodeFixProvider()

    static let userOpName = "FSharpChangeRefCellDerefToNotExpressionCodeFix"
    let fixableDiagnosticIds = set ["FS0001"]

    override __.FixableDiagnosticIds = Seq.toImmutableArray fixableDiagnosticIds

    override this.RegisterCodeFixesAsync context : Task =
        asyncMaybe {
            // All of this is setup to be able to parse a document
            let document = context.Document
            let! parsingOptions, _ = projectInfoManager.TryGetOptionsForEditingDocumentOrProject(document, context.CancellationToken, userOpName)
            let! sourceText = context.Document.GetTextAsync(context.CancellationToken)

            // The actual parsing call, which is slightly complex
            let! parseResults = checkerProvider.Checker.ParseFile(document.FilePath, sourceText.ToFSharpSourceText(), parsingOptions, userOpName) |> liftAsync

            // Converting to an F# range
            let errorRange = RoslynHelpers.TextSpanToFSharpRange(document.FilePath, context.Span, sourceText)

            // Getting a range of a dereference operator
            let! derefRange = parseResults.TryRangeOfRefCellDereferenceContainingPos errorRange.Start

            // Converting back into Roslyn-based spans
            let! derefSpan = RoslynHelpers.TryFSharpRangeToTextSpan(sourceText, derefRange)

            let title = SR.UseNotForNegation()

            let diagnostics =
                context.Diagnostics
                |> Seq.filter (fun x -> fixableDiagnosticIds |> Set.contains x.Id)
                |> Seq.toImmutableArray

            let codeFix =
                CodeFixHelpers.createTextChangeCodeFix(
                    title,
                    context,

                    // The actual fix is trivial, just place `!` with `not `
                    (fun () -> asyncMaybe.Return [| TextChange(derefSpan, "not ") |]))

            context.RegisterCodeFix(codeFix, diagnostics)
        }
        |> Async.Ignore
        |> RoslynHelpers.StartAsyncUnitAsTask(context.CancellationToken)
```

And that's it! There's a bit of ceremony to get access to the data we need and to convert back and forth between different textual representations, but after that the actual code fix is trivial.

Harder quick fixer example: analyzing semantics
Finally, you may also need to analyze F# semantics to be able to offer up a quick fix. Some errors that involve typechecking require you to analyze typecheck results to get the information that you're after.

Consider the following code:

```fsharp
let x = 12
x <- 13
```

This will fail to compile because we're trying to mutate `x`, but it isn't declared as `mutable`. I personally run into this all the time because I won't always know that I want to mutate something until I decide it's necessary, then I have to go back and modify the declaration manually. Why not have a quick fixer do that?

To make this quick fixer, we need to now also analyze semantics, because we need to find the declaration location of a given value. Specifically, we'll need to do the following:

1. Find the F# symbol for `x` in the erroneous `x <- 13` call
2. Find the declaration of `x` once we've resolved it at its use
3. Check that it's not a parameter (if it is, we can't declare it as `mutable`)
4. Apply the `mutable` keyword to the declaration of `x`

There's more code involved here than before, much of which is just boilerplate needed to be able to get a declaration of a value. Unfortunately, this boilerplate is fairly complex, so I would not classify this kind of code fix as easy.

This is what the boilerplate needed in Visual Studio to be able to get a declaration looks like, which I've annotated to the best of my ability:

```fsharp
// Just setting up some values and doing a quick check
let document = context.Document
do! Option.guard (not(isSignatureFile document.FilePath))
let checker = checkerProvider.Checker

// This is critical. Use the START of the diagnostic span
let position = context.Span.Start

// Accessing the data that we need to make certain API calls
let! parsingOptions, projectOptions = projectInfoManager.TryGetOptionsForEditingDocumentOrProject(document, CancellationToken.None, userOpName)
let! sourceText = document.GetTextAsync () |> liftTaskAsync
let defines = CompilerEnvironment.GetCompilationDefinesForEditing parsingOptions
let textLine = sourceText.Lines.GetLineFromPosition position
let textLinePos = sourceText.Lines.GetLinePosition position
let fcsTextLineNumber = Line.fromZ textLinePos.Line

// Parse and typecheck a document, getting results for the parsing and typechecking
let! parseFileResults, _, checkFileResults = checker.ParseAndCheckDocument (document, projectOptions, sourceText=sourceText, userOpName=userOpName)

// Build a "lexer symbol" - this will quickly isolate the `x` from the rest of the expression and generate an F# SynExpr.Ident that can be used in other API calls
let! lexerSymbol = Tokenizer.getSymbolAtPosition (document.Id, sourceText, position, document.FilePath, defines, SymbolLookupKind.Greedy, false, false)

// Finally, get the declaration of the symbol that a position corresponds to
let decl = checkFileResults.GetDeclarationLocation (fcsTextLineNumber, lexerSymbol.Ident.idRange.EndColumn, textLine.ToString(), lexerSymbol.FullIsland, false)
It's quite a lot, and we're planning on finding ways to improve F# compiler service APIs to make this kind of boilerplate no longer necessary.
```

Next, we'll also need to detect if the declaration is contained within a parameter or not. We'll need to also have an `FSharpParseFileResults` extension like before:

```fsharp
open FSharp.Compiler
open FSharp.Compiler.Text
open FSharp.Compiler.Range
open FSharp.Compiler.SourceCodeServices

[<AutoOpen>]
module ParseTreeExtensions =
    type FSharpParseFileResults with
        member scope.IsPositionContainedInACurriedParameter pos =
            match input with
            | Some input ->
                let result =
                    AstTraversal.Traverse(pos, input, { new AstTraversal.AstVisitorBase<_>() with 
                        member _.VisitExpr(_path, traverseSynExpr, defaultTraverse, expr) =
                            defaultTraverse(expr)

                        override _.VisitBinding (_, binding) =
                            match binding with
                            | Binding(_, _, _, _, _, _, valData, _, _, _, range, _) when rangeContainsPos range pos ->
                                let info = valData.SynValInfo.CurriedArgInfos
                                let mutable found = false
                                for group in info do
                                    for arg in group do
                                        match arg.Ident with
                                        | Some ident when rangeContainsPos ident.idRange pos ->
                                            found <- true
                                        | _ -> ()
                                if found then Some range else None
                            | _ ->
                                None
                    })
                result.IsSome
            | _ -> false
```

In this case, we just use `defaultTraverse` for any arbitary `SynExpr`, but we override the `VisitBinding` member. `VisitBinding` traverses a `SynExpr.Binding`, which is typicall a let binding. We need to then inspect data called `valData`, which contains a list of all curried parameter definitions for the binding, if they exist. We then loop through each and detect if the given position is within the range of one of the defined parameter bindings. For example, consider the following:

```fsharp
let f (x: int) (y: int) =
    x <- 12 // Error
    y
```

This code will result in `x` being defined as a parameter. So we can pass the start position of its range to the tree traversal, which will then loop through each parameter until it finds the `x` definition. It will verify that the range of `x` contains the position we're after, return true, and then we'll know that `x` is defined as a parameter!

Putting it all together looks like this:

```fsharp
namespace Microsoft.VisualStudio.FSharp.Editor

open System.Composition
open System.Threading
open System.Threading.Tasks

open Microsoft.CodeAnalysis.Text
open Microsoft.CodeAnalysis.CodeFixes

open FSharp.Compiler.Range
open FSharp.Compiler.SourceCodeServices
open FSharp.Compiler.AbstractIL.Internal.Library

[<ExportCodeFixProvider(FSharpConstants.FSharpLanguageName, Name = "MakeDeclarationMutable"); Shared>]
type internal FSharpMakeDeclarationMutableFixProvider
    [<ImportingConstructor>]
    (
        checkerProvider: FSharpCheckerProvider, 
        projectInfoManager: FSharpProjectOptionsManager
    ) =
    inherit CodeFixProvider()

    static let userOpName = "MakeDeclarationMutable"

    let fixableDiagnosticIds = set ["FS0027"]

    override _.FixableDiagnosticIds = Seq.toImmutableArray fixableDiagnosticIds

    override _.RegisterCodeFixesAsync context : Task =
        asyncMaybe {
            let diagnostics =
                context.Diagnostics
                |> Seq.filter (fun x -> fixableDiagnosticIds |> Set.contains x.Id)
                |> Seq.toImmutableArray

            let document = context.Document
            do! Option.guard (not(isSignatureFile document.FilePath))
            let position = context.Span.Start
            let checker = checkerProvider.Checker
            let! parsingOptions, projectOptions = projectInfoManager.TryGetOptionsForEditingDocumentOrProject(document, CancellationToken.None, userOpName)
            let! sourceText = document.GetTextAsync () |> liftTaskAsync
            let defines = CompilerEnvironment.GetCompilationDefinesForEditing parsingOptions
            let textLine = sourceText.Lines.GetLineFromPosition position
            let textLinePos = sourceText.Lines.GetLinePosition position
            let fcsTextLineNumber = Line.fromZ textLinePos.Line
            let! parseFileResults, _, checkFileResults = checker.ParseAndCheckDocument (document, projectOptions, sourceText=sourceText, userOpName=userOpName)
            let! lexerSymbol = Tokenizer.getSymbolAtPosition (document.Id, sourceText, position, document.FilePath, defines, SymbolLookupKind.Greedy, false, false)
            let decl = checkFileResults.GetDeclarationLocation (fcsTextLineNumber, lexerSymbol.Ident.idRange.EndColumn, textLine.ToString(), lexerSymbol.FullIsland, false)

            match decl with
            // Only do this for symbols in the same file. That covers almost all cases anyways.
            // We really shouldn't encourage making values mutable outside of local scopes anyways.
            | FSharpFindDeclResult.DeclFound declRange when declRange.FileName = document.FilePath ->
                let! span = RoslynHelpers.TryFSharpRangeToTextSpan(sourceText, declRange)

                // Bail if it's a parameter, because like, that ain't allowed
                do! Option.guard (not (parseFileResults.IsPositionContainedInACurriedParameter declRange.Start))

                let title = SR.MakeDeclarationMutable()
                let codeFix =
                    CodeFixHelpers.createTextChangeCodeFix(
                        title,
                        context,
                        (fun () -> asyncMaybe.Return [| TextChange(TextSpan(span.Start, 0), "mutable ") |]))

                context.RegisterCodeFix(codeFix, diagnostics)
            | _ ->
                ()
        }
        |> Async.Ignore
        |> RoslynHelpers.StartAsyncUnitAsTask(context.CancellationToken)
```

And that's it!

## Contribute your own code fixer

If you've made it this far, you should be armed to add all kinds of code fixers. There is actually another class of fixer that I can discuss in another blog post, where we pair a code analyzer that raises custom diagnostics with a fixer that acts on those diagnostics. But the contents of this post should be enough to add lots of different kinds of fixers.

If you want to add one to Visual Studio, check out the fixers in the CodeFix folder. You can copy/paste one into a new file and change stuff as you go. Syntax tree extensions are typically moved into the F# compiler API itself, and with corresponding unit tests. But we can help you get that stuff added correctly during code review.

If you want to add one to VSCode, check out the CodeFixes file and take a look at the variety of code fixers available there and add a new one. I advise looking through the git history of the file to see where various helpers, such as syntax tree extensions, are located.

Happy code fixing!