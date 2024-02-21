# ev/with-deadline

## Information

`(ev/with-deadline sec & body)`

### Option 1

Create a fiber to execute `body`, schedule the event loop to cancel
the task (root fiber) associated with `body`'s fiber, and start
`body`'s fiber by resuming it.

The event loop will try to cancel the aforementioned root fiber if
`body`'s fiber has not completed after at least `sec` seconds.

`sec` is a number that can have a fractional part.

### Option 2

Run a body of code with a deadline of `sec` seconds, such that if the
code does not complete before the deadline is up, the value of
`(fiber/root)` will be canceled. `sec` is a number that can have a
fractional part.

### Option 3

Run a body of code with a deadline of `sec` seconds, such that if the
code does not complete before the deadline is up, it will be canceled.

`sec` is a number that can have a fractional part.

> consider incorporating ideas from following ([original](https://github.com/janet-lang/janet/pull/1410#discussion_r1497037526))

> If the body of code inside `ev/with-deadline` discards the error
> caused by `ev/with-deadline`, it is not cancelled.

> possibly related is llmII's point about the current behavior not
> matching the docstring in some cases (see [this link](https://github.com/janet-lang/janet/pull/1410#issuecomment-1955482107))

## Sample Code

```janet
(ev/with-deadline 0.01
                  (ev/sleep 1)
                  (pp :hi))
```

Sample output:

```
error: deadline expired
  in ev/sleep [src/core/ev.c] on line 2945
  in <anonymous> [repl] on line 2, column 19
  in _thunk [repl] (tailcall) on line 1, column 1
```

## Janet Implementation

```janet
(defmacro ev/with-deadline
  [deadline & body]
  (with-syms [f]
    ~(let [,f (coro ,;body)]
       (,ev/deadline ,deadline nil ,f)
       (,resume ,f))))
```
