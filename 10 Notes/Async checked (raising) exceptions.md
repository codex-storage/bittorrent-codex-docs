For a seasoned Nim developer a lot of things I am writing here may be obvious, but for those in a continuous learning path, it may bring some consolation.

The [The Status Nim style guide](https://status-im.github.io/nim-style-guide) recommends [explicit error handling mechanisms](https://status-im.github.io/nim-style-guide/errors.result.html) and handling errors locally at each abstraction level, avoiding spurious abstraction leakage. This is in contrast to leaking exception types between layers and translating exceptions that causes high visual overhead, specially when hierarchy is used, often becoming not practical, loosing all advantages of using exceptions in the first place (read more [here](https://status-im.github.io/nim-style-guide/errors.exceptions.html)).

Handling error and working with exceptions is easier to grasp when not using asynchronous code. But when you start, there are some subtle traps you may be falling into.

This short note focuses on asynchronous code. It is not complete, but pragmatic, it has gaps, but provides directions if one wants to research further.

In our code we often use the following patterns:
1. using [nim-results](https://github.com/arnetheduck/nim-results) and [std/options](https://nim-lang.org/docs/options.html) to communicate the results and failures to the caller,
2. *async* operations are annotated with `{.async: (raises: [CancelledError]).}`.

Some interesting things are happening when you annotate a Nim `proc` with "async raises".

Let's looks at some examples.

Imagine you have the following type definitions:

```nim
type
  MyError = object of CatchableError
  Handle1 = Future[void].Raising([CancelledError])
  Handle2 = Future[void].Raising([CancelledError, MyError])

  SomeType = object
    name: string
    handle1: Handle1
    handle2: Handle2
```

`Handle1` and `Handle2` are *raising exceptions*. By using `Rasing` macro, passing the list of allowed exceptions coming out from the future as an argument, `Handle1` and `Handle2` are no longer well-known `Future[void]`, but rather a descendant of it:

```nim
type
  InternalRaisesFuture*[T, E] = ref object of Future[T]
    ## Future with a tuple of possible exception types
    ## eg InternalRaisesFuture[void, (ValueError, OSError)]
    ##
    ## This type gets injected by `async: (raises: ...)` and similar utilities
    ## and should not be used manually as the internal exception representation
    ## is subject to change in future chronos versions.
    # TODO https://github.com/nim-lang/Nim/issues/23418
    # TODO https://github.com/nim-lang/Nim/issues/23419
    when E is void:
      dummy: E
    else:
      dummy: array[0, E]
```

The comment is saying something important: if you annotate a `proc` with `async: (raises: ...)`, you are changing the type being returned by the `proc`. To see what does it mean, lets start with something easy. Let's write a constructor for `SomeType`:

```nim
proc newSomeType(name: string): SomeType =
  let t = SomeType(
    name: name,
    # both fail
    handle1: newFuture[void](),
    handle2: newFuture[void](),
  )
  t
```

Well, this will not compile. `handle1` expects `InternalRaisesFuture[system.void, (CancelledError,)]`, but instead it gets `Future[system.void]`. Yes, we are trying to cast a more generic to a less generic type. This is because `newSomeType` is not annotated with `async: (raises: ...)` and therefore every time you use `newFuture` inside it, `newFuture` returns regular `Future[void]`.

So, the first time I encountered this problem I went into a rabbit hole of understanding how to create *raising* futures by solely relaying on `async: (raises: ...)` pragma. But there is actually a public (I guess) interface allowing us to create a raising future without relaying on `async: (raises: ...)` annotation:

```
```nim
proc newSomeType(name: string): SomeType =
  let t = SomeType(
    name: name,
    # both fail
    handle1: Future[void].Raising([CancelledError]).init(),
    handle2: Future[void].Raising([CancelledError, MyError]).init(),
  )
  t
```

A bit verbose, but perfectly fine otherwise, and it works as expected:

```nim
let someTypeInstance = newSomeType("test")

echo typeof(someTypeInstance.handle1) # outputs "Handle1"
echo typeof(someTypeInstance.handle2) # outputs "Handle2"
```

`init` has the following definition:

```nim
template init*[T, E](
    F: type InternalRaisesFuture[T, E], fromProc: static[string] = ""): F =
  ## Creates a new pending future.
  ##
  ## Specifying ``fromProc``, which is a string specifying the name of the proc
  ## that this future belongs to, is a good habit as it helps with debugging.
  when not hasException(type(E), "CancelledError"):
    static:
      raiseAssert "Manually created futures must either own cancellation schedule or raise CancelledError"


  let res = F()
  internalInitFutureBase(res, getSrcLocation(fromProc), FutureState.Pending, {})
  res
```

and is very similar to:

```nim
proc newInternalRaisesFutureImpl[T, E](
    loc: ptr SrcLoc, flags: FutureFlags): InternalRaisesFuture[T, E] =
  let fut = InternalRaisesFuture[T, E]()
  internalInitFutureBase(fut, loc, FutureState.Pending, flags)
  fut
```

thus, if we had exposed the internals, our example would be:

```nim
proc newSomeType(name: string): SomeType =
  let t = SomeType(
    name: name,
    handle1: newInternalRaisesFuture[void, (CancelledError,)](),
    handle2: newInternalRaisesFuture[void, (CancelledError, MyError)](),
  )
  t
```


It is still very educational to study the chronos source code to undertstand how does the `newFuture` know which type to return: `Future[T]` or `InternalRaisesFuture[T, E]` when a proc is annotated with `async` or `async: (raises: ...)`.

If you study `chronos/internal/asyncfutures.nim` you will see that `newFuture` is implemented with the following template:

```nim
template newFuture*[T](fromProc: static[string] = "",
                       flags: static[FutureFlags] = {}): auto =
  ## Creates a new future.
  ##
  ## Specifying ``fromProc``, which is a string specifying the name of the proc
  ## that this future belongs to, is a good habit as it helps with debugging.
  when declared(InternalRaisesFutureRaises): # injected by `asyncraises`
    newInternalRaisesFutureImpl[T, InternalRaisesFutureRaises](
      getSrcLocation(fromProc), flags)
  else:
    newFutureImpl[T](getSrcLocation(fromProc), flags)
```

We see the the actual implementation depends on the existence of `InternalRaisesFutureRaises`. Let's see how it is being setup...

The `async` pragma is a macro defined in `chronos/internal/asyncmacro.nim`:

```nim
macro async*(params, prc: untyped): untyped =
  ## Macro which processes async procedures into the appropriate
  ## iterators and yield statements.
  if prc.kind == nnkStmtList:
    result = newStmtList()
    for oneProc in prc:
      result.add asyncSingleProc(oneProc, params)
  else:
    result = asyncSingleProc(prc, params)
```

The `asyncSingleProc` is where a lot of things happen. This is where the errors *The raises pragma doesn't work on async procedures* or *Expected return type of 'Future' got ...* come from. The place where the return type is determined is interesting:

```nim
let baseType =
  if returnType.kind == nnkEmpty:
    ident "void"
  elif not (
      returnType.kind == nnkBracketExpr and
      (eqIdent(returnType[0], "Future") or eqIdent(returnType[0], "InternalRaisesFuture"))):
    error(
      "Expected return type of 'Future' got '" & repr(returnType) & "'", prc)
    return
  else:
    returnType[1]
```

An async proc can have two (explicit) return types: `Future[baseType]` or `InternalRaisesFuture[baseType]`. If no return type is specified for an async proc, the return base type is concluded to be `void`. Now the crucial part: the internal return type (we are still inside of `asyncSingleProc`):

```nim
baseTypeIsVoid = baseType.eqIdent("void")
(raw, raises, handleException) = decodeParams(params)
internalFutureType =
  if baseTypeIsVoid:
    newNimNode(nnkBracketExpr, prc).
      add(newIdentNode("Future")).
      add(baseType)
  else:
    returnType
internalReturnType = if raises == nil:
  internalFutureType
else:
  nnkBracketExpr.newTree(
    newIdentNode("InternalRaisesFuture"),
    baseType,
    raises
  )
```

To focus on the most important part, at the end we see that if `raises` attribute is present and set (`async: (raises: [])` means it does not raise, but the attribute is still present and detected), the `internalReturnType` will be set to:

```nim
nnkBracketExpr.newTree(
  newIdentNode("InternalRaisesFuture"),
  baseType,
  raises
)
```

Thus, for `async: (raises: [CancelledError, ValueError])`, the return type will be `InternalRaisesFuture[baseType, (CancelledError, ValueError,)`.

If the `async` has `raw: true` param set, e.g. `async: (raw: true, raises: [CancelledError, ValueError])`, then `prc.body` gets prepended with the type definition we already recognize from `newFuture` above: `InternalRaisesFutureRaises`

```nim
if raw: # raw async = body is left as-is
  if raises != nil and prc.kind notin {nnkProcTy, nnkLambda} and not isEmpty(prc.body):
    # Inject `raises` type marker that causes `newFuture` to return a raise-
    # tracking future instead of an ordinary future:
    #
    # type InternalRaisesFutureRaises = `raisesTuple`
    # `body`
    prc.body = nnkStmtList.newTree(
      nnkTypeSection.newTree(
        nnkTypeDef.newTree(
          nnkPragmaExpr.newTree(
            ident"InternalRaisesFutureRaises",
            nnkPragma.newTree(ident "used")),
          newEmptyNode(),
          raises,
        )
      ),
      prc.body
    )
```

For our example of  `async: (raw: true, raises: [CancelledError, ValueError])`, this will be:

```nim
type InternalRaisesFutureRaises {.used.} = (CancelledError, ValueError,)
```

This allows the `newFuture` template to recognize it has to use `InternalRaisesFuture` as the return type.

### Experimenting with *Raising Futures*

With the `Future[...].Raising(...).init()` construct we can quite elegantly create new raising futures in regular proc not annotated with `async: (raises: ...)`.  But to get more intuition, let's play a bit with creating our own version of `Future[...].Raising(...).init()` that will be built on top of `async: (raises: ...)` pragma.

> [!info]
> This is just an exercise. It reveals some interesting details about how `async` is implemented. I also used it to learn some basics about using macros and how they can help where generics have limitations.

Let's start with creating a proc that returns type `Handle1`?

Recall that `Handle1` is defined as follows:

```nim
type
  Handle1 = Future[void].Raising([CancelledError])
```

```nim
proc newHandle1(): Handle1 {.async: (raw: true, [CancelledError]).} =
  newFuture[void]()

proc newSomeType(name: string): SomeType =
  let t = SomeType(
    name: name,
    handle1: newHandle1(),
    handle2: Future[void].Raising([CancelledError, MyError]).init(),
  )
  t
```

That would be nice an concise, yet, you remember now the "Expected return type of 'Future' got ..." error from `asyncSingleProc`, right? This is what we will get:

```bash
Error: Expected return type of 'Future' got 'Handle1'
```

Knowing the implementation of the `asyncSingleProc` macro, we know that `InternalRaisesFuture[void, (CancelledError,)]` would work just fine as the return type:

```nim
proc newHandle1(): InternalRaisesFuture[void, (CancelledError,)] {.async: (raw: true, raises: [CancelledError]).} =
  newFuture[void]()
```

but not:

```nim
proc newHandle1(): Future[void].Raises(CancelledError) {.async: (raw: true, raises: [CancelledError]).} =
  newFuture[void]()
```

Thus we have to stick to `Future` as the return type if we want to stick to the public interface:

```nim
proc newHandle1(): Future[void]
    {.async: (raw: true, raises: [CancelledError]).} =
  newFuture[void]()
```

It actually does not matter that we specify `Future[void]` as the return type (yet, it has to be `Future`): the actual return type of `newFuture` and of the `newHandle1` proc  will be `InternalRaisesFuture[void, (CancelledError,)]` thanks to the `assert: (raw: true, raises: [CancelledError])`.

It would be nice if we can create a more generic version of `newHandle`, so that we do not have to create a new one for each single raising future type. Ideally, we would like this generic to also allow us handling the raised exceptions accordingly.

Using just plain generic does not seem to allow us passing the list of exception types so that it lands nicely in the `raises: [...]` attribute:


```nim
proc newRaisingFuture[T, E](): Future[T] {.async: (raw: true, raises: [E]).} =
  newFuture[T]()
```

With this we can pass a single exception type as E. To pass a list of exceptions we can use a template:

```nim
template newRaisingFuture[T](raising: typed): untyped =
  block:
    proc wrapper(): Future[T] {.async: (raw: true, raises: raising).} =
      newFuture[T]()
    wrapper()
```

With the `newRaisingFuture` template we can simplify our example to get:

```nim
proc newSomeType(name: string): SomeType =
  let t = SomeType(
    name: name,
    handle1: newRaisingFuture[void]([CancelledError]),
    handle2: newRaisingFuture[void]([CancelledError, MyError]),
  )
  t
```

Perhaps, a more elegant solution would be to use an IIFE (Immediately Invoked Function Expression), e.g.:

```nim
(proc (): Future[void]
    {.async: (raw: true, raises: [CancelledError, MyError]).} =
  newFuture[void]())()
```

so that we can create a raising future instance like this:

```nim
let ff = (
  proc (): Future[void]
      {.async: (raw: true, raises: [CancelledError, MyError]).} =
    newFuture[void]())()
)()
```

Unfortunately, this will fail with error similar to this one:

```bash
Error: type mismatch: got 'Future[system.void]' for '
newFutureImpl(srcLocImpl("", (filename: "raisingfutures.nim", line: 264,
    column: 19).filename, (filename: "raisingfutures.nim", line: 264,
    column: 19).line), {})' but expected 
    'InternalRaisesFuture[system.void, (CancelledError, MyError)]'
```

To see what happened, we can use the `-d:nimDumpAsync` option when compiling, e.g.:

```bash
nim c -r -o:build/ --NimblePath:.nimble/pkgs2 -d:nimDumpAsync raisingfutures.nim 
```

This option will print us the expanded `async` macro, where we can find that our call expanded to:

```nim
proc (): InternalRaisesFuture[void, (CancelledError, MyError)]
    {.raises: [], gcsafe.} =
  newFuture[void]()
```

This obviously misses the definition of the `InternalRaisesFutureRaises` type before calling `newFuture`, which would change the behavior of the `newFuture` call so that instead of returning a regular `Future[void]` it would return  `InternalRaisesFuture[system.void, (CancelledError, MyError)]`. The same function, evaluated as regular proc (and not as lambda call) would take the following form: 

```nim
proc (): InternalRaisesFuture[seq[int], (CancelledError, MyError)]
    {.raises: [], gcsafe.} =
  type InternalRaisesFutureRaises {.used.} = (CancelledError, ValueError,)
  newFuture[seq[int]]()
```

Looking again into the `chronos/internal/asyncmacro.nim`:

```nim
if raw: # raw async = body is left as-is
  if raises != nil and prc.kind notin {nnkProcTy, nnkLambda} and not isEmpty(prc.body):
    # Inject `raises` type marker that causes `newFuture` to return a raise-
    # tracking future instead of an ordinary future:
    #
    # type InternalRaisesFutureRaises = `raisesTuple`
    # `body`
    prc.body = nnkStmtList.newTree(
      nnkTypeSection.newTree(
        nnkTypeDef.newTree(
          nnkPragmaExpr.newTree(
            ident"InternalRaisesFutureRaises",
            nnkPragma.newTree(ident "used")),
          newEmptyNode(),
          raises,
        )
      ),
      prc.body
    )

```

we see the condition:

```nim
if raises != nil and prc.kind notin {nnkProcTy, nnkLambda} and not isEmpty(prc.body):
```

Unfortunately, in our case `prc.kind` is `nnkLambda`, and so the above mentioned type infusion will not happen...

> I do not know why it is chosen to be like this...

Thus, if we would like to use IIFE, we do have to use an internal function from `chronos/internal/asyncfutures.nim`:

```nim
(proc (): Future[void]
    {.async: (raw: true, raises: [CancelledError, MyError]).} =
  newInternalRaisesFuture[void, (CancelledError, MyError)]())()
```

This call will work, and we can then "hide" the internal primitive in a macro. Below I show the macro, we can use to conveniently create *raising futures* using the IIFE: 

```nim
macro newRaisingFuture(T: typedesc, E: typed): untyped =
  let 
    baseType = T.strVal
    e =
      case E.getTypeInst().typeKind()
      of ntyTypeDesc: @[E]
      of ntyArray:
        for x in E:
          if x.getTypeInst().typeKind != ntyTypeDesc:
            error("Expected typedesc, got " & repr(x), x)
        E.mapIt(it)
      else:
        error("Expected typedesc, got " & repr(E), E)

  let raises = if e.len == 0:
    nnkBracket.newTree()
  else:
    nnkBracket.newTree(e)
  let raisesTuple = if e.len == 0:
    makeNoRaises()
  else:
    nnkTupleConstr.newTree(e)
  
  result = nnkStmtList.newTree(
    nnkCall.newTree(
      nnkPar.newTree(
        nnkLambda.newTree(
          newEmptyNode(),
          newEmptyNode(),
          newEmptyNode(),
          nnkFormalParams.newTree(
            nnkBracketExpr.newTree(
              newIdentNode("Future"),
              newIdentNode(baseType)
            )
          ),
          nnkPragma.newTree(
            nnkExprColonExpr.newTree(
              newIdentNode("async"),
              nnkTupleConstr.newTree(
                nnkExprColonExpr.newTree(
                  newIdentNode("raw"),
                  newIdentNode("true")
                ),
                nnkExprColonExpr.newTree(
                  newIdentNode("raises"),
                  raises
                )
              )
            )
          ),
          newEmptyNode(),
          nnkStmtList.newTree(
            nnkCall.newTree(
              nnkBracketExpr.newTree(
                newIdentNode("newInternalRaisesFuture"),
                newIdentNode(baseType),
                raisesTuple
              )
            )
          )
        )
      )
    )
  )
```

Now, creating a raising future is quite elegant:

```nim
proc newSomeType(name: string): SomeType =
  let t = SomeType(
    name: name,
    handle1: newRaisingFuture(void, CancelledError),
    handle2: newRaisingFuture(void, [CancelledError, MyError]),
  )
  t
```

### Using raising futures types

While `Future[...].Raising(...).init()` provides us with quite elegant (although verbose) interface to create raising futures, it seems to display some subtle limitations.

To demonstrate them, let start with the following, quite innocent looking proc:

```nim
proc waitHandle[T](h: Future[T]): Future[T]
    {.async: (raises: [CancelledError]).} =
  await h
```

Now, let's try to call it passing a raising future as an argument:

```nim
let handle = newRaisingFuture(int, [CancelledError])
handle.complete(42)
echo waitFor waitHandle(handle)
```

> [!info]
> In the examples I am using our macro - just for fun, and it is also shorter to type than `Future[...].Raising(...).init()`

The compilation will fail with the following error:

```bash
Error: cast[type(h)](chronosInternalRetFuture.internalChild).internalError can raise an unlisted exception: ref CatchableError
```

First, realize that we are passing `InternalRaisesFuture[void, (CancelledError,)]` as `Future[void]`. Because we have that:

```nim
type InternalRaisesFuture*[T, E] = ref object of Future[T]
```

it will not cause any troubles. For `Future[T]`, the following version of `await` will be called:

```nim
template await*[T](f: Future[T]): T =
  ## Ensure that the given `Future` is finished, then return its value.
  ##
  ## If the `Future` failed or was cancelled, the corresponding exception will
  ## be raised instead.
  ##
  ## If the `Future` is pending, execution of the current `async` procedure
  ## will be suspended until the `Future` is finished.
  when declared(chronosInternalRetFuture):
    chronosInternalRetFuture.internalChild = f
    # `futureContinue` calls the iterator generated by the `async`
    # transformation - `yield` gives control back to `futureContinue` which is
    # responsible for resuming execution once the yielded future is finished
    yield chronosInternalRetFuture.internalChild
    # `child` released by `futureContinue`
    cast[type(f)](chronosInternalRetFuture.internalChild).internalRaiseIfError(f)
    when T isnot void:
      cast[type(f)](chronosInternalRetFuture.internalChild).value()
  else:
    unsupported "await is only available within {.async.}"
```

The reason for exception is:

```nim
cast[type(f)](chronosInternalRetFuture.internalChild).internalRaiseIfError(f)
```

which is:

```nim
macro internalRaiseIfError*(fut: FutureBase, info: typed) =
  # Check the error field of the given future and raise if it's set to non-nil.
  # This is a macro so we can capture the line info from the original call and
  # report the correct line number on exception effect violation
  let
    info = info.lineInfoObj()
    res = quote do:
      if not(isNil(`fut`.internalError)):
        when chronosStackTrace:
          injectStacktrace(`fut`.internalError)
        raise `fut`.internalError
  res.deepLineInfo(info)
  res
```

Thus, this will cause the error we see above.

>[!info]
> Notice that it is not *casting* that causes an error.

We could cast to `InternalRaisesFuture` before calling `await`, but although we can often be pretty sure that what we are doing is right, it would be better to avoid downcasting if possible.

Thus, in our `waitHandle` it would be better that we capture the correct type. Fortunately, this is possible, although not obvious.

Unfortunately, the following will not work as the type of the argument `h`:

```nim
Future[T].Raising([CancelledError])
Raising[T](Future[T],[CancelledError])
Raising(Future[T],[CancelledError])
```

Sadly, original `Raising` macro is not flexible enough to handle complications of the generics.

We may like to define a custom type, e.g.:

```nim
type
  Handle[T] = Future[T].Raising([CancelledError])
```

Unfortunately, `Raising` macro is again not flexible enough to handle this. Doing:

```nim
type
  Handle[T] = Raising(Future[T], [CancelledError])
```

looks more promising, but trying to use `Handle[T]` as type for `h` in our `waitHandle` fails.

What works is `auto`:

```nim
proc waitHandle[T](_: typedesc[Future[T]], h: auto): Future[T] {.async: (raises: [CancelledError]).} =
  await h

let handle = newRaisingFuture(int, [CancelledError])
handle.complete(42)
echo waitFor Future[int].waitHandle(handle)
```

Finally, I have experimented a bit with modifying the original `Rasing` macro from chronos, to see if I can make it a bit more permissive. In particular, where `Future[...].Raising(...)` seem to have limitations are generic types, e.g.:

```nim
SomeType2[T] = object
  name: string
  # will not compile
  handle: Future[T].Raising([CancelledError])
```

Here is a version that works:

```nim
from pkg/chronos/internal/raisesfutures import makeNoRaises

macro RaisingFuture(T: typedesc, E: typed): untyped =
  let 
    baseType = T.strVal
    e =
      case E.getTypeInst().typeKind()
      of ntyTypeDesc: @[E]
      of ntyArray:
        for x in E:
          if x.getTypeInst().typeKind != ntyTypeDesc:
            error("Expected typedesc, got " & repr(x), x)
        E.mapIt(it)
      else:
        error("Expected typedesc, got " & repr(E), E)
        # @[]

  let raises = if e.len == 0:
    makeNoRaises()
  else:
    nnkTupleConstr.newTree(e)

  result = nnkBracketExpr.newTree(
    ident "InternalRaisesFuture",
    newIdentNode(baseType),
    raises
  )
```

We can then do:

```nim
SomeType2[T] = object
  name: string
  # will not compile
  handle: RaisingFuture(T, [CancelledError])
```

We finish this note with some examples of using the `RasingFuture` and `newRaisingFuture` macros:

```nim
type
  MyError = object of CatchableError

  SomeType = object
    name: string  
    handle1: RaisingFuture(void, CancelledError)
    handle2: RaisingFuture(void, [CancelledError, MyError])
    handle3: RaisingFuture(int, [CancelledError])
  
  SomeType2[T] = object
    name: string
    handle: RaisingFuture(T, [CancelledError])
    

proc newSomeType(name: string): SomeType =
  let t = SomeType(
    name: name,
    handle1: newRaisingFuture(void, CancelledError),
    handle2: newRaisingFuture(void, [CancelledError, MyError]),
    handle3: newRaisingFuture(int, [CancelledError]),
  )
  t

proc newSomeType2[T](name: string): SomeType2[T] =
  let t = SomeType2[T](
    name: name,
    handle: newRaisingFuture(T, CancelledError),
  )
  t

let someTypeInstance = newSomeType("test")

echo typeof(someTypeInstance.handle1)
echo typeof(someTypeInstance.handle2)
echo typeof(someTypeInstance.handle3)

let someType2Instance = newSomeType2[int]("test2")
echo typeof(someType2Instance.handle)

proc waitHandle[T](_: typedesc[Future[T]], h: auto): Future[T] {.async: (raises: [CancelledError]).} =
  await h

proc waitHandle2[T](_: typedesc[Future[T]], h: RaisingFuture(T, [CancelledError])): Future[T] {.async: (raises: [CancelledError]).} =
  return await h


someTypeInstance.handle1.complete()

waitFor Future[void].waitHandle(someTypeInstance.handle1)
echo "done 1"  

someType2Instance.handle.complete(42)

echo waitFor Future[int].waitHandle(someType2Instance.handle)
echo "done 2"

let handle = newRaisingFuture(int, CancelledError)
handle.complete(43)
echo waitFor Future[int].waitHandle2(handle)
echo "done 3"
```

You can also find the source files on GitHub:

- [raisingfutures.nim](https://gist.github.com/marcinczenko/cb48898d24314fbdebe57fd815c3c1be#file-raisingfutures-nim)
- [raisingfutures2.nim](https://gist.github.com/marcinczenko/c667fad0b70718d6f157275a2a7e7a93#file-raisingfutures2-nim)
