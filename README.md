# EventLoop Component

[![Build Status](https://travis-ci.org/reactphp/event-loop.svg?branch=master)](https://travis-ci.org/reactphp/event-loop)
[![Code Climate](https://codeclimate.com/github/reactphp/event-loop/badges/gpa.svg)](https://codeclimate.com/github/reactphp/event-loop)

Event loop abstraction layer that libraries can use for evented I/O.

In order for async based libraries to be interoperable, they need to use the
same event loop. This component provides a common `LoopInterface` that any
library can target. This allows them to be used in the same loop, with one
single `run()` call that is controlled by the user.

> The master branch contains the code for the upcoming 0.5 release.
For the code of the current stable 0.4.x release, checkout the
[0.4 branch](https://github.com/reactphp/event-loop/tree/0.4).

**Table of Contents**

* [Quickstart example](#quickstart-example)
* [Usage](#usage)
* [Loop implementations](#loop-implementations)
* [Install](#install)
* [Tests](#tests)
* [License](#license)

## Quickstart example

Here is an async HTTP server built with just the event loop.

```php
$loop = React\EventLoop\Factory::create();

$server = stream_socket_server('tcp://127.0.0.1:8080');
stream_set_blocking($server, 0);

$loop->addReadStream($server, function ($server) use ($loop) {
    $conn = stream_socket_accept($server);
    $data = "HTTP/1.1 200 OK\r\nContent-Length: 3\r\n\r\nHi\n";
    $loop->addWriteStream($conn, function ($conn) use (&$data, $loop) {
        $written = fwrite($conn, $data);
        if ($written === strlen($data)) {
            fclose($conn);
            $loop->removeStream($conn);
        } else {
            $data = substr($data, $written);
        }
    });
});

$loop->addPeriodicTimer(5, function () {
    $memory = memory_get_usage() / 1024;
    $formatted = number_format($memory, 3).'K';
    echo "Current memory usage: {$formatted}\n";
});

$loop->run();
```

See also the [examples](examples).

## Usage

Typical applications use a single event loop which is created at the beginning
and run at the end of the program.

```php
// [1]
$loop = React\EventLoop\Factory::create();

// [2]
$loop->addPeriodicTimer(1, function () {
    echo "Tick\n";
});

$stream = new React\Stream\ReadableResourceStream(
    fopen('file.txt', 'r'),
    $loop
);

// [3]
$loop->run();
```

1. The loop instance is created at the beginning of the program. A convenience
   factory `React\EventLoop\Factory::create()` is provided by this library which
   picks the best available [loop implementation](#loop-implementations).
2. The loop instance is used directly or passed to library and application code.
   In this example, a periodic timer is registered with the event loop which
   simply outputs `Tick` every second and a
   [readable stream](https://github.com/reactphp/stream#readableresourcestream)
   is created by using ReactPHP's
   [stream component](https://github.com/reactphp/stream) for demonstration
   purposes.
3. The loop is run with a single `$loop->run()` call at the end of the program.

## Loop implementations

In addition to the interface there are the following implementations provided:

* `StreamSelectLoop`: This is the only implementation which works out of the
  box with PHP. It does a simple `select` system call. It's not the most
  performant of loops, but still does the job quite well.

* `LibEventLoop`: This uses the `libevent` pecl extension. `libevent` itself
  supports a number of system-specific backends (epoll, kqueue).

* `LibEvLoop`: This uses the `libev` pecl extension
  ([github](https://github.com/m4rw3r/php-libev)). It supports the same
  backends as libevent.

* `ExtEventLoop`: This uses the `event` pecl extension. It supports the same
  backends as libevent.

All of the loops support these features:

* File descriptor polling
* One-off timers
* Periodic timers
* Deferred execution on future loop tick

### addTimer()

The `addTimer(float $interval, callable $callback): TimerInterface` method can be used to
enqueue a callback to be invoked once after the given interval.

The timer callback function MUST be able to accept a single parameter,
the timer instance as also returned by this method or you MAY use a
function which has no parameters at all.

The timer callback function MUST NOT throw an `Exception`.
The return value of the timer callback function will be ignored and has
no effect, so for performance reasons you're recommended to not return
any excessive data structures.

Unlike [`addPeriodicTimer()`](#addperiodictimer), this method will ensure
the callback will be invoked only once after the given interval.
You can invoke [`cancelTimer`](#canceltimer) to cancel a pending timer.

```php
$loop->addTimer(0.8, function () {
    echo 'world!' . PHP_EOL;
});

$loop->addTimer(0.3, function () {
    echo 'hello ';
});
```

See also [example #1](examples).

If you want to access any variables within your callback function, you
can bind arbitrary data to a callback closure like this:

```php
function hello(LoopInterface $loop, $name)
{
    $loop->addTimer(1.0, function () use ($name) {
        echo "hello $name\n";
    });
}

hello('Tester');
```

The execution order of timers scheduled to execute at the same time is
not guaranteed.

### addPeriodicTimer()

The `addPeriodicTimer(float $interval, callable $callback): TimerInterface` method can be used to
enqueue a callback to be invoked repeatedly after the given interval.

The timer callback function MUST be able to accept a single parameter,
the timer instance as also returned by this method or you MAY use a
function which has no parameters at all.

The timer callback function MUST NOT throw an `Exception`.
The return value of the timer callback function will be ignored and has
no effect, so for performance reasons you're recommended to not return
any excessive data structures.

Unlike [`addTimer()`](#addtimer), this method will ensure the the
callback will be invoked infinitely after the given interval or until you
invoke [`cancelTimer`](#canceltimer).

```php
$timer = $loop->addPeriodicTimer(0.1, function () {
    echo 'tick!' . PHP_EOL;
});

$loop->addTimer(1.0, function () use ($loop, $timer) {
    $loop->cancelTimer($timer);
    echo 'Done' . PHP_EOL;
});
```

See also [example #2](examples).

If you want to limit the number of executions, you can bind
arbitrary data to a callback closure like this:

```php
function hello(LoopInterface $loop, $name)
{
    $n = 3;
    $loop->addPeriodicTimer(1.0, function ($timer) use ($name, $loop, &$n) {
        if ($n > 0) {
            --$n;
            echo "hello $name\n";
        } else {
            $loop->cancelTimer($timer);
        }
    });
}

hello('Tester');
```

The execution order of timers scheduled to execute at the same time is
not guaranteed.

### cancelTimer()

The `cancelTimer(TimerInterface $timer): void` method can be used to
cancel a pending timer.

See also [`addPeriodicTimer()`](#addperiodictimer) and [example #2](examples).

You can use the [`isTimerActive()`](#istimeractive) method to check if
this timer is still "active". After a timer is successfully canceled,
it is no longer considered "active".

Calling this method on a timer instance that has not been added to this
loop instance or on a timer

### isTimerActive()

The `isTimerActive(TimerInterface $timer): bool` method can be used to
check if a given timer is active.

A timer is considered "active" if it has been added to this loop instance
via [`addTimer()`](#addtimer) or [`addPeriodicTimer()`](#addperiodictimer)
and has not been canceled via [`cancelTimer()`](#canceltimer) and is not
a non-periodic timer that has already been triggered after its interval.

### futureTick()

The `futureTick(callable $listener): void` method can be used to
schedule a callback to be invoked on a future tick of the event loop.

This works very much similar to timers with an interval of zero seconds,
but does not require the overhead of scheduling a timer queue.

The tick callback function MUST be able to accept zero parameters.

The tick callback function MUST NOT throw an `Exception`.
The return value of the tick callback function will be ignored and has
no effect, so for performance reasons you're recommended to not return
any excessive data structures.

If you want to access any variables within your callback function, you
can bind arbitrary data to a callback closure like this:

```php
function hello(LoopInterface $loop, $name)
{
    $loop->futureTick(function () use ($name) {
        echo "hello $name\n";
    });
}

hello('Tester');
```

Unlike timers, tick callbacks are guaranteed to be executed in the order
they are enqueued.
Also, once a callback is enqueued, there's no way to cancel this operation.

This is often used to break down bigger tasks into smaller steps (a form
of cooperative multitasking).

```php
$loop->futureTick(function () {
    echo 'b';
});
$loop->futureTick(function () {
    echo 'c';
});
echo 'a';
```

See also [example #3](examples).

## Install

The recommended way to install this library is [through Composer](http://getcomposer.org).
[New to Composer?](http://getcomposer.org/doc/00-intro.md)

This will install the latest supported version:

```bash
$ composer require react/event-loop
```

## Tests

To run the test suite, you first need to clone this repo and then install all
dependencies [through Composer](http://getcomposer.org):

```bash
$ composer install
```

To run the test suite, go to the project root and run:

```bash
$ php vendor/bin/phpunit
```

## License

MIT, see [LICENSE file](LICENSE).
