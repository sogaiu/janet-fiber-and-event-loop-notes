# ev/read

## Information

`(ev/read stream n &opt buffer timeout)`

Read up to n bytes into a buffer asynchronously from a stream.

`n` can also be the keyword `:all` to read into the buffer until end of
stream.

Optionally provide a buffer to write into as well as a timeout in
seconds after which to cancel the operation and raise an error.

Returns the buffer if the read was successful or nil if end-of-stream
reached.

Will raise an error if there are problems with the IO operation.

## Sample Code

Success with reading one byte, returns buffer:

```janet
(def [rs ws] (os/pipe))

(ev/write ws @"fun")

(ev/read rs 1)
# =>
@"f"
```

Success using `:all`, returns buffer:

```janet
(def [rs ws] (os/pipe))

(ev/write ws @"fun")

(ev/read rs :all)
# =>
@"fun"
```

Returns `nil` if end of stream is reached:

```janet
(def [rs ws] (os/pipe))

(ev/write ws @"fun")
(ev/close ws)

(ev/read rs :all)
# =>
@"fun"

(ev/read rs :all)
# =>
nil
```

Timeout expired, operation cancelled, and error raised:

```janet
(def [rs ws] (os/pipe))

(def timeout 1)

(ev/spawn
  (ev/sleep (* timeout 2))
  (ev/write ws @"fun"))

(ev/spawn
  (try
    (ev/read rs 1 @"" timeout)
    ([e]
      (eprintf "ev/read errored: %s" e))))
```

Sample output:

```
ev/read errored: timeout
```

## C Implementation

[`ev/read` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L3119-L3130):

```c
    JanetStream *stream = janet_getabstract(argv, 0, &janet_stream_type);
    janet_stream_flags(stream, JANET_STREAM_READABLE);
    JanetBuffer *buffer = janet_optbuffer(argv, argc, 2, 10);
    double to = janet_optnumber(argv, argc, 3, INFINITY);
    if (janet_keyeq(argv[1], "all")) {
        if (to != INFINITY) janet_addtimeout(to);
        janet_ev_readchunk(stream, buffer, INT32_MAX);
    } else {
        int32_t n = janet_getnat(argv, 1);
        if (to != INFINITY) janet_addtimeout(to);
        janet_ev_read(stream, buffer, n);
    }
```

