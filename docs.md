# LibUV in Lua

The [luv][] project provides access to the multi-platform support library
[libuv][] to lua code.  It was primariliy developed for the [luvit][] project as
the `uv` builtin module, but can be used in other lua environments.

### TCP Echo Server Example

Here is a small example showing a TCP echo server:

```lua
local uv = require('uv')

local server = uv.new_tcp()
server:bind("127.0.0.1", 1337)
server:listen(128, function (err)
  assert(not err, err)
  local client = uv.new_tcp()
  server:accept(client)
  client:read_start(function (err, chunk)
    assert(not err, err)
    if chunk then
      client:write(chunk)
    else
      client:shutdown()
      client:close()
    end
  end)
end)
print("TCP server listening at 127.0.0.1 port 1337")
uv.run()
```

### Methods vs Functions

As a quick note, [libuv][] is a C library and as such, there are no such things
as methods.  The [luv][] bindings allow calling the libuv functions as either
functions or methods.  For example, calling `server:bind(host, port)` is
equivalent to calling `uv.tcp_bind(server, host, port)`.  All wrapped uv types
in lua have method shortcuts where is makes sense.  Some are even renamed
shorter like the `tcp_` prefix that removed in method form.  Under the hood it's
the exact same C function.

## Table Of Contents

The rest of the docs are organized by libuv type.  There is some hierarchy as
most types are considered handles and some are considered streams.

 - [`uv_loop_t`][] — Event loop
 - [`uv_handle_t`][] — Base handle
   - [`uv_timer_t`][] — Timer handle
   - [`uv_prepare_t`][] — Prepare handle
   - [`uv_check_t`][] — Check handle
   - [`uv_idle_t`][] — Idle handle
   - [`uv_async_t`][] — Async handle
   - [`uv_poll_t`][] — Poll handle
   - [`uv_signal_t`][] — Signal handle
   - [`uv_process_t`][] — Process handle
   - [`uv_stream_t`][] — Stream handle
     - [`uv_tcp_t`][] — TCP handle
     - [`uv_pipe_t`][] — Pipe handle
     - [`uv_tty_t`][] — TTY handle
   - [`uv_udp_t`][] — UDP handle
   - [`uv_fs_event_t`][] — FS Event handle
   - [`uv_fs_poll_t`][] — FS Poll handle
 - [Filesystem operations][]
 - [DNS utility functions][]
 - [Miscellaneous utilities][]

## `uv_loop_t` — Event loop

[`uv_loop_t`]: #uv_loop_t--event-loop

The event loop is the central part of libuv’s functionality. It takes care of
polling for i/o and scheduling callbacks to be run based on different sources of
events.

In [luv][], there is an implicit uv loop for every lua state that loads the
library.  You can use this library in an multithreaded environment as long as
each thread has it's own lua state with corresponsding own uv loop.

### `uv.loop_close()`

Closes all internal loop resources. This function must only be called once the
loop has finished its execution or it will raise a UV_EBUSY error.

### `uv.run([mode])`

> optional `mode` defaults to `"default"`

This function runs the event loop. It will act differently depending on the
specified mode:

 - `"default"`: Runs the event loop until there are no more active and
   referenced handles or requests. Always returns `false`.

 - `"once"`: Poll for i/o once. Note that this function blocks if there are no
   pending callbacks. Returns `false` when done (no active handles or requests
   left), or `true` if more callbacks are expected (meaning you should run
   the event loop again sometime in the future).

 - `"nowait"`: Poll for i/o once but don’t block if there are no
   pending callbacks. Returns `false` if done (no active handles or requests
   left), or `true` if more callbacks are expected (meaning you should run
   the event loop again sometime in the future).

Luvit will implicitly call `uv.run()` after loading user code, but if you use
the `luv` bindings directly, you need to call this after registering your
initial set of event callbacks to start the event loop.

### `uv.loop_alive()`

Returns true if there are active handles or request in the loop.

### `uv.stop()`

Stop the event loop, causing `uv_run()` to end as soon as possible. This
will happen not sooner than the next loop iteration. If this function was called
before blocking for i/o, the loop won’t block for i/o on this iteration.

### `uv.backend_fd()`

Get backend file descriptor. Only kqueue, epoll and event ports are supported.

This can be used in conjunction with `uv_run("nowait")` to poll in one thread
and run the event loop’s callbacks in another.

**Note**: Embedding a kqueue fd in another kqueue pollset doesn’t work on all
platforms. It’s not an error to add the fd but it never generates events.

### `uv.backend_timeout()`

Get the poll timeout. The return value is in milliseconds, or -1 for no timeout.

### `uv.now()`

