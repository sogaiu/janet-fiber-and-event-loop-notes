# janet-fiber-and-event-loop-notes

## Callables

* [cancel](doc/cancel.md)
* [ev/cancel](doc/ev_cancel.md)
* [ev/deadline](doc/ev_deadline.md)
* [ev/read](doc/ev_read.md)
* [ev/with-deadline](doc/ev_with-deadline.md)
* [fiber/status](doc/fiber_status.md)
* [resume](doc/resume.md)

## Possible Website Doc Changes

* Regarding channels, the text: "and only work inside a thread"
  appears in the Task Communication section of the Event Loop page.
  There are threaded channels now so the text seems a bit off.

* Add something about catching errors that result from using `cancel`
  or `ev/cancel`.  For example, the code:

    ```janet
    (def fib
      (coro
        (try
          (yield 1)
          ([e]
            (eprint "nifty")))))

    (resume fib)

    (cancel fib :error)
    ```

  results in the output "nifty".

* Consider code sample and associated description for choosing
  arguments for `ev/deadline`.  Specifically, cover the case where
  `tocancel` can arrange to take care of `tocheck`.  See the item
  above about "catching errors that result from".  It's possible the
  text could live near it.

* Consider enhancing the pseudo-code for the event loop:

    ```janet
    # Functions that yield to the event loop will put (fiber/root) here.
    (def pending-tasks @[])

    # Pseudo-code of the event loop scheduler
    (while (not (empty? pending-tasks))
      (def [root-fiber data] (wait-for-next-event))
      (resume root-fiber data))
    ```

  so that it also illustrates handling of timeouts and may be some
  other bits?

## C Function Documentation

```c
JanetSignal janet_continue(JanetFiber *fiber, Janet in, Janet *out);
```

Resumes a new or suspended `fiber`. Returns a signal that corresponds
to the status of the fiber after execution, and places the
return/signal value in `out`. When resuming a fiber, the value to
resume with should be in the argument `in`, which corresponds to the
second argument to the Janet `resume` function.

```c
JanetSignal janet_continue_no_check(JanetFiber *fiber, Janet in, Janet *out);
```

Other `janet_continue*` functions are wrappers around this one.

```c
JanetSignal janet_continue_signal(JanetFiber *fiber, Janet in, Janet *out, JanetSignal sig);
```

Indirectly enables C functions that yield to the event loop to raise
errors or other signals (via CHANGELOG).

## Questions

