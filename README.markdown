[![Build Status](https://travis-ci.org/Kong/lua-resty-worker-events.svg?branch=master)](https://travis-ci.org/Kong/lua-resty-worker-events)

lua-resty-worker-events
=======================

Inter process events for Nginx worker processes

Table of Contents
=================

* [Name](#name)
* [Status](#status)
* [Synopsis](#synopsis)
* [Description](#description)
* [Troubleshooting](#troubleshooting)
* [Methods](#methods)
    * [configure](#configure)
    * [configured](#configured)
    * [event_list](#event_list)
    * [poll](#poll)
    * [post](#post)
    * [post_local](#post_local)
    * [register](#register)
    * [register_weak](#register_weak)
    * [unregister](#unregister)
* [Installation](#installation)
* [Bugs and Patches](#bugs-and-patches)
* [Author](#author)
* [Copyright and License](#copyright-and-license)
* [History](#history)
* [See Also](#see-also)

Status
======

This library is production ready.

Synopsis
========

```nginx
http {
    lua_package_path "/path/to/lua-resty-worker-events/lib/?.lua;;";

    # the size depends on the number of event to handle:
    lua_shared_dict process_events 1m;

    init_by_lua_block {
        -- init the correct event start id, avoid some event lost during reload
        require("resty.worker.events").init("process_events")
        package.loaded["resty.worker.events"] = nil
    }

    init_worker_by_lua_block {
        local ev = require "resty.worker.events"

        local handler = function(data, event, source, pid)
            print("received event; source=",source,
                  ", event=",event,
                  ", data=", tostring(data),
                  ", from process ",pid)
        end

        ev.register(handler)

        local ok, err = ev.configure {
            shm = "process_events", -- defined by "lua_shared_dict"
            timeout = 2,            -- life time of unique event data in shm
            interval = 1,           -- poll interval (seconds)

            wait_interval = 0.010,  -- wait before retry fetching event data
            wait_max = 0.5,         -- max wait time before discarding event
            shm_retries = 999,      -- retries for shm fragmentation (no memory)
        }
        if not ok then
            ngx.log(ngx.ERR, "failed to start event system: ", err)
            return
        end
    }

    server {
        ...

        # example for polling:
        location = /some/path {

            default_type text/plain;
            content_by_lua_block {
                -- manually call `poll` to stay up to date, can be used instead,
                -- or together with the timer interval. Polling is efficient,
                -- so if staying up-to-date is important, this is preferred.
                require("resty.worker.events").poll()

                -- do regular stuff here

            }
        }
    }
}
```

Description
===========

[Back to TOC](#table-of-contents)

This module provides a way to send events to the other worker processes in an Nginx
server. Communication is through a shared memory zone where event data will be stored.

The order of events in all workers is __guaranteed__ to be the same.

The worker process will setup a timer to check for events in the background. The
module follows a singleton pattern and hence runs once per worker. If staying
up-to-date is important though, the interval can be set to a lesser frequency and a
call to [poll](#poll) upon each request received makes sure everything is handled
as soon as possible.

The design allows for 3 usecases;

1. broadcast an event to all workers processes, see [post](#post). In this case
the order of the events is guaranteed to be the same in all worker processes. Example;
a healthcheck running in one worker, but informing all workers of a failed
upstream node.
2. broadcast an event to the local worker only, see [post_local](#post_local).
3. coalesce external events to a single action. Example; all workers watch
external events indicating an in-memory cache needs to be refreshed. When
receiving it they all post it with a unique event hash (all workers generate the
same hash), see `unique` parameter of [post](#post). Now only 1 worker will
receive the event _only once_, so only one worker will hit the upstream
database to refresh the in-memory data.

This module itself will fire two events with `source="resty-worker-events"`;
 * `event="started"` when the module is first configured (note: the event handler must be
   [registered](#register) before calling [configure](#configure) to be able to catch the event)
 * `event="stopping"` when the worker process exits (based on a timer `premature` setting)

See [event_list](#event_list) for using events without hardcoded magic
values/strings.


Troubleshooting
================

To properly size the shm, it is important to understand how it is being used.
Event data is stored in the shm to pass it to the other workers. As such there
are 2 types of entries in the shm:

1. events that are to be executed by only a single worker (see the
   `unique` parameter of the `post` method). These entries get a `ttl` in the
   shm and will hence expire.
2. all other events (except local events which do not use the SHM). In these
   cases there is no `ttl` set.

The result of the above is that the SHM will always be full! so that is not a
metric to investigate at.

How to prevent problems:

* the SHM size must at least be a multiple of the maximum payload expected. It
  must be able to cater for all the events that might be send within one
  `interval` (see `configure`).
* `no memory` errors *cannot* be resolved by making the SHM bigger. The only way
  to resolve those is by increasing the `shm_retries` option passed to
  `configure` (which already has a high default).
  This is because the error is due to fragmentation and not a lack of memory.
* the `waiting for event data timed out` error happens if event data gets
  evicted before all the workers got to deal with it. This can happen if
  there is a burst of (large-payload) events. To resolve these:

    * try to avoid big event payloads
    * use a smaller `interval`, so workers check for (and deal with) events
      more frequently (see `interval` option as passed to `configure`)
    * increase the SHM size, such that it can hold all the event data that
      might be send within 1 interval.


Methods
=======

[Back to TOC](#table-of-contents)

configure
---------
`syntax: success, err = events.configure(opts)`

Will initialize the event listener. This should typically be called from the
`init_by_lua` handler, because it will make sure all workers start with the
first event. In case of a reload of the system (starting new and stopping old
workers) past events will not be replayed. And because the order in which
workers reload cannot be guaranteed, also the event start cannot be guaranteed.
So if some sort of state is derived from the events you have to manage that
state separately.

The `opts` parameter is a Lua table with named options:

* `shm`: (required) name of the shared memory to use. Event data will not expire, so
  the module relies on the shm lru mechanism to evict old events from the shm. As such
  the shm should probably not be used for other purposes.
* `shm_retries`: (optional) number of retries when the shm returns "no memory" on posting
  an event, default 999. Each time there is an insertion attempt and no memory is available
  (either no space is available or the memory is available but fragmented), "up to tens"
  of old entries are evicted. After that, if there's still no memory available, the
  "no memory" error is returned. Retrying the insertion triggers the eviction phase
  several times, increasing the memory available as well as the probability of finding a
  large enough contiguous memory block available for the new event data.
* `interval`: (optional) interval to poll for events (in seconds), default 1. Set to 0 to
  disable polling.
* `wait_interval`: (optional) interval between two tries when a new eventid is found, but the
  data is not available yet (due to asynchronous behaviour of the worker processes)
* `wait_max`: (optional) max time to wait for data when event id is found, before discarding
  the event. This is a fail-safe setting in case something went wrong.
* `timeout`: (optional) timeout of unique event data stored in shm (in seconds), default 2.
  See the `unique` parameter of the [post](#post) method.

The return value will be `true`, or `nil` and an error message.

This method can be called repeatedly to update the settings, except for the `shm` value which
cannot be changed after the initial configuration.

NOTE: the `wait_interval` is executed using the `ngx.sleep` function. In contexts where this
function is not available (eg. `init_worker`) it will execute a busy-wait to execute the delay.

[Back to TOC](#table-of-contents)

configured
----------
`syntax: is_already_configured = events.configured()`

The events module runs as a singelton per workerprocess. The `configured`
function allows to check whether it is already up and running.
A check before starting any dependencies is recommended;
```lua
local events = require "resty.worker.events"

local initialization_of_my_module = function()
    assert(events.configured(), "Please configure the 'lua-resty-worker-events' "..
           "module before using my_module")

    -- do initialization here
end
```

[Back to TOC](#table-of-contents)

event_list
----------
`syntax: _M.events = events.event_list(sourcename, event1, event2, ...)`

Utility function to generate event lists and prevent typos in
magic strings. Accessing a non-existing event on the returned table will result
in an 'unknown event error'.
The first parameter `sourcename` is a unique name that identifies the event
source, which will be available as field `_source`. All following parameters
are the named events generated by the event source.

Example usage;
```lua
local ev = require "resty.worker.events"

-- Event source example

local events = ev.event_list(
        "my-module-event-source", -- available as _M.events._source
        "started",                -- available as _M.events.started
        "event2"                  -- available as _M.events.event2
    )

local raise_event = function(event, data)
    return ev.post(events._source, event, data)
end

-- Post my own 'started' event
raise_event(events.started, nil) -- nil for clarity, no eventdata is passed

-- define my module table
local _M = {
  events = events   -- export events table

  -- implementation goes here
}
return _M



-- Event client example;
local mymod = require("some_module")  -- module with an `events` table

-- define a callback and use source modules events table
local my_callback = function(data, event, source, pid)
    if event == mymod.events.started then  -- 'started' is the event name

        -- started event from the resty-worker-events module

    elseif event == mymod.events.stoppping then  -- 'stopping' is the event name

        -- the above will throw an error because of the typo in `stoppping`

    end
end

ev.register(my_callback, mymod.events._source)

```

[Back to TOC](#table-of-contents)

poll
----
`syntax: success, err = events.poll()`

Will poll for new events and handle them all (call the registered callbacks). The implementation is
efficient, it will only check a single shared memory value and return immediately if no new events
are available.

The return value will be `"done"` when it handled all events, `"recursive"` if it was
already in a polling-loop, or `nil + error` if something went wrong.
The `"recursive"` result simply means that an event-handler called `poll` again.

[Back to TOC](#table-of-contents)

post
----
`syntax: success, err = events.post(source, event, data, unique)`

Will post a new event. `source` and `event` are both strings. `data` can be anything (including `nil`)
as long as it is (de)serializable by the cjson module.

If the `unique` parameter is provided then only one worker will execute the event,
the other workers will ignore it. Also any follow up events with the same `unique`
value will be ignored (for the `timeout` period specified to [configure](#configure)).
The process executing the event will not necessarily be the process posting the event.

The return value will be `true` when the event was successfully posted or
`nil + error` in case of failure.

*Note*: the worker process sending the event, will also receive the event! So if
the eventsource will also act upon the event, it should not do so from the event
posting code, but only when receiving it.

[Back to TOC](#table-of-contents)

post_local
----------
`syntax: success, err = events.post_local(source, event, data)`

The same as [post](#post) except that the event will be local to the worker process,
it will not be broadcasted to other workers. With this method, the `data` element
will not be jsonified.

The return value will be `true` when the event was successfully posted or
`nil + error` in case of failure.

[Back to TOC](#table-of-contents)

register
--------
`syntax: events.register(callback, source, event1, event2, ...)`

Will register a callback function to receive events. If `source` and `event` are omitted, then the
callback will be executed on _every_ event, if `source` is provided, then only events with a
matching source will be passed. If (one or more) event name is given, then only when
both `source` and `event` match the callback is invoked.

The callback should have the following signature;

`syntax: callback = function(data, event, source, pid)`

The parameters will be the same as the ones provided to [post](#post), except for the extra value
`pid` which will be the pid of the originating worker process, or `nil` if it was a local event
only. Any return value from `callback` will be discarded.
*Note:* `data` may be a reference type of data (eg. a Lua `table`  type). The same value is passed
to all callbacks, _so do not change the value in your handler, unless you know what you are doing!_

The return value of `register` will be `true`, or it will throw an error if `callback` is not a
function value.

*WARNING*: event handlers must return quickly. If a handler takes more time than
the configured `timeout` value, events will be dropped!

*Note*: to receive the process own `started` event, the handler must be registered before
calling [configure](#configure)

[Back to TOC](#table-of-contents)

register_weak
-------------
`syntax: events.register_weak(callback, source, event1, event2, ...)`

This function is identical to `register`, with the exception that the module
will only hold _weak references_ to the `callback` function.

[Back to TOC](#table-of-contents)

unregister
----------
`syntax: events.unregister(callback, source, event1, event2, ...)`

Will unregister the callback function and prevent it from receiving further events. The parameters
work exactly the same as with [register](#register).

The return value will be `true` if it was removed, `false` if it was not in the handlers list, or
it will throw an error if `callback` is not a function value.

[Back to TOC](#table-of-contents)


Installation
============

Nothing special is required, install like any other pure Lua module. Just make
sure its location is in the module search path.

[Back to TOC](#table-of-contents)


Bugs and Patches
================

Please report bugs or submit patches by creating a ticket on the [GitHub Issue Tracker](http://github.com/Kong/lua-resty-worker-events/issues),

[Back to TOC](#table-of-contents)

Author
======

Thijs Schreijer <thijs@konghq.com>, Kong Inc.

[Back to TOC](#table-of-contents)

Copyright and License
=====================

This module is licensed under the [Apache 2.0 license](https://opensource.org/licenses/Apache-2.0).

Copyright (C) 2016-2020, by Thijs Schreijer, Kong Inc.

All rights reserved.

[Back to TOC](#table-of-contents)


History
=======

### Releasing new versions

- make sure changelog below is up-to-date
- update version number in the code
- create a new rockspec in `./rockspecs`
- commit with message `release x.x.x`
- tag the commit as `x.x.x`
- push commit and tags
- upload to luarocks

### unreleased

- chore: remove redundant type checking of unique_timeout.
- chore: add stacktrace to `post` errors for better debug information.

### 2.0.1, 28-June-2021

- fix: possible deadlock in the `init` phase

### 2.0.0, 16-September-2020

- BREAKING: the `post` function does not call `poll` anymore, making all events
  asynchronous. When an immediate treatment to an event is needed an explicit
  call to `poll` must be done.
- BREAKING: the `post_local` function does not immediately execute the
  event anymore, making all local events asynchronous. When an immediate
  treatment to an event is needed an explicit call to `poll` must be done.
- fix: prevent spinning at 100% CPU when during a reload the event-shm is
  cleared
- fix: improved logging in case of failure to write to shm (add payload size
  for troubleshooting purposes)
- fix: do not log the payload anymore, since it might expose sensitive data
  through the logs
- change: updated `shm_retries` default to 999
- change: changed timer loop to a sleep-loop (performance)
- fix: when re-configuring make sure callbacks table is initialized

### 1.1.0, 23-Dec-2020 (maintenance release)

- feature: the polling loop now runs forever, sleeping for 0.5 seconds between
  runs, avoiding to create new timers on every step.

### 1.0.0, 18-July-2019

- BREAKING: the return values from `poll` (and hence also `post` and `post_local`)
  changed to be more lua-ish, to be truthy when all is well.
- feature: new option `shm_retries` to fix "no memory" errors caused by memory
  fragmentation in the shm when posting events.
- fix: fixed two typos in variable names (edge cases)

### 0.3.3, 8-May-2018

- fix: timeouts in init phases, by removing timeout setting, see issue #9

### 0.3.2, 11-Apr-2018

- change: add a stacktrace to handler errors
- fix: failing error handler if value was non-serializable, see issue #5
- fix: fix a test for the weak handlers


[Back to TOC](#table-of-contents)


See Also
========
* OpenResty: http://openresty.org

[Back to TOC](#table-of-contents)

