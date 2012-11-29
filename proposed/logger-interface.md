Logger Interface
================

This document describes a common interface for logging libraries.

The main goal is to allow libraries to receive a `Psr\Log\LoggerInterface`
object and write logs to it in a simple and universal way. Frameworks
and CMSs that have custom needs MAY extend the interface for their own
purpose, but SHOULD remain compatible with this document. This ensures
that the third-party libraries an application uses can write to the
centralized application logs.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119][].

[RFC 2119]: http://tools.ietf.org/html/rfc2119

1. Specification
-----------------

- The interface exposes eight methods to write logs to the eight [RFC 5424][]
  levels (debug, info, notice, warning, error, critical, alert, emergency).

- A ninth method, `log`, accepts a log level as first argument. Calling this
  method with one of the log level constants MUST have the same result as
  calling the level-specific method. Calling this method with a level not
  defined by this specification MUST throw an `InvalidArgumentException` if
  the implementation does not know about the level. Users SHOULD NOT use a
  custom level without knowing for sure the current implementation supports it.

- Every method accepts a string as the message, or an object with a
  `__toString()` method. Implementors MAY have special handling for the passed
  objects. If that is not the case, implementors MUST cast it to a string.

- Every method accepts an array as context data. This is meant to hold any
  extraneous information that does not fit well in a string. The array can
  contain anything. Implementors MUST ensure they treat context data with
  as much lenience as possible. A given value in the context MUST NOT throw
  an exception or trigger a php `E_ERROR`, `E_WARNING`, ...

- If an `Exception` object is passed in the context data, it MUST be in the
  `'exception'` key. Logging exceptions is a common pattern and this allows
  implementors to extract a stack trace from the exception when the log
  backend supports it. Implementors MUST still verify that the `'exception'`
  key is actually an `Exception` before using it as such, as it MAY contain
  anything.

- The message MAY contain placeholders in `%variable%` format (e.g. `%foo%`).
  Those placeholders MAY be replaced by their corresponding value in the
  context array. See for example:

  ```php
  $logger = new Logger();
  $username = 'Bob';

  // MAY produce "User Bob created" or store
  // "User %username% created" + a context array for later use
  $logger->log('User %username% created', array('username' => $username));
  ```

  Implementors MAY use placeholders to implement various escaping strategies
  and translate logs for display. As such their use is very much encouraged.
  Users SHOULD NOT pre-escape values however since they can not know in which
  context the data will be displayed.

  There is no support for deep replacement of placeholders. Placeholder names
  match directly to the keys of the context array. Here is a sample
  implementation to make it clear:

  ```php
  $message = 'User %username% created';
  $context = array('username' => $username);

  foreach ($context as $key => $val) {
      $message = str_replace('%'.$key.'%', $val, $message);
  }
  ```

- A `Psr\Log\NullLogger` is provided together with the interface. It MAY be
  used by users of the interface to provide a fall-back "black hole"
  implementation if no logger is given to them. However conditional logging
  may be a better approach if context data creation is expensive.

[RFC 5424]: http://tools.ietf.org/html/rfc5424

2. `Psr\Log\LoggerInterface`
----------------------------

```php
<?php

namespace Psr\Log;

/**
 * Describes a logger instance
 *
 * The message MUST be a string or object implementing __toString().
 *
 * The message MAY contain placeholders in the form: %foo% where foo
 * will be replaced by the context data in key "foo".
 *
 * The context array can contain arbitrary data, the only assumption that
 * can be made by implementors is that if an Exception instance is given
 * to produce a stack trace, it MUST be in a key named "exception".
 *
 * See https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md
 * for the full interface specification.
 */
interface LoggerInterface
{
    const EMERGENCY = 'emergency';
    const ALERT = 'alert';
    const CRITICAL = 'critical';
    const ERROR = 'error';
    const WARNING = 'warning';
    const NOTICE = 'notice';
    const INFO = 'info';
    const DEBUG = 'debug';

    /**
     * System is unusable.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function emergency($message, array $context = array());

    /**
     * Action must be taken immediately.
     *
     * Example: Entire website down, database unavailable, etc. This should
     * trigger the SMS alerts and wake you up.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function alert($message, array $context = array());

    /**
     * Critical conditions.
     *
     * Example: Application component unavailable, unexpected exception.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function critical($message, array $context = array());

    /**
     * Runtime errors that do not require immediate action but should typically
     * be logged and monitored.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function error($message, array $context = array());

    /**
     * Exceptional occurrences that are not errors.
     *
     * Example: Use of deprecated APIs, poor use of an API, undesirable things
     * that are not necessarily wrong.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function warning($message, array $context = array());

    /**
     * Normal but significant events.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function notice($message, array $context = array());

    /**
     * Interesting events.
     *
     * Example: User logs in, SQL logs.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function info($message, array $context = array());

    /**
     * Detailed debug information.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function debug($message, array $context = array());

    /**
     * Logs with an arbitrary level.
     *
     * @param mixed $level
     * @param string $message
     * @param array $context
     * @return null
     */
    public function log($level, $message, array $context = array());
}
```

3. `Psr\Log\NullLogger`
-----------------------

```php
<?php

namespace Psr\Log;

/**
 * This Logger can be used to avoid conditional log calls
 *
 * Logging should always be optional, and if no logger is provided to your
 * library creating a NullLogger instance to have something to throw logs at
 * is a good way to avoid littering your code with `if ($this->logger) { }`
 * blocks.
 */
class NullLogger implements LoggerInterface
{
    /**
     * {@inheritDoc}
     */
    public function emergency($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function alert($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function critical($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function error($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function warning($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function notice($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function info($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function debug($message, array $context = array())
    {
        // noop
    }

    /**
     * {@inheritDoc}
     */
    public function log($level, $message, array $context = array())
    {
        // noop
    }
}
```
