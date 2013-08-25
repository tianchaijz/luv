luv
===

Bare [libuv][] bindings for [lua][]

This is [lua][]/[luajit][] bindings for [libuv][], the evented library that powers [node.js][].  If you want a more opinionated framework with an improved module system over what stock lua provides, look at the [luvit][] project.

## Build Instructions

This is a native module using the lua C addon API.  Libuv is included as a git submodule and is required to build.

To use, first setup a lua build environment.  The easiest way on linux is to download the latest lua or luajit (I recommend luajit) release and build from source and install to `/usr/local`.  Or if you use your system's native package management make sure to install the development package for lua.

Once you have a lua build environment, use the included `Makefile` to build `luv.so`.  This single file will contain all of libuv along with the nessecary lua bindings to make it a lua module.

## API

Luv tries to stick as close to the libuv API as possible while still being easy to use.  This means it's mostly a flat list of functions.

I'll show some basic examples and go over the lua API here, but extensive docs can be found at the [uv book][] site.

## Constructors

Libuv is a sort-of object oriented API.  There is a hierchary of object types (implemented as structs internally).

 - `uv_handle_t` - The base abstract type.  All subtypes can also be used in functions that expect handles.
   - `uv_timer_t` - The type for timeouts and intervals.  Create instances using `new_timer()`.
   - `uv_stream_t` - A shared abstract type for all stream-like types.
     - `uv_tcp_t` - Handle type for TCP servers and clients.  Create using `new_tcp()`.
     - `uv_tty_t` - A special stream for wrapping TTY file descriptors.  Create using `new_tty(fd, readable)`.
     - `uv_pipe` - A 

### new_tcp() -> uv_tcp_t

Create a new tcp userdata.  This can later be turned into a TCP server or client.

```lua
local server = uv.new_tcp()
```

### new_tty(fd, readable) -> uv_tty_t

Create a new tty userdata.  This is good for wrapping stdin, stdout, and stderr when the process is used via tty.  The tty type inherits from the stream type in libuv.

`fd` is an integer (0 for stdin, 1 for stdout, 2 for stderr). `readable` is a boolean (usually only for stdin).

```lua
local std = {
  in = uv.new_tty(0, true),
  out = uv.new_tty(1),
  err = uv.new_tty(2)
}
```

### new_pipe(ipc) -> uv_pipe_t

Create a new pipe userdata.  Pipes can be used for many things such as named pipes, anonymous pipes, child_process stdio, or even file stream pipes.  `ipc` is normally `false` or omitted except for advanced usage.

```lua
local pipe = uv.new_pipe()
```

### guess_handle(fd) -> type string

Given a file descriptor, this will guess the type of stream.  This is especially useful to find out if stdin, stdout, and stderr are indeed TTY streams or if they are something else.

```lua
local stdin
if uv.guess_handle(0) == "TTY" then
  stdin = uv.new_tty(0)
else
  ...
end
```

### update_time()

Force libuv to internally update it's time.

### now() -> timestamp

Get a relative timestamp. The value is in milliseconds, but is only good for relative measurements, not dates.  Use `update_time()` if you are getting stale values.

```lua
uv.update_time()
local start = uv.now()

-- Do something slow

uv.update_time()
local delta = uv.now() - start
```

### loadavg() -> load

Get the load average as 3 returned integers.

### execpath() -> path to executable

Gives the absolute path to the executable.  This would usually be wherever luajit or lua is installed on your system.  Useful for writing portable applications that embed their own copy of lua and want to load resources relative to it in the filesystem.  This is much more reliable and useful than simply getting `argv[0]`.

```lua
local path = uv.execpath()
```

### cwd() -> path to current working directory

```lua
local cwd = uv.cwd()
```

### chdir(path)

Change current working directory to a new path.

### get_free_memory() -> bytes of free mem

Get the number of bytes of free memory.

### get_total_memory() -> bytes of total mem

Get the number of byyed of total memory.

### get_process_title() -> title

NOTE: currently does not work on some platforms because we're a library and can't access `argc` and `argv` directly.

