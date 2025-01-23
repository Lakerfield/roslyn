# Add a new `in` expression to C#

## Goals

In this workshop, we'll add a new `in` expression to C#, with example use cases shown below:

```csharp
x in 1..10     // x >= 1 && x < 10
'a' in "bar"   // "bar".IndexOf('a') >= 0   -or-   "bar".Contains('a')
x in xs        // xs.Contains(x)
```

## Prerequisites

1. Visual Studio 2022
2. GitHub account
3. Local installation of git (check `git --version`)
4. [.NET 7.0 RC1](https://dotnet.microsoft.com/en-us/download/dotnet/7.0) (check `dotnet --version`)
5. [ILSpy](https://github.com/icsharpcode/ILSpy/releases)

## Fork and clone the repo

1. Fork github.com/dotnet/roslyn to your own account
2. Clone your fork using `git clone https://github.com/user/roslyn`
3. Open the working folder using `cd roslyn`
4. Create a new branch using `git checkout -b InExpression`

## Clean build of Roslyn

1. Restore dependencies using `restore.cmd`
2. Build Roslyn using `build.cmd`

## Prepare Visual Studio solution

1. Open `Compilers.slnf`
2. Unload every project except for
   1. `Compilers/Core/Microsoft.CodeAnalysis`
   2. `Compilers/CSharp/Microsoft.CodeAnalysis.CSharp`
   3. `Compilers/CSharp/csc`
   4. `Dependencies/Microsoft.CodeAnalysis.Collections`
   5. `Dependencies/Microsoft.CodeAnalysis.PooledObjects`
3. Save the solution
4. Build the solution using CTRL-SHIFT-B

The result of step 3 should be a `Compilers.slnf` file containing:

```json
{
  "solution": {
    "path": "Roslyn.sln",
    "projects": [
      "src\\Compilers\\CSharp\\Portable\\Microsoft.CodeAnalysis.CSharp.csproj",
      "src\\Compilers\\CSharp\\csc\\AnyCpu\\csc.csproj",
      "src\\Compilers\\Core\\Portable\\Microsoft.CodeAnalysis.csproj",
      "src\\Dependencies\\Collections\\Microsoft.CodeAnalysis.Collections.shproj",
      "src\\Dependencies\\PooledObjects\\Microsoft.CodeAnalysis.PooledObjects.shproj"
    ]
  }
}
```

## Prepare a test folder

1. `mkdir c:\temp\InExpression`
2. `cd c:\temp\InExpression`
3. Create a new `test.cs` file

   ```csharp
   using System;
   
   bool b = 5 in 0..10;
   
   Console.WriteLine(b);
   ```
3. Copy prerequisite files from the .NET SDK for simplicity (see later)

   ```cmd
   copy /Y "C:\Program Files\dotnet\packs\Microsoft.NETCore.App.Ref\7.0.0-rc.1.22426.10\ref\net7.0\mscorlib.dll" .
   copy /Y "C:\Program Files\dotnet\packs\Microsoft.NETCore.App.Ref\7.0.0-rc.1.22426.10\ref\net7.0\netstandard.dll" .
   copy /Y "C:\Program Files\dotnet\packs\Microsoft.NETCore.App.Ref\7.0.0-rc.1.22426.10\ref\net7.0\System.Console.dll" .
   copy /Y "C:\Program Files\dotnet\packs\Microsoft.NETCore.App.Ref\7.0.0-rc.1.22426.10\ref\net7.0\System.dll" .
   copy /Y "C:\Program Files\dotnet\packs\Microsoft.NETCore.App.Ref\7.0.0-rc.1.22426.10\ref\net7.0\System.Runtime.dll" .
   copy /Y "C:\Program Files\dotnet\packs\Microsoft.NETCore.App.Ref\7.0.0-rc.1.22426.10\ref\net7.0\System.Collections.dll" .
   ```

## Set `csc` as the startup project

1. Right-click on the `csc` project and select `Set as Startup Project`
2. Right-click on the `csc` project and select `Properties`
3. Under `Debug`, `General`, click `Open debug launch profiles UI`
4. Specify the following for `Command line arguments`
   ```
   /noconfig /fullpaths /nostdlib+
   /reference:mscorlib.dll
   /reference:netstandard.dll
   /reference:System.Console.dll
   /reference:System.dll
   /reference:System.Runtime.dll
   /reference:System.Collections.dll
   /out:test.dll /optimize- test.cs
   ```
5. Set `Working directory` to `C:\temp\InExpression`
6. Close the dialog
7. Press CTRL-F5 to build and run
8. The output of `csc` should produce errors because `in` is not valid (yet)

   ```
   Microsoft (R) Visual C# Compiler version 4.5.0-dev (<developer build>)
   Copyright (C) Microsoft Corporation. All rights reserved.

   C:\temp\InExpression\test.cs(3,12): error CS1003: Syntax error, ',' expected

   C:\Program Files\dotnet\dotnet.exe (process 12020) exited with code 1.
   ```
   
## Step 1 - Syntax

1. Open the `src\Compilers\CSharp\Portable\Syntax\Syntax.xml` file.

2. Search for `BinaryExpressionSyntax`:

   ```xml
   <Node Name="BinaryExpressionSyntax" Base="ExpressionSyntax">
   ```
   
   and add a new `Kind` value:

   ```xml
   <Kind Name="InExpression"/>
   ```
   
3. Search for the `OperatorToken` element underneath the `Node` for `BinaryExpressionSyntax`:

   ```xml
   <Field Name="OperatorToken" Type="SyntaxToken">
   ```
   
   and add a new `Kind` value:
   
   ```xml
   <Kind Name="InKeyword"/>
   ```
   
4. Compile the project using CTRL-SHIFT-B. A few errors will occur:

   ```
   Error CS0117 'SyntaxKind' does not contain a definition for 'InExpression'	Microsoft.CodeAnalysis.CSharp src\Compilers\CSharp\Portable\CSharpSyntaxGenerator\CSharpSyntaxGenerator.SourceGenerator\Syntax.xml.Internal.Generated.cs
   ```
   
5. Open the `src\Compilers\CSharp\Portable\Syntax\SyntaxKind.cs` file.

6. Search for `UnsignedRightShiftExpression`:

   ```csharp
   UnsignedRightShiftExpression = 8692,
   ```
   
   and add a new enum value for `InExpression`:
   
   ```csharp
   InExpression = 8693,
   ```
   
7. A red squiggle will occur underneath `InExpression` with error:

   ```
   Error RS0016 Symbol 'InExpression' is not part of the declared API
   ```
   
8. Fix the error by using CTRL+. to invoke the `Add InExpression to public API file PublicAPI.Unshipped.txt`:

   ```
   Microsoft.CodeAnalysis.CSharp.SyntaxKind.InExpression = 8693 -> Microsoft.CodeAnalysis.CSharp.SyntaxKind
   ```
   
9. Build the project again using CTRL-SHIFT-B.

10. Run our new build of the compiler with F5. The same errors still occur:

   ```
   C:\temp\InExpression\test.cs(3,12): error CS1003: Syntax error, ',' expected
   ```
   
11. Open the `src\Compilers\CSharp\Portable\Syntax\SyntaxKindFacts.cs` file.

12. Search for `GetBinaryExpression`:

   ```csharp
   public static SyntaxKind GetBinaryExpression(SyntaxKind token)
   ```
   
   and add a case to the `switch (token)` statement:
   
   ```csharp
   case SyntaxKind.InKeyword:
       return SyntaxKind.InExpression;
   ```
   
13. Build the project again using CTRL-SHIFT-B.

14. Run our new build of the compiler with F5. This time we cause an assert in `LanguageParser.cs`:

   ```csharp
   throw ExceptionUtilities.UnexpectedValue(op);
   ```
   
   with stack trace:
   
   ```
   Microsoft.CodeAnalysis.dll!Roslyn.Utilities.ExceptionUtilities.UnexpectedValue(object o)
      Microsoft.CodeAnalysis.CSharp.dll!Microsoft.CodeAnalysis.CSharp.Syntax.InternalSyntax.LanguageParser.GetPrecedence(Microsoft.CodeAnalysis.CSharp.SyntaxKind op)
   ...
   ```

## Step 2 - Parser

1. Open the `src\Compilers\CSharp\Portable\Parser\LanguageParser.cs` file.

2. Search for `GetPrecedence`:

   ```csharp
   private static Precedence GetPrecedence(SyntaxKind op)
   ```
   
3. Search for `Precedence.Relational`:

   ```csharp
   case SyntaxKind.LessThanExpression:
   case SyntaxKind.LessThanOrEqualExpression:
   case SyntaxKind.GreaterThanExpression:
   case SyntaxKind.GreaterThanOrEqualExpression:
   case SyntaxKind.IsExpression:
   case SyntaxKind.AsExpression:
   case SyntaxKind.IsPatternExpression:
       return Precedence.Relational;
   ```
   
   and add a case for `InExpression`:
   
   ```csharp
   ...
   case SyntaxKind.IsPatternExpression:
   case SyntaxKind.InExpression:
       return Precedence.Relational;
   ```
   
4. Build the project again using CTRL-SHIFT-B.

5. Run our new build of the compiler with F5. This time we cause an assert in `Binder_Expression.cs`:

   ```csharp
   Debug.Assert(false, "Unexpected SyntaxKind " + node.Kind());
   ```
   
   in:
   
   ```
   Microsoft.CodeAnalysis.CSharp.dll!Microsoft.CodeAnalysis.CSharp.Binder.BindExpressionInternal(Microsoft.CodeAnalysis.CSharp.Syntax.ExpressionSyntax node, Microsoft.CodeAnalysis.CSharp.BindingDiagnosticBag diagnostics, bool invoked, bool indexed)
   ```

## Step 3 - Bound nodes (first pass)

1. Open the `src\Compilers\CSharp\Portable\BoundTree\BoundNodes.xml` file.

2. Search for `BoundIsOperator`:

   ```xml
   <Node Name="BoundIsOperator" Base="BoundExpression">
       ...
   </Node>
   ```

3. We will insert a new `Node` element underneath:

   ```xml
   <Node Name="BoundInOperator" Base="BoundExpression">
     <!-- Non-null type is required for this node kind -->
     <Field Name="Type" Type="TypeSymbol" Override="true" Null="disallow"/>

     <Field Name="Element" Type="BoundExpression"/>
     <Field Name="Source" Type="BoundExpression"/>
   </Node>
   ```
   
4. Run the following command from the root of the `roslyn` cloned repo:

   ```
   artifacts\bin\CompilersBoundTreeGenerator\x64\Debug\net6.0\BoundTreeGenerator.exe CSharp src\Compilers\CSharp\Portable\BoundTree\BoundNodes.xml src\Compilers\CSharp\Portable\Generated\BoundNodes.xml.Generated.cs
   ```
   
5. Build the project again using CTRL-SHIFT-B.

6. Open the `src\Compilers\CSharp\Portable\Binder\Binder_Expressions.cs` file.

7. Locate the place where the assert happened:

   ```csharp
   default:
       ...
       Debug.Assert(false, "Unexpected SyntaxKind " + node.Kind());
   ```
   
   and insert a case for `SyntaxKind.InExpression` above:
   
   ```csharp
   case SyntaxKind.InExpression:
       return BindInExpression((BinaryExpressionSyntax)node, diagnostics);
   ```
   
8. Open the `src\Compilers\CSharp\Portable\Binder\Binder_Operators.cs` file.

9. Add a new placeholder `BindInExpression` method at the bottom:

   ```csharp
   private BoundExpression BindInExpression(BinaryExpressionSyntax node, BindingDiagnosticBag diagnostics)
   {
       throw new NotImplementedException();
   }
   ```
   
10. Build the project again using CTRL-SHIFT-B.

11. Run our new build of the compiler with F5. This time we hit our `NotImplementedException`.

## Step 4 - Binder (first pass)

1. Edit the `BindInExpression` to add recursion.

   ```csharp
   BoundExpression leftOperand = BindRValueWithoutTargetType(node.Left, diagnostics);
   BoundExpression rightOperand = BindRValueWithoutTargetType(node.Right, diagnostics);
   ```
   
2. Add top-level matching logic for the structure and types of the operands:

   ```csharp
   if (rightOperand is BoundRangeExpression)
   {
       throw new NotImplementedException();
   }
   else
   {
       if (rightOperand.Type is ArrayTypeSymbol { IsSZArray: true })
       {
           throw new NotImplementedException();
       }
       else if (rightOperand.Type.IsStringType())
       {
           throw new NotImplementedException();
       }
       else
       {
           throw new NotImplementedException();
       }
   }
   ```
   
3. Build the project again using CTRL-SHIFT-B.

4. Run our new build of the compiler with F5. This time we should see our first case being taken.

   ```csharp
   5 in 0..10
   ```
   
   where the right operand is a `BoundRangeExpression`.
   
5. Edit this case to add an initial implementation.

   ```csharp
   var booleanType = Compilation.GetSpecialType(SpecialType.System_Boolean);
   
   if (rightOperand is BoundRangeExpression range)
   {
       if (leftOperand.Type?.SpecialType != SpecialType.System_Int32)
       {
           throw new NotImplementedException();
       }

       void checkRangeOperand(BoundExpression? operand)
       {
           if (operand != null)
           {
               if (operand is not BoundConversion { Operand.Type.SpecialType: SpecialType.System_Int32 })
               {
                   throw new NotImplementedException();
               }
           }
       }

       checkRangeOperand(range.LeftOperandOpt);
       checkRangeOperand(range.RightOperandOpt);
       return new BoundInOperator(node, leftOperand, rightOperand, booleanType);
   }
   ...
   ```
   
6. Build the project again using CTRL-SHIFT-B.

7. Run our new build of the compiler with F5. This time we hit an assert in `AbstractFlowPass.cs`:

   ```csharp
   public override BoundNode DefaultVisit(BoundNode node)
   {
       Debug.Assert(false, $"Should Visit{node.Kind} be overridden in {this.GetType().Name}?");
       Diagnostics.Add(ErrorCode.ERR_InternalError, node.Syntax.Location);
       return null;
   }
   ```
   
   with a call stack that mentions `NullableWalker`:
   
   ```
   Microsoft.CodeAnalysis.CSharp.dll!Microsoft.CodeAnalysis.CSharp.NullableWalker.VisitExpressionWithoutStackGuard(Microsoft.CodeAnalysis.CSharp.BoundExpression node)
   ```
   
## Step 5 - Fix `NullableWalker`

1. Open the `src\Compilers\CSharp\Portable\FlowAnalysis\NullableWalker.cs file`.

2. Search for `VisitIsOperator`:

   ```csharp
   public override BoundNode? VisitIsOperator(BoundIsOperator node)
   {
       ...
   }
   ```

   and insert the following below:
   
   ```csharp
   public override BoundNode VisitInOperator(BoundInOperator node)
   {
       SetNotNullResult(node);
       return null!;
   }
   ```
   
3. Build the project again using CTRL-SHIFT-B.

4. Run our new build of the compiler with F5. This time we hit the same assert in `AbstractFlowPass.cs`:

   ```csharp
   public override BoundNode DefaultVisit(BoundNode node)
   {
       Debug.Assert(false, $"Should Visit{node.Kind} be overridden in {this.GetType().Name}?");
       Diagnostics.Add(ErrorCode.ERR_InternalError, node.Syntax.Location);
       return null;
   }
   ```
   
   but with a call stack that mentions `DefiniteAssignmentPass`:
   
   ```
    	Microsoft.CodeAnalysis.CSharp.dll!Microsoft.CodeAnalysis.CSharp.DefiniteAssignmentPass.VisitRvalue(Microsoft.CodeAnalysis.CSharp.BoundExpression node, bool isKnownToBeAnLvalue)
   ```
   
   The right fix will be to override the default visitor behavior for `BoundInExpression`.
   
## Step 6 - Fix `AbstractFlowPass`

1. Open the `src\Compilers\CSharp\Portable\FlowAnalysis\AbstractFlowPass.cs` file.

2. Search for `VisitIsOperator`:

   ```csharp
   public override BoundNode VisitIsOperator(BoundIsOperator node)
   {
       ...
   }
   ```
   
   and insert the following below:
   
   ```csharp
   public override BoundNode VisitInOperator(BoundInOperator node)
   {
       VisitRvalue(node.Element);
       VisitRvalue(node.Source);
       return null;
   }
   ```

3. Build the project again using CTRL-SHIFT-B.

4. Run our new build of the compiler with F5. This time we hit an assert in `EmitExpression.cs`:

   ```csharp
   throw ExceptionUtilities.UnexpectedValue(expression.Kind);
   ```
   
   with a callstack that mentions `EmitExpression`:
   
   ```
      Microsoft.CodeAnalysis.CSharp.dll!Microsoft.CodeAnalysis.CSharp.CodeGen.CodeGenerator.EmitExpression(Microsoft.CodeAnalysis.CSharp.BoundExpression expression, bool used)
   ```
   
   To fix this, we will lower the `BoundInExpression` to more primitive operations.
   
## Step 7 - Lowering (first pass)

1. Open the `src\Compilers\CSharp\Portable\Lowering\LocalRewriter\LocalRewriter_BinaryOperator.cs` file.

2. Add the end of the file, add an override for `VisitInOperator`.

   ```csharp
   public override BoundNode? VisitInOperator(BoundInOperator node)
   {
       throw new NotImplementedException();
   }
   ```
   
3. Build the project again using CTRL-SHIFT-B.

4. Run our new build of the compiler with F5. This time we hit our `NotImplementedException`.

5. Edit the implementation of `VisitInOperator` to handle `BoundRangeExpression`.

   ```csharp
   if (node.Source is BoundRangeExpression range)
   {
      var element = VisitExpression(node.Element);

      BoundExpression? expr = null;

      if (range.LeftOperandOpt is BoundConversion { Operand: var left })
      {
          var l = VisitExpression(left);
          expr = _factory.IntLessThanOrEqual(l, element);
      }

      if (range.RightOperandOpt is BoundConversion { Operand: var right })
      {
          var r = VisitExpression(right);
          var check = _factory.IntLessThan(element, right);

          if (expr == null)
          {
              expr = check;
          }
          else
          {
              expr = _factory.LogicalAnd(expr, check);
          }
      }

      expr ??= _factory.Literal(true);

      return expr;
   }
   else
   {
       throw new NotImplementedException();
   }
   ```
   
6. Open the `src\Compilers\CSharp\Portable\Lowering\SyntheticBoundNodeFactory.cs` file.

7. Search for `IntLessThan`:

   ```csharp
   public BoundBinaryOperator IntLessThan(BoundExpression left, BoundExpression right)
   ```
   
   and add another factory below:
   
   ```csharp
   public BoundBinaryOperator IntLessThanOrEqual(BoundExpression left, BoundExpression right)
   {
       return Binary(BinaryOperatorKind.IntLessThanOrEqual, SpecialType(Microsoft.CodeAnalysis.SpecialType.System_Boolean), left, right);
   }
   ```
   
8. Build the project again using CTRL-SHIFT-B.

9. Run our new build of the compiler with F5. This time it doesn't crash.

## Step 8 - Inspect the output

1. Open ILSpy.

2. Open the `c:\temp\InExpression\test.dll` file.

3. Under `-`, `Program`, `<Main>$(string[] args)` we'll find:

   ```csharp
   // Program
   using System;

   private static void <Main>$(string[] args)
   {
      bool value = 0 <= 5 && 5 < 10;
      Console.WriteLine(value);
   }
   ```
   
## Step 9 - Edit the test code

1. Open the `c:\test\InExpression\test.cs` file.

2. Edit the contents:

   ```csharp
   using System;
   
   int ReadInt(string name)
   {
       Console.Write(name + ": ");
       return int.Parse(Console.ReadLine());
   }
   
   bool b = ReadInt("x") in ReadInt("start")..ReadInt("end");
   
   Console.WriteLine(b);
   ```
   
3. Re-run the Roslyn project using F5.

4. Inspect the output in ILSpy:

   ```csharp
   // Program
   using System;

   private static void <Main>$(string[] args)
   {
      bool value = ReadInt("start") <= ReadInt("x") && ReadInt("x") < ReadInt("end");
      Console.WriteLine(value);
      static int ReadInt(string name)
      {
         Console.Write(name + ": ");
         return int.Parse(Console.ReadLine());
      }
   }
   ```
   
5. Note the following problems in the generated code:
   1. Duplicate evaluation of `ReadInt("x")`;
   2. Potentially not evaluating `ReadInt("end")` if the `<=` check does not pass.

## Step 10 - Lowering (revisiting)

1. Edit the `VisitInOperator` from step 7 as follows:

   ```csharp
   var temps = ImmutableArray.CreateBuilder<LocalSymbol>();
   var stores = ImmutableArray.CreateBuilder<BoundExpression>();

   BoundLocal storeToTemp(BoundExpression expression)
   {
       var local = _factory.StoreToTemp(expression, out var store);
       temps.Add(local.LocalSymbol);
       stores.Add(store);
       return local;
   }

   BoundExpression? expr = null;

   if (node.Source is BoundRangeExpression range)
   {
       var element = storeToTemp(VisitExpression(node.Element));

       if (range.LeftOperandOpt is BoundConversion { Operand: var left })
       {
           var l = VisitExpression(left);
           expr = _factory.IntLessThanOrEqual(storeToTemp(l), element);
       }

       if (range.RightOperandOpt is BoundConversion { Operand: var right })
       {
           var r = VisitExpression(right);
           var check = _factory.IntLessThan(element, storeToTemp(r));

           if (expr == null)
           {
               expr = check;
           }
           else
           {
               expr = _factory.LogicalAnd(expr, check);
           }
       }

       expr ??= _factory.Literal(true);
   }
   else
   {
       throw new NotImplementedException();
   }

   return _factory.Sequence(temps.ToImmutableArray(), stores.ToImmutableArray(), expr);
   ```
   
   We're adding a `storeToTemp` local function, and use it to generate temporary variables and assignments to these variables, which are put together in a `Sequence` at the end.
   
2. Build the project again using CTRL-SHIFT-B.

3. Run our new build of the compiler with F5.

4. Inspect the output in ILSpy:

   ```csharp
   // Program
   using System;

   private static void <Main>$(string[] args)
   {
      int num = ReadInt("x");
      int num2 = ReadInt("start");
      int num3 = ReadInt("end");
      bool value = num2 <= num && num < num3;
      Console.WriteLine(value);
      static int ReadInt(string name)
      {
         Console.Write(name + ": ");
         return int.Parse(Console.ReadLine());
      }
   }
   ```

## Step 11 - Bound nodes (revisited)

In order to support other forms of `in` involving strings, arrays, and collections, we like to perform lowering like this:

- With `char c;` and `string str;`:
  - Lower `c in str`
  - to `str.IndexOf(c) >= 0`
- With `int x;` and `int[] xs;`:
  - Lower `x in xs`
  - to `Array.IndexOf(x) >= 0`
- With `E element;` and `S source;`:
  - Lower `element in source`
  - to `source.Contains(element)`

To achieve this, we want the binder to resolve the operations required to evaluate the `in` expression based on the type of the element and the source. To represent the Boolean expression that will be applied to the lowered form of source and element, we introduce placeholders:

```csharp
expr_element in expr_source
```

becomes

```csharp
BoundInOperatorSourcePlaceholder s = ...;
BoundInOperatorElementPlaceholder e = ...;

new BoundInOperator
{
    Source = expr_source,
    Element = expr_element,
    
    SourcePlaceholder = s,
    ElementPlaceholder = e,
    Test = /* any expression using s and e */
}
```

where the `Test` can be any expression that refers to the placeholders `s` and `e`, e.g. to construct a `s.IndexOf(e) >= 0` expression. At the time the lowering is performed, this `Test` is inlined with the placeholders being substituted by the lowered element and source expressions.

1. Open the `src\Compilers\CSharp\Portable\BoundTree\BoundNodes.xml` file.

2. Search for the `BoundInOperator` and insert two placeholder types in front:

   ```xml
   <Node Name="BoundInOperatorElementPlaceholder" Base="BoundValuePlaceholderBase">
     <Field Name="Type" Type="TypeSymbol" Override="true" Null="disallow"/>
   </Node>

   <Node Name="BoundInOperatorSourcePlaceholder" Base="BoundValuePlaceholderBase">
     <Field Name="Type" Type="TypeSymbol" Override="true" Null="disallow"/>
   </Node>
   ```
   
3. Edit the `BoundInOperator` to use the new placeholders:

   ```xml
   <Node Name="BoundInOperator" Base="BoundExpression">
     <!-- Non-null type is required for this node kind -->
     <Field Name="Type" Type="TypeSymbol" Override="true" Null="disallow"/>

     <Field Name="Element" Type="BoundExpression"/>
     <Field Name="Source" Type="BoundExpression"/>

     <Field Name="ElementPlaceholder" Type="BoundInOperatorElementPlaceholder?" SkipInVisitor="true" Null="allow"/>
     <Field Name="SourcePlaceholder" Type="BoundInOperatorSourcePlaceholder?" SkipInVisitor="true" Null="allow"/>

     <Field Name="Test" Type="BoundExpression?" SkipInVisitor="true" Null="allow"/>
   </Node>
   ```

4. Run the following command from the root of the `roslyn` cloned repo:

   ```
   artifacts\bin\CompilersBoundTreeGenerator\x64\Debug\net6.0\BoundTreeGenerator.exe CSharp src\Compilers\CSharp\Portable\BoundTree\BoundNodes.xml src\Compilers\CSharp\Portable\Generated\BoundNodes.xml.Generated.cs
   ```
   
5. Build the project again using CTRL-SHIFT-B. A few errors will occur:

   ```
   Error CS0534  'BoundInOperatorSourcePlaceholder' does not implement inherited abstract member 'BoundValuePlaceholderBase.IsEquivalentToThisReference.get'
   Error CS0534  'BoundInOperatorElementPlaceholder' does not implement inherited abstract member 'BoundValuePlaceholderBase.IsEquivalentToThisReference.get'
   Error CS7036  There is no argument given that corresponds to the required formal parameter 'sourcePlaceholder' of 'BoundInOperator.BoundInOperator(SyntaxNode, BoundExpression, BoundExpression, BoundInOperatorElementPlaceholder?, BoundInOperatorSourcePlaceholder?, BoundExpression?, TypeSymbol, bool)'
   ```
   
6. We'll fix the first two errors by opening `src\Compilers\CSharp\Portable\BoundTree\BoundExpression.cs` and search for `BoundThisReference`. Insert the following code:

   ```csharp
   internal partial class BoundInOperatorSourcePlaceholder
   {
       public sealed override bool IsEquivalentToThisReference => false;
   }

   internal partial class BoundInOperatorElementPlaceholder
   {
       public sealed override bool IsEquivalentToThisReference => false;
   }
   ```
   
7. For the last error, edit `BindInExpression` in the `Binder_Operators.cs` file and change:

   ```csharp
   if (rightOperand is BoundRangeExpression)
   {
       var booleanType = Compilation.GetSpecialType(SpecialType.System_Boolean);

       return new BoundInOperator(node, leftOperand, rightOperand, booleanType, hasErrors: false);
   }
   ```
   
   to
   
   ```csharp
   if (rightOperand is BoundRangeExpression)
   {
       var booleanType = Compilation.GetSpecialType(SpecialType.System_Boolean);

       return new BoundInOperator(node, leftOperand, rightOperand, elementPlaceholder: null, sourcePlaceholder: null, test: null, booleanType, hasErrors: false);
   }
   ```
   
8. Build the project again using CTRL-SHIFT-B. The build should pass now.

## Step 12 - Binder (second pass)

1. Open the `src\Compilers\CSharp\Portable\Binder\Binder_Operators.cs` file.

2. Edit the `BindInExpression` method:

   ```csharp
   private BoundExpression BindInExpression(BinaryExpressionSyntax node, BindingDiagnosticBag diagnostics)
   ```
   
   1. First, change the structure of the code to handle two cases, range versus others, using placeholders for the latter:
   
      ```csharp
      var booleanType = Compilation.GetSpecialType(SpecialType.System_Boolean);
      
      BoundInOperatorElementPlaceholder? elementPlaceholder = null;
      BoundInOperatorSourcePlaceholder? sourcePlaceholder = null;
      BoundExpression? test = null;
      
      if (rightOperand is BoundRangeExpression range)
      {
          if (leftOperand.Type?.SpecialType != SpecialType.System_Int32)
          {
              throw new NotImplementedException();
          }

          void checkRangeOperand(BoundExpression? operand)
          {
              if (operand != null)
              {
                  if (operand is not BoundConversion { Operand.Type.SpecialType: SpecialType.System_Int32 })
                  {
                      throw new NotImplementedException();
                  }
              }
          }

          checkRangeOperand(range.LeftOperandOpt);
          checkRangeOperand(range.RightOperandOpt);
      }
      else
      {
          elementPlaceholder = new(node, leftOperand.Type);
          sourcePlaceholder = new(node, rightOperand.Type);

          if (rightOperand.Type is ArrayTypeSymbol { IsSZArray: true })
          {
              throw new NotImplementedException();
          }
          else if (rightOperand.Type.IsStringType())
          {
              throw new NotImplementedException();
          }
          else
          {
              throw new NotImplementedException();
          }
      }
      
      return new BoundInOperator(node, leftOperand, rightOperand, elementPlaceholder, sourcePlaceholder, test, booleanType);
      ```
      
   2. Edit the case handling arrays as follows:
   
      ```csharp
      var int32Type = GetSpecialType(SpecialType.System_Int32, diagnostics, node);

      if (rightOperand.Type is ArrayTypeSymbol { IsSZArray: true, ElementType: var elementType })
      {
          if (!elementType.Equals(leftOperand.Type, TypeCompareKind.ConsiderEverything))
          {
              throw new NotImplementedException();
          }

          var arrayType = GetSpecialType(SpecialType.System_Array, diagnostics, node);

          test =
              new BoundBinaryOperator(
                  node,
                  BinaryOperatorKind.GreaterThanOrEqual,
                  data: null,
                  LookupResultKind.Viable,
                  left: MakeInvocationExpression(
                      node,
                      new BoundTypeExpression(node, aliasOpt: null, arrayType),
                      "IndexOf",
                      ImmutableArray.Create<BoundExpression>(
                          sourcePlaceholder,
                          elementPlaceholder
                      ),
                      diagnostics
                  ),
                  right: new BoundLiteral(
                      node,
                      ConstantValue.Create(0),
                      int32Type
                  ),
                  booleanType
              );
      }
      ```
      
3. Build the project again using CTRL-SHIFT-B. The build should pass.

## Step 13 - Lowering (revisiting, again)

1. Edit the `VisitInOperator` from step 7 by changing the `else` case:

   ```csharp
   else
   {
       var element = storeToTemp(VisitExpression(node.Element));
       var source = storeToTemp(VisitExpression(node.Source));

       AddPlaceholderReplacement(node.ElementPlaceholder, element);
       AddPlaceholderReplacement(node.SourcePlaceholder, source);

       var test = VisitExpression(node.Test);

       RemovePlaceholderReplacement(node.SourcePlaceholder);
       RemovePlaceholderReplacement(node.ElementPlaceholder);

       expr = test;
   }
   ```
   
2. Add the following overrides to the class:

   ```csharp
   public override BoundNode? VisitInOperatorElementPlaceholder(BoundInOperatorElementPlaceholder node) => PlaceholderReplacement(node);
   public override BoundNode? VisitInOperatorSourcePlaceholder(BoundInOperatorSourcePlaceholder node) => PlaceholderReplacement(node);
   ```
   
3. Build the project again using CTRL-SHIFT-B.

4. Edit `c:\temp\InExpression\test.cs`:

   ```csharp
   bool b = ReadInt("x") in new int[] { ReadInt("x1"), ReadInt("x2") };
   ```
   
5. Re-run the Roslyn project using F5. A crash occurs in `LocalRewriter.cs`:

   ```csharp
   Debug.Assert(expr is not BoundValuePlaceholderBase, $"Placeholder kind {expr.Kind} must be handled explicitly");
   ```
   
6. Edit the code to handle the new placeholders:

   ```csharp
   case BoundKind.InOperatorElementPlaceholder:
   case BoundKind.InOperatorSourcePlaceholder:
       return false;
   ```
   
   A better implementation would consider the possiblity of passing these by reference to a `Contains` method when `in` is applied to arbitrary source and element types.
   
7. Build the project again using CTRL-SHIFT-B.

8. Re-run the Roslyn project using F5.

9. Inspect the output in ILSpy:

   ```csharp
   // Program
   using System;

   private static void <Main>$(string[] args)
   {
      int value = ReadInt("x");
      bool value2 = Array.IndexOf(new int[2]
      {
         ReadInt("x1"),
         ReadInt("x2")
      }, value) >= 0;
      Console.WriteLine(value2);
      static int ReadInt(string name)
      {
         Console.Write(name + ": ");
         return int.Parse(Console.ReadLine());
      }
   }
   ```
   
## Step 14 - Binder (remaining cases)

1. Open the `src\Compilers\CSharp\Portable\Binder\Binder_Operators.cs` file.

2. Edit the `BindInExpression` method:

   1. Abstract over the `IndexOf` and `>= 0` checking logic by lifting it out of the array case.
   
      ```csharp
      else
      {
          elementPlaceholder = new(node, leftOperand.Type);
          sourcePlaceholder = new(node, rightOperand.Type);

          var int32Type = GetSpecialType(SpecialType.System_Int32, diagnostics, node);

          BoundExpression makeIndexOfGreaterThanZero(BoundExpression receiver, ImmutableArray<BoundExpression> arguments)
          {
              return
                  new BoundBinaryOperator(
                      node,
                      BinaryOperatorKind.GreaterThanOrEqual,
                      data: null,
                      LookupResultKind.Viable,
                      left: MakeInvocationExpression(
                          node,
                          receiver,
                          "IndexOf",
                          arguments,
                          diagnostics
                      ),
                      right: new BoundLiteral(
                          node,
                          ConstantValue.Create(0),
                          int32Type
                      ),
                      booleanType
                  );
          }

          if (rightOperand.Type is ArrayTypeSymbol { IsSZArray: true, ElementType: var elementType })
          {
              if (!elementType.Equals(leftOperand.Type, TypeCompareKind.ConsiderEverything))
              {
                  throw new NotImplementedException();
              }

              var arrayType = GetSpecialType(SpecialType.System_Array, diagnostics, node);

              test = makeIndexOfGreaterThanZero(
                  new BoundTypeExpression(node, aliasOpt: null, arrayType),
                  ImmutableArray.Create<BoundExpression>(
                      sourcePlaceholder,
                      elementPlaceholder
                  )
              );
          }
          ...
      ```
      
   2. Implement the `string` case as follows:
   
      ```csharp
       else if (rightOperand.Type.IsStringType())
       {
           if (leftOperand.Type?.SpecialType != SpecialType.System_Char)
           {
               throw new NotImplementedException();
           }

           test = makeIndexOfGreaterThanZero(
               sourcePlaceholder,
               ImmutableArray.Create<BoundExpression>(
                   elementPlaceholder
               )
           );
       }
      ```
      
   3. Implement the remaining case to find a `Contains` method:
   
      ```csharp
      else
      {
          test = MakeInvocationExpression(
              node,
              sourcePlaceholder,
              "Contains",
              ImmutableArray.Create<BoundExpression>(
                  elementPlaceholder
              ),
              diagnostics
          );
      }
      ```

## Step 15 - Test

1. Edit the `c:\temp\InExpression\test.cs` file:

   ```csharp
   using System;
   using System.Collections.Generic;

   int ReadInt(string name)
   {
       Console.Write(name + ": ");
       return int.Parse(Console.ReadLine());
   }

   bool b0 = ReadInt("x") in ReadInt("start")..ReadInt("end");
   bool b1 = ReadInt("x") in new int[] { ReadInt("x1"), ReadInt("x2") };
   bool b2 = ReadInt("x") in new List<int> { ReadInt("x1"), ReadInt("x2") };
   bool b3 = 'a' in "Bart";
   ```
   
2. Run the Roslyn project using F5.

3. Inspect the output using ILSpy:

   ```csharp
   // Program
   using System;
   using System.Collections.Generic;

   private static void <Main>$(string[] args)
   {
      int num = ReadInt("x");
      int num2 = ReadInt("start");
      int num3 = ReadInt("end");
      bool flag = num2 <= num && num < num3;
      num3 = ReadInt("x");
      bool flag2 = Array.IndexOf(new int[2]
      {
         ReadInt("x1"),
         ReadInt("x2")
      }, num3) >= 0;
      num3 = ReadInt("x");
      bool flag3 = new List<int>
      {
         ReadInt("x1"),
         ReadInt("x2")
      }.Contains(num3);
      char value = 'a';
      bool flag4 = "Bart".IndexOf(value) >= 0;
      static int ReadInt(string name)
      {
         Console.Write(name + ": ");
         return int.Parse(Console.ReadLine());
      }
   }
   ```