Return the current timestamp in milliseconds. The timestamp is cached at the
start of the event loop tick, see `uv.update_time()` for details and rationale.

The timestamp increases monotonically from some arbitrary point in time. Don’t
make assumptions about the starting point, you will only get disappointed.

**Note**: Use `uv.hrtime()` if you need sub-millisecond granularity.

### `uv.update_time()`

Update the event loop’s concept of “now”. Libuv caches the current time at the
start of the event loop tick in order to reduce the number of time-related
system calls.

You won’t normally need to call this function unless you have callbacks that
block the event loop for longer periods of time, where “longer” is somewhat
subjective but probably on the order of a millisecond or more.

### `uv.walk(callback)`

Walk the list of handles: `callback` will be executed with the handle.

```lua
-- Example usage of uv.walk to close all handles that aren't already closing.
uv.walk(function (handle)
  if not handle:is_closing() then
    handle:close()
  end
end)
```

## `uv_handle_t` — Base handle

[`uv_handle_t`]: #uv_handle_t--base-handle

`uv_handle_t` is the base type for all libuv handle types.

Structures are aligned so that any libuv handle can be cast to `uv_handle_t`.
All API functions defined here work with any handle type.

### `uv.is_active(handle)`

> method form `handle:is_active()`

Returns `true` if the handle is active, `false` if it’s inactive. What “active” means depends on the type of handle:

 - A [`uv_async_t`][] handle is always active and cannot be deactivated, except
   by closing it with uv_close().

 - A [`uv_pipe_t`][], [`uv_tcp_t`][], [`uv_udp_t`][], etc. handlebasically any
   handle that deals with i/ois active when it is doing something that
   involves i/o, like reading, writing, connecting, accepting new connections,
   etc.

 - A [`uv_check_t`][], [`uv_idle_t`][], [`uv_timer_t`][], etc. handle is active
   when it has been started with a call to `uv.check_start()`,
   `uv.idle_start()`, etc.

Rule of thumb: if a handle of type `uv_foo_t` has a `uv.foo_start()` function,
then it’s active from the moment that function is called. Likewise,
`uv.foo_stop()` deactivates the handle again.

### `uv.is_closing(handle)`

> method form `handle:is_closing()`

Returns `true` if the handle is closing or closed, `false` otherwise.

**Note**: This function should only be used between the initialization of the
handle and the arrival of the close callback.

### `uv.close(handle, callback)`

> method form `handle:close(callback)`

Request handle to be closed. `callback` will be called asynchronously after this
call. This MUST be called on each handle before memory is released.

Handles that wrap file descriptors are closed immediately but `callback` will
still be deferred to the next iteration of the event loop. It gives you a chance
to free up any resources associated with the handle.

In-progress requests, like `uv_connect_t` or `uv_write_t`, are cancelled and
have their callbacks called asynchronously with `status=UV_ECANCELED`.

### `uv.ref(handle)`

> method form `handle:ref()`

Reference the given handle. References are idempotent, that is, if a handle is
already referenced calling this function again will have no effect.

See [Reference counting][].

### `uv.unref(handle)`

> method form `handle:unref()`

Un-reference the given handle. References are idempotent, that is, if a handle
is not referenced calling this function again will have no effect.

See [Reference counting][].

### `uv.has_ref(handle)`

> method form `handle:has_ref()`

Returns `true` if the handle referenced, `false` otherwise.

See [Reference counting][].

### `uv.send_buffer_size(handle, [size]) -> size`

> method form `handle:send_buffer_size(size)`

Gets or sets the size of the send buffer that the operating system uses for the
socket.

If `size` is omitted, it will return the current send buffer size, otherwise it
will use `size` to set the new send buffer size.

This function works for TCP, pipe and UDP handles on Unix and for TCP and UDP
handles on Windows.

**Note**: Linux will set double the size and return double the size of the
original set value.

### `uv.recv_buffer_size(handle, [size])`

> method form `handle:recv_buffer_size(size)`

Gets or sets the size of the receive buffer that the operating system uses for
the socket.

If `size` is omitted, it will return the current receive buffer size, otherwise
it will use `size` to set the new receive buffer size.

This function works for TCP, pipe and UDP handles on Unix and for TCP and UDP
handles on Windows.

**Note**: Linux will set double the size and return double the size of the
original set value.

### `uv.fileno(handle)`

> method form `handle:fileno()`

Gets the platform dependent file descriptor equivalent.

The following handles are supported: TCP, pipes, TTY, UDP and poll. Passing any
other handle type will fail with UV_EINVAL.

If a handle doesn’t have an attached file descriptor yet or the handle itself
has been closed, this function will return UV_EBADF.

**Warning**: Be very careful when using this function. libuv assumes it’s in
control of the file descriptor so any change to it may lead to malfunction.