Get the current process title as reported in tools like top.

### set_process_title(title)

NOTE: currently does not work on some platforms because we're a library and can't access `argc` and `argv` directly.

Get the current process title as reported in tools like top.

### hrtime() -> timestamp

High-resolution timestamp in ms as a floating point number.  Value is relative and not absolute like a date based timestamp.

### uptime() -> uptime

Return the computer's current uptime in seconds as a floating point number.

### cpu_info() -> info

Return detailed information about the CPUs.

```lua
> uv.cpu_info()
{
  { times = table: 0x41133530, model = "Intel(R) Core(TM) i5-3427U CPU @ 1.80GHz", speed = 1800 },
  { times = table: 0x41129e00, model = "Intel(R) Core(TM) i5-3427U CPU @ 1.80GHz", speed = 1800 },
  { times = table: 0x4112a020, model = "Intel(R) Core(TM) i5-3427U CPU @ 1.80GHz", speed = 1801 },
  { times = table: 0x4112a278, model = "Intel(R) Core(TM) i5-3427U CPU @ 1.80GHz", speed = 1800 }
}
> uv.cpu_info()[1]
{
  times = { irq = 600, user = 8959100, idle = 78622000, sys = 2754000, nice = 0 },
  model = "Intel(R) Core(TM) i5-3427U CPU @ 1.80GHz",
  speed = 1800
}
```

### interface_addresses() -> info

Return detailed information about network interfaces.

```lua
> uv.interface_addresses()
{ lo = { table: 0x4112fb90, table: 0x4112f460 }, wlan0 = { table: 0x41132360, table: 0x41132e68 } }
> uv.interface_addresses().lo
{
  { address = "127.0.0.1", internal = true, family = "IPv4" },
  { address = "::1", internal = true, family = "IPv6" }
}
> uv.interface_addresses().wlan0
{
  { address = "172.19.131.166", internal = false, family = "IPv4" },
  { address = "fe80::9e2a:70ff:fe0c:ba8b", internal = false, family = "IPv6" }
}
```

## is_active(handle) -> boolean

Returns 1 if the prepare/check/idle/timer handle has been started, 0 otherwise. For other handle types this always returns 1.

### walk(callback)

Walk all active handles.

```lua
uv.walk(function (handle, description)
  -- Do something with this information.
end)
```

### ref(handle)

Increment the refcount of a handle manually.  Only handles with positive refcounts will hold the main event loop open.

### unref(handle)

Decrement the refcount of a handle manually.  Only handles with positive refcounts will hold the main event loop open.

This is useful for things like a background interval that doesn't prevent the process from exiting naturally.

### close(handle)

Tell a handle.  Attach an `onclose` handler if you wish to be notified when it's complete.

```lua
function handle:onclose()
  -- Now it's closed
end
uv.close(handle)
```

### is_closing(handle) -> boolean

Lets you know if a handle is already closing.


## Timers

Luv provides non-blocking timers so that you can schedule code to run on an interval or after a period of timeout.

The `uv_timer_t` handle type if a direct desccendent of `uv_handle_t`.

Here is an example of how to implement a JavaScript style `setTimeout` function:

```lua
local function setTimeout(fn, ms)
  local handle = uv.new_timer()
  function handle:ontimeout()
    fn()
    uv.timer_stop(handle)
    ub.close(handle)
  end
  uv.timer_start(handle)
end
```

And here is an example of implementing a JavaScript style `setInterval` function:

```lua
local function setInterval(fn, ms)
  local handle = uv.new_timer()
  function handle:ontimeout()
    fn();
  end
  uv.timer_start(handle, ms, ms)
  return handle
end

local clearTimer(handle)
  uv.timer_stop(handle)
  uv.close(handle)
end
```

And here is a more advanced example that creates a repeating timer that halves the delay each iteration till it's down to 1ms.