[`JanetStream` in janet.h](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/include/janet.h#L615-L625):

```c
typedef struct JanetStream JanetStream;

/* Wrapper around file descriptors and HANDLEs that can be polled. */
struct JanetStream {
    JanetHandle handle;
    uint32_t flags;
    uint32_t index;
    JanetFiber *read_fiber;
    JanetFiber *write_fiber;
    const void *methods; /* Methods for this stream */
};
```

[`janet_stream_flags` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L2200-L2213):

```c
void janet_stream_flags(JanetStream *stream, uint32_t flags) {
    if (stream->flags & JANET_STREAM_CLOSED) {
        janet_panic("stream is closed");
    }
    if ((stream->flags & flags) != flags) {
        const char *rmsg = "", *wmsg = "", *amsg = "", *dmsg = "", *smsg = "stream";
        if (flags & JANET_STREAM_READABLE) rmsg = "readable ";
        if (flags & JANET_STREAM_WRITABLE) wmsg = "writable ";
        if (flags & JANET_STREAM_ACCEPTABLE) amsg = "server ";
        if (flags & JANET_STREAM_UDPSERVER) dmsg = "datagram ";
        if (flags & JANET_STREAM_SOCKET) smsg = "socket";
        janet_panicf("bad stream, expected %s%s%s%s%s", rmsg, wmsg, amsg, dmsg, smsg);
    }
}
```

[`janet_addtimeout` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L616-L626):

```c
/* Set timeout for the current root fiber */
void janet_addtimeout(double sec) {
    JanetFiber *fiber = janet_vm.root_fiber;
    JanetTimeout to;
    to.when = ts_delta(ts_now(), sec);
    to.fiber = fiber;
    to.curr_fiber = NULL;
    to.sched_id = fiber->sched_id;
    to.is_error = 1;
    add_timeout(to);
}
```

[`janet_ev_readchunk` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L2474-L2476):

```c
JANET_NO_RETURN void janet_ev_readchunk(JanetStream *stream, JanetBuffer *buf, int32_t nbytes) {
    janet_ev_read_generic(stream, buf, nbytes, 1, JANET_ASYNC_READMODE_READ, 0);
}
```

[`janet_ev_read_generic` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L2456-L2469):

```c
static JANET_NO_RETURN void janet_ev_read_generic(JanetStream *stream, JanetBuffer *buf, int32_t nbytes, int is_chunked, JanetReadMode mode, int flags) {
    StateRead *state = janet_malloc(sizeof(StateRead));
    state->is_chunk = is_chunked;
    state->buf = buf;
    state->bytes_left = nbytes;
    state->bytes_read = 0;
    state->mode = mode;
#ifdef JANET_WINDOWS
    state->flags = (DWORD) flags;
#else
    state->flags = flags;
#endif
    janet_async_start(stream, JANET_ASYNC_LISTEN_READ, ev_callback_read, state);
}
```

[`StateRead` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L2247-L2273):

```c
/* State machine for read/recv/recvfrom */

typedef enum {
    JANET_ASYNC_READMODE_READ,
    JANET_ASYNC_READMODE_RECV,
    JANET_ASYNC_READMODE_RECVFROM
} JanetReadMode;

typedef struct {
    // Windows bits elided
    int32_t bytes_left;
    int32_t bytes_read;
    JanetBuffer *buf;
    int is_chunk;
    JanetReadMode mode;
} StateRead;
```

[`janet_async_start` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L301-L304):

```c
void janet_async_start(JanetStream *stream, JanetAsyncMode mode, JanetEVCallback callback, void *state) {
    janet_async_start_fiber(janet_vm.root_fiber, stream, mode, callback, state);
    janet_await();
}
```

[`janet_async_start_fiber` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L285-L299):

```c
void janet_async_start_fiber(JanetFiber *fiber, JanetStream *stream, JanetAsyncMode mode, JanetEVCallback callback, void *state) {
    janet_assert(!fiber->ev_callback, "double async on fiber");
    if (mode & JANET_ASYNC_LISTEN_READ) {
        stream->read_fiber = fiber;
    }
    if (mode & JANET_ASYNC_LISTEN_WRITE) {
        stream->write_fiber = fiber;
    }
    fiber->ev_callback = callback;
    fiber->ev_stream = stream;
    janet_ev_inc_refcount();
    janet_gcroot(janet_wrap_abstract(stream));
    fiber->ev_state = state;
    callback(fiber, JANET_ASYNC_EVENT_INIT);
}
```

[`janet_await` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L610-L614):

```c
/* Shorthand to yield to event loop */
void janet_await(void) {
    /* Store the fiber in a global table */
    janet_signalv(JANET_SIGNAL_EVENT, janet_wrap_nil());
}
```

[`janet_signalv` in `capi.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/capi.c#L63-L78):

```c
void janet_signalv(JanetSignal sig, Janet message) {
    if (janet_vm.return_reg != NULL) {
        *janet_vm.return_reg = message;
        if (NULL != janet_vm.fiber) {
            janet_vm.fiber->flags |= JANET_FIBER_DID_LONGJUMP;
        }
#if defined(JANET_BSD) || defined(JANET_APPLE)
        _longjmp(*janet_vm.signal_buf, sig);
#else
        longjmp(*janet_vm.signal_buf, sig);
#endif
    } else {
        const char *str = (const char *)janet_formatc("janet top level signal - %v\n", message);
        janet_top_level_signal(str);
    }
}
```

[`janet_ev_read` in `ev.c`](https://github.com/janet-lang/janet/blob/6535d72bd409eaeb01ee069d7a4b0e1fd46209c7/src/core/ev.c#L2471-L2473):

```c
JANET_NO_RETURN void janet_ev_read(JanetStream *stream, JanetBuffer *buf, int32_t nbytes) {
    janet_ev_read_generic(stream, buf, nbytes, 0, JANET_ASYNC_READMODE_READ, 0);
}
```