## Reference counting

[reference counting]: #reference-counting

The libuv event loop (if run in the default mode) will run until there are no
active and referenced handles left. The user can force the loop to exit early by
unreferencing handles which are active, for example by calling `uv.unref()`
after calling `uv.timer_start()`.

A handle can be referenced or unreferenced, the refcounting scheme doesn’t use a
counter, so both operations are idempotent.

All handles are referenced when active by default, see `uv.is_active()` for a
more detailed explanation on what being active involves.

## `uv_timer_t` — Timer handle

[`uv_timer_t`]: #uv_timer_t--timer-handle

Timer handles are used to schedule callbacks to be called in the future.

### `uv.new_timer() -> timer`

Creates and initialized a new `uv_timer_t` returns the lua userdata wrapping it.

```lua
-- Creating a simple setTimeout wrapper
local function setTimeout(timeout, callback)
  local timer = uv.new_timer()
  timer:start(timeout, 0, function ()
    timer:stop()
    timer:close()
    callback()
  end)
  return timer
end

-- Creating a simple setInterval wrapper
local function setInterval(interval, callback)
  local timer = uv.new_timer()
  timer:start(interval, interval, function ()
    timer:stop()
    timer:close()
    callback()
  end)
  return timer
end

-- And clearInterval
local function clearInterval(timer)
  timer:stop()
  timer:close()
end
```

### `uv.timer_start(timer, timeout, repeat, callback)`

> method form `timer:start(timeout, repeat, callback)`

Start the timer. `timeout` and `repeat` are in milliseconds.

If `timeout` is zero, the callback fires on the next event loop iteration. If
`repeat` is non-zero, the callback fires first after timeout milliseconds and
then repeatedly after repeat milliseconds.

### `uv.timer_stop(timer)`

> method form `timer:stop()`

Stop the timer, the callback will not be called anymore.

### `uv.timer_again(timer)`

> method form `timer:again()`

Stop the timer, and if it is repeating restart it using the repeat value as the timeout. If the timer has never been started before it raises `EINVAL`.

### `uv.timer_set_repeat(timer, repeat)`

> method form `timer:set_repeat(repeat)`

Set the repeat value in milliseconds.

**Note**: If the repeat value is set from a timer callback it does not
immediately take effect. If the timer was non-repeating before, it will   have
been stopped. If it was repeating, then the old repeat value will   have been
used to schedule the next timeout.

### `uv.timer_get_repeat(timer) -> repeat`

> method form `timer:get_repeat() -> repeat`

Get the timer repeat value.

## `uv_prepare_t` — Prepare handle

[`uv_prepare_t`]: #uv_prepare_t--prepare-handle

## `uv_check_t` — Check handle

[`uv_check_t`]: #uv_check_t--check-handle

## `uv_idle_t` — Idle handle

[`uv_idle_t`]: #uv_idle_t--idle-handle

## `uv_async_t` — Async handle

[`uv_async_t`]: #uv_async_t--async-handle

## `uv_poll_t` — Poll handle

[`uv_poll_t`]: #uv_poll_t--poll-handle

## `uv_signal_t` — Signal handle

[`uv_signal_t`]: #uv_signal_t--signal-handle

## `uv_process_t` — Process handle

[`uv_process_t`]: #uv_process_t--process-handle

## `uv_stream_t` — Stream handle

[`uv_stream_t`]: #uv_stream_t--stream-handle

Stream handles provide an abstraction of a duplex communication channel.
[`uv_stream_t`][] is an abstract type, libuv provides 3 stream implementations in
the form of [`uv_tcp_t`][], [`uv_pipe_t`][] and [`uv_tty_t`][].

### `uv.shutdown(stream, [callback]) -> req`

> (method form `stream:shutdown([callback]) -> req`)

Shutdown the outgoing (write) side of a duplex stream. It waits for pending
write requests to complete. The callback is called after
shutdown is complete.

### `uv.listen(stream, backlog, callback)`

> (method form `stream:listen(backlog, callback)`)

Start listening for incoming connections. `backlog` indicates the number of
connections the kernel might queue, same as `listen(2)`. When a new incoming
connection is received the callback is called.

### `uv.accept(stream, client_stream)`

> (method form `stream:accept(client_stream)`)

This call is used in conjunction with `uv.listen()` to accept incoming
connections. Call this function after receiving a callback to accept the
connection.

When the connection callback is called it is guaranteed that this function
will complete successfully the first time. If you attempt to use it more than
once, it may fail. It is suggested to only call this function once per
connection call.

```lua
server:listen(128, function (err)
  local client = uv.new_tcp()
  server:accept(client)
end)
```

### `uv.read_start(stream, callback)`