```lua
local handle = uv.new_timer()
local delay = 1024
function handle:ontimeout()
  p("tick", delay)
  delay = delay / 2
  if delay >= 1 then
    uv.timer_set_repeat(handle, delay)
    uv.timer_again(handle)
  else
    uv.timer_stop(handle)
    uv.close(handle)
    p("done")
  end
end
uv.timer_start(handle, delay, 0)
```

### new_timer() -> uv_timer_t

Create a new timer userdata.  Later this can be turned into a timeout or interval using the timer functions.

### timer_start(timer, timeout, repeat)

Given a timer handle, start it with a timeout and repeat.  To create a one-shot timeout, set repeat to zero.  For a recurring interval, set the same value to repeat.  Attach the `ontimeout` listener to know when it timeouts.

### timer_stop(timer)

Stop a timer.  Use this to cancel timeouts or intervals.

### timer_again(timer)

Use this to resume a stopped timer.

### timer_set_repeat(timer, repeat)

Give the timer a new repeat value.  If it was stopped, you'll need to resume it as well after setting this.

### timer_get_repeat(timer) -> repeat

Read the repeat value out of a timer instance

## Streams

Stream is a common interface between several libuv types.  Concrete types that are also streams include `uv_tty_t`, `uv_tcp_t`, and `uv_pipe_t`.

> 
>   {"write", luv_write},
>   {"shutdown", luv_shutdown},
>   {"read_start", luv_read_start},
>   {"read_stop", luv_read_stop},
>   {"listen", luv_listen},
>   {"accept", luv_accept},
>   {"write", luv_write},
>   {"is_readable", luv_is_readable},
>   {"is_writable", luv_is_writable},
> 
>   {"tcp_bind", luv_tcp_bind},
>   {"tcp_getsockname", luv_tcp_getsockname},
>   {"tcp_getpeername", luv_tcp_getpeername},
>   {"tcp_connect", luv_tcp_connect},
>   {"tcp_open", luv_tcp_open},
>   {"tcp_nodelay", luv_tcp_nodelay},
>   {"tcp_keepalive", luv_tcp_keepalive},
> 
>   {"tty_set_mode", luv_tty_set_mode},
>   {"tty_reset_mode", luv_tty_reset_mode},
>   {"tty_get_winsize", luv_tty_get_winsize},
> 
>   {"pipe_open", luv_pipe_open},
>   {"pipe_bind", luv_pipe_bind},
>   {"pipe_connect", luv_pipe_connect},
> 
>   {"spawn", luv_spawn},
>   {"kill", luv_kill},
>   {"process_kill", luv_process_kill},
> 
>   {"fs_open", luv_fs_open},
>   {"fs_close", luv_fs_close},
>   {"fs_read", luv_fs_read},
>   {"fs_write", luv_fs_write},
>   {"fs_stat", luv_fs_stat},
>   {"fs_fstat", luv_fs_fstat},
>   {"fs_lstat", luv_fs_lstat},
>   {"fs_unlink", luv_fs_unlink},
>   {"fs_mkdir", luv_fs_mkdir},
>   {"fs_rmdir", luv_fs_rmdir},
>   {"fs_readdir", luv_fs_readdir},
>   {"fs_rename", luv_fs_rename},
>   {"fs_fsync", luv_fs_fsync},
>   {"fs_fdatasync", luv_fs_fdatasync},
>   {"fs_ftruncate", luv_fs_ftruncate},
>   {"fs_sendfile", luv_fs_sendfile},
>   {"fs_chmod", luv_fs_chmod},
>   {"fs_utime", luv_fs_utime},
>   {"fs_futime", luv_fs_futime},
>   {"fs_link", luv_fs_link},
>   {"fs_symlink", luv_fs_symlink},
>   {"fs_readlink", luv_fs_readlink},
>   {"fs_fchmod", luv_fs_fchmod},
>   {"fs_chown", luv_fs_chown},
>   {"fs_fchown", luv_fs_fchown},
> 

[lua]: http://www.lua.org/
[luajit]: http://luajit.org/
[libuv]: https://github.com/joyent/libuv
[node.js]: http://nodejs.org/
[luvit]: http://luvit.io/
[uv book]: http://nikhilm.github.io/uvbook/