# ev/with-deadline

## Information

`(ev/with-deadline sec & body)`

Create a fiber to execute `body`, schedule the event loop to cancel
the task (root fiber) associated with `body`'s fiber, and start
`body`'s fiber by resuming it.

The event loop will try to cancel the root fiber if `body`'s fiber has
not completed after at least `sec` seconds.

`sec` is a number that can have a fractional part.

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