* What do the following phrases mean in detail?

  * Suspend

    * "...suspend(ing|s) the current fiber"
      * `ev/do-thread`
      * `ev/give`
      * `ev/take`
      * `ev/write`
      * `ev/sleep`
      * `ev/thread`
      * `net/read`
      * `net/write`

    * "(Cancel|Resume) a (new or suspended) fiber..."
      * `ev/cancel`
      * `resume`

    * "...suspended..." (`fiber/status`)
      * :debug - the fiber is suspended in debug mode
      * :user(0-7) - the fiber is suspended by a user signal
      * :suspended - the fiber is waiting to be resumed by the scheduler

    * [`is_suspended` in `janet_loop1` in `ev.c`](https://github.com/janet-lang/janet/blob/03ae2ec1537efee3259a77639932cddfc318e995/src/core/ev.c#L1334):

        ```c
        int is_suspended = sig == JANET_SIGNAL_EVENT || sig == JANET_SIGNAL_YIELD || sig == JANET_SIGNAL_INTERRUPT;
        ```

  * Pending

    * "...a pending fiber"
      * `cancel`
      * `resume`

    * "pending" (`fiber/status`)
      * :pending - the fiber has been yielded

  * Schedule(d|r)

    * `ev/all-tasks`
    * `ev/call`
    * `ev/go`
    * `fiber/status`

* What does it mean for one fiber to be a child fiber of another
  (from the perspective of Janet's C source code)?

  Have seen the pattern of setting / ensuring a fiber's child and then
  calling some kind of `janet_continue*` function, possibly followed
  by setting the fiber's child field to `NULL`.

  Perhaps this is setting things up so that if some type of thing
  occurs while the child is being "continued", the parent can be used
  in some appropriate way.  May be one thing this has to do with is
  error-handling?


    ```c
        fiber->child = child;
        JanetSignal sig = janet_continue_no_check(child, stack[C], &retreg);
        if (sig != JANET_SIGNAL_OK && !(child->flags & (1 << sig))) {
            vm_return(sig, retreg);
        }
        fiber->child = NULL;
    ```

    ```c
        fiber->child = child;
        JanetSignal sig = janet_continue_signal(child, stack[C], &retreg, JANET_SIGNAL_ERROR);
        if (sig != JANET_SIGNAL_OK && !(child->flags & (1 << sig))) {
            vm_return(sig, retreg);
        }
        fiber->child = NULL;
    ```

    ```c
    /* Continue child fiber if it exists */
    if (fiber->child) {
        if (janet_vm.root_fiber == NULL) janet_vm.root_fiber = fiber;
        JanetFiber *child = fiber->child;
        uint32_t instr = (janet_stack_frame(fiber->data + fiber->frame)->pc)[0];
        janet_vm.stackn++;
        JanetSignal sig = janet_continue(child, in, &in);
        janet_vm.stackn--;
        if (janet_vm.root_fiber == fiber) janet_vm.root_fiber = NULL;
        if (sig != JANET_SIGNAL_OK && !(child->flags & (1 << sig))) {
            *out = in;
            janet_fiber_set_status(fiber, sig);
            fiber->last_value = child->last_value;
            return sig;
        }
        // ... elided ... //
        }
        fiber->child = NULL;
    ```

    ```c
            janet_vm.fiber->child = child;
            JanetSignal sig = janet_continue(child, janet_wrap_nil(), &retreg);
            if (sig != JANET_SIGNAL_OK && !(child->flags & (1 << sig))) {
                if (is_interpreter) {
                    janet_signalv(sig, retreg);
                } else {
                    janet_vm.fiber->child = NULL;
                    janet_panicv(retreg);
                }
            }
            janet_vm.fiber->child = NULL;
    ```

* What are all of the functions that can block apart from the
  following?

  * all(?) functions in `file/`
    * file/close
    * file/flush
    * file/lines
    * file/open
    * file/read
    * file/seek
    * file/tell
    * file/temp
    * file/write
  * `os/sleep`
  * `getline`
  * `ev/read` can block if working with something returned from `os/open` (see [this](https://github.com/janet-lang/janet/issues/1397))

## Resolved Questions

* There appear to be at least two types of fibers in Janet.  Those
  that end up on the event loop and those that don't.  Is it fair to
  make this distinction using the term "task" to describe the fibers
  that end up on the event loop and may be "non-task" for those that
  don't?

  > The distinction between "task" fibers and normal fibers is now
  > kept by a flag that is set when a fiber is resumed - if it is the
  > outermost fiber on the stack, it is considered a root fiber. All
  > fibers scheduled with ev/go or by the event loop are root fibers,
  > and thus cannot be cancelled or resumed with `cancel` or `resume`
  > \- instead, use `ev/cancel` or `ev/go`.

  (source: text from [commit addressing #920](https://github.com/janet-lang/janet/commit/a9f38dfce4e892dad370efe768bb3f59eb2b79ab))

  ```c
  /* If a "task" fiber is trying to be used as a normal fiber, detect that. See bug #920.
   * Fibers must be marked as root fibers manually, or by the ev scheduler. */
  ```

  (source: [comment in vm.c](https://github.com/janet-lang/janet/blob/431ecd3d1a4caabc66b62f63c2f83ece2f74e9f9/src/core/vm.c#L1401-L1402))

* Are the terms "root-fiber" and "task" equivalent?  It seems they are
  at least close.  (Peripheral case: if janet is compiled without ev
  support, are there tasks?  There may be root-fibers...).

## Official Doc Snippets

* Each root-fiber, or task, is a fiber that will be automatically
  resumed when an event (or sequence of events) that it is waiting for
  occurs.  Generally, one should not manually resume tasks - the event
  loop will call resume when the completion event occurs. (source:
  event loop page)

* To be precise, a task is just any fiber that was scheduled to run by
  the event loop. (source: event loop page)

* A default Janet program has a single task that will run until
  complete.  To create new tasks, Janet provides two built-in
  functions - `ev/go` and `ev/call`. (source: event loop page)

* You can get the currently executing task in Janet with
  `(fiber/root)`. (source: event loop page)

* The root fiber is the oldest ancestor that does not have a parent.
  (source: the `fiber/root` docstring)

## Credits

* amano.kenji - code, discussion, feedback
* bakpakin - code, discussion, tips
* llmII - code, discussion, feedback
