# Weeder

Weeder is an application to perform whole-program dead-code analysis. Dead code
is code that is written, but never reachable from any other code. Over the
lifetime of a project, this happens as code is added and removed, and leftover
code is never cleaned up. While GHC has warnings to detect dead code is a single
module, these warnings don't extend across module boundaries - this is where
Weeder comes in.

Weeder uses HIE files produced by GHC - these files can be thought of as source
code that has been enhanced by GHC, adding full symbol resolution and type
information. Weeder builds a dependency graph from these files to understand how
code interacts. Once all analysis is done, Weeder performs a traversal of this
graph from a set of roots (e.g., your `main` function), and determines which
code is reachable and which code is dead.

# Using Weeder

## Preparing Your Code for Weeder

To use Weeder, you will need to generate `.hie` files from your source code. If
you use Cabal, this is easily done by adding one line to your
`cabal.project.local` file:

``` cabal
package *
  ghc-options: -fwrite-ide-info
```

Once this has been added, perform a full rebuild of your project:

``` shell
cabal clean
cabal build all
```

## Calling Weeder

To call Weeder, you first need to provide a configuration file. Weeder uses
[Dhall](https://dhall-lang.org) as its configuration format, and configuration
files have the type:

``` dhall
{ roots : List Text, type-class-roots : Bool }
```

`roots` is a list of regular expressions of symbols that are considered as
alive. If you're building an executable, the pattern `^Main.main$` is a
good starting point - specifying that `main` is a root.

`type-class-roots` configures whether or not Weeder should consider anything in
a type class instance as a root. Weeder is currently unable to add dependency
edges into type class instances, and without this flag may produce false
positives. It's recommended to initially set this to `True`:

``` dhall
{ roots = [ "^Main.main$" ], type-class-roots = True }
```

Now invoke the `weeder` executable, and - if your project has weeds - you will
see something like the following:

``` shell
$ weeder

src/Dhall/TH.hs:187:1: error: toDeclaration is unused

     185 ┃     -> HaskellType (Expr s a)
     186 ┃     -> Q Dec
     187 ┃ toDeclaration haskellTypes MultipleConstructors{..} = do
     188 ┃     case code of
     189 ┃         Union kts -> do

    Delete this definition or add ‘Dhall.TH.toDeclaration’ as a root to fix this error.


src/Dhall/TH.hs:106:1: error: toNestedHaskellType is unused

     104 ┃     -- ^ Dhall expression to convert to a simple Haskell type
     105 ┃     -> Q Type
     106 ┃ toNestedHaskellType haskellTypes = loop
     107 ┃   where
     108 ┃     loop dhallType = case dhallType of

    Delete this definition or add ‘Dhall.TH.toNestedHaskellType’ as a root to fix this error.
```

(Please note these warnings are just for demonstration and not necessarily weeds
in the Dhall project).

# Limitations

Weeder currently has a few limitations:

## Type Class Instances

Weeder is not currently able to analyse whether a type class instance is used.
For this reason, Weeder adds all symbols referenced to from a type class
instance to the root set, keeping this code alive. In short, this means Weeder
might not detect dead code if it's used from a type class instance which is
never actually needed.

You can toggle whether Weeder consider type class instances as roots with the
`type-class-roots` configuration option.

## Template Haskell

Weeder is currently unable to parse the result of a Template Haskell splice. If
some Template Haskell code refers to other source code, this dependency won't be
tracked by Weeder, and thus Weeder might end up with false positives.