> (method form `stream:read_start(callback)`)

Callback is of the form `(err, data)`.

Read data from an incoming stream. The callback will be made several times until
there is no more data to read or `uv.read_stop()` is called. When we’ve reached
EOF, `data` will be `nil`.

```lua
stream:read_start(function (err, chunk)
  if err then
    -- handle read error
  elseif chunk then
    -- handle data
  else
    -- handle disconnect
  end
end)
```

### `uv.read_stop(stream)`

> (method form `stream:read_stop()`)

Stop reading data from the stream. The read callback will no longer be called.

### `uv.write(stream, data, [callback])`

> (method form `stream:write(data, [callback])`)

Write data to stream.

`data` can either be a lua string or a table of strings.  If a table is passed
in, the C backend will use writev to send all strings in a single system call.

The optional `callback` is for knowing when the write is
complete.

### `uv.write2(stream, data, send_handle, callback)`

> (method form `stream:write2(data, send_handle, callback)`)

Extended write function for sending handles over a pipe. The pipe must be
initialized with ip option to `true`.

**Note: `send_handle` must be a TCP socket or pipe, which is a server or a
connection (listening or connected state). Bound sockets or pipes will be
assumed to be servers.

### `uv.try_write(stream, data)`

> (method form `stream:try_write(data)`)

Same as `uv.write()`, but won’t queue a write request if it can’t be completed
immediately.

Will return number of bytes written (can be less than the supplied buffer size).

### `uv.is_readable(stream)`

> (method form `stream:is_readable()`)

Returns `true` if the stream is readable, `false` otherwise.

### `uv.is_writable(stream)`

> (method form `stream:is_writable()`)

Returns `true` if the stream is writable, `false` otherwise.

### `uv.stream_set_blocking(stream, blocking)`

> (method form `stream:set_blocking(blocking)`)

Enable or disable blocking mode for a stream.

When blocking mode is enabled all writes complete synchronously. The interface
remains unchanged otherwise, e.g. completion or failure of the operation will
still be reported through a callback which is made asynchronously.

**Warning**: Relying too much on this API is not recommended. It is likely to
change significantly in the future. Currently this only works on Windows and
only for uv_pipe_t handles. Also libuv currently makes no ordering guarantee
when the blocking mode is changed after write requests have already been
submitted. Therefore it is recommended to set the blocking mode immediately
after opening or creating the stream.

## `uv_tcp_t` — TCP handle

[`uv_tcp_t`]: #uv_tcp_t--tcp-handle

TCP handles are used to represent both TCP streams and servers.

`uv_tcp_t` is a ‘subclass’ of [`uv_stream_t`](#uv_stream_t--stream-handle).

### `uv.new_tcp() -> tcp`

### `uv.tcp_open(tcp, sock)`

> (method form `tcp:open(sock)`)

### `uv.tcp_nodelay(tcp, enable)`

> (method form `tcp:nodelay(enable)`)

### `uv.tcp_keepalive(tcp, enable)`

> (method form `tcp:keepalive(enable)`)

### `uv.tcp_simultaneous_accepts(tcp, enable)`

> (method form `tcp:simultaneous_accepts(enable)`)

### `uv.tcp_bind(tcp, host, port)`

> (method form `tcp:bind(host, port)`)

### `uv.tcp_getpeername(tcp)`

> (method form `tcp:getpeername()`)

### `uv.tcp_getsockname(tcp)`

> (method form `tcp:getsockname()`)

### `uv.tcp_connect(tcp, host, port, callback) -> req`

> (method form `tcp:connect(host, port, callback) -> req`)

### `uv.tcp_write_queue_size(tcp) -> size`

> (method form `tcp:write_queue_size() -> size`)


## `uv_pipe_t` — Pipe handle

[`uv_pipe_t`]: #uv_pipe_t--pipe-handle

## `uv_tty_t` — TTY handle

[`uv_tty_t`]: #uv_tty_t--tty-handle

## `uv_udp_t` — UDP handle

[`uv_udp_t`]: #uv_udp_t--udp-handle

## `uv_fs_event_t` — FS Event handle

[`uv_fs_event_t`]: #uv_fs_event_t--fs-event-handle

## `uv_fs_poll_t` — FS Poll handle

[`uv_fs_poll_t`]: #uv_fs_poll_t--fs-poll-handle

## Filesystem operations

[Filesystem operations]:#filesystem-operations

## DNS utility functions

[DNS utility functions]: #dns-utility-functions

## Miscellaneous utilities

[Miscellaneous utilities]: #miscellaneous-utilities


[luv]: https://github.com/luvit/luv
[luvit]: https://github.com/luvit/luvit
[libuv]: https://github.com/libuv/libuv
