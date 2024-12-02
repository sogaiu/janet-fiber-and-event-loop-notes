# net/read

## Information

`(net/read stream nbytes &opt buf timeout)`

Read up to n bytes from a stream, suspending the current fiber until
the bytes are available.

`n` can also be the keyword `:all` to read into the buffer until end
of stream.

If less than n bytes are available (and more than 0), will push those
bytes and return early.

Takes an optional timeout in seconds, after which will raise an error.

Returns a buffer with up to n more bytes in it, or raises an error if
the read failed.

## Sample Code

Success with reading one byte, returns buffer:

```janet
(def [host port] ["127.0.0.1" "9899"])

(net/server host port
            (fn [stream]
              (net/write stream @"fun")))

(def c (net/connect host port))

(net/read c 1)
# =>
@"f"
```

Success using `:all`, returns buffer:

```janet
(def [host port] ["127.0.0.1" "9899"])

(net/server host port
            (fn [stream]
              (net/write stream @"fun")
              (net/close stream)))

(def c (net/connect host port))

(net/read c :all)
# =>
@"fun"
```

Returns `nil` if end of stream is reached:

```janet
(def [host port] ["127.0.0.1" "9899"])

(net/server host port
            (fn [stream]
              (net/write stream @"fun")
              (net/close stream)))

(def c (net/connect host port))

(net/read c :all)
# =>
@"fun"

(net/read c :all)
# =>
nil
```

Timeout expired, operation cancelled, and error raised:

```janet
(def [host port] ["127.0.0.1" "9899"])

(def timeout 1)

(net/server host port
            (fn [stream]
              (ev/sleep 2)
              (net/write stream @"fun")
              (net/close stream)))

(def c (net/connect host port))

(try
  (net/read c 1 @"" timeout)
  ([e]
    (eprintf "net/read errored: %s" e)
    :bye))
# =>
:bye
```

Sample output:

```
net/read errored: timeout
```

## C Implementation

[`net/read` in `net.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/net.c#L850-L861):

```c
    JanetStream *stream = janet_getabstract(argv, 0, &janet_stream_type);
    janet_stream_flags(stream, JANET_STREAM_READABLE | JANET_STREAM_SOCKET);
    JanetBuffer *buffer = janet_optbuffer(argv, argc, 2, 10);
    double to = janet_optnumber(argv, argc, 3, INFINITY);
    if (janet_keyeq(argv[1], "all")) {
        if (to != INFINITY) janet_addtimeout(to);
        janet_ev_recvchunk(stream, buffer, INT32_MAX, MSG_NOSIGNAL);
    } else {
        int32_t n = janet_getnat(argv, 1);
        if (to != INFINITY) janet_addtimeout(to);
        janet_ev_recv(stream, buffer, n, MSG_NOSIGNAL);
    }
```

[`JanetStream` in janet.h](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/include/janet.h#L615-L625):

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

[`janet_stream_flags` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L2212-L2225):

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

[`janet_addtimeout` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L616-L626):

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

[`janet_ev_recvchunk` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L2493-L2495):

```c
JANET_NO_RETURN void janet_ev_recvchunk(JanetStream *stream, JanetBuffer *buf, int32_t nbytes, int flags) {
    janet_ev_read_generic(stream, buf, nbytes, 1, JANET_ASYNC_READMODE_RECV, flags);
}
```

[`janet_even_read_generic` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L2468-L2481):

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

[`StateRead` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L2259-L2285):

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

[`janet_async_start` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L301-L304):

```c
void janet_async_start(JanetStream *stream, JanetAsyncMode mode, JanetEVCallback callback, void *state) {
    janet_async_start_fiber(janet_vm.root_fiber, stream, mode, callback, state);
    janet_await();
}
```

[`janet_async_start_fiber` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L285-L299):

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

[`janet_await` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L610-L614):

```c
/* Shorthand to yield to event loop */
void janet_await(void) {
    /* Store the fiber in a global table */
    janet_signalv(JANET_SIGNAL_EVENT, janet_wrap_nil());
}
```

[`janet_signalv` in `capi.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/capi.c#L63-L78):

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

[`janet_ev_recv` in `ev.c`](https://github.com/janet-lang/janet/blob/a0eeb630e70a0271d4cd72e153dbb39078ffbcbb/src/core/ev.c#L2490-L2492):

```c
JANET_NO_RETURN void janet_ev_recv(JanetStream *stream, JanetBuffer *buf, int32_t nbytes, int flags) {
    janet_ev_read_generic(stream, buf, nbytes, 0, JANET_ASYNC_READMODE_RECV, flags);
}
```

