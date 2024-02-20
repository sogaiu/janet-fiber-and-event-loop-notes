# ev/with-deadline

## Information

`(ev/with-deadline sec & body)`

Run a body of code with a deadline of `sec` seconds, such that if the
code does not complete before the deadline is up, it will be canceled.

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
