# Context Manager with iterator.

Somewhere in the web I found (cannot remember where!!) a nice pattern that combine an Iterator with a context-manager.

```python
class RetryError(Exception):
    pass


class SwallowException:
    def __init__(self, notify_success, max_retry):
        self._notify_success = notify_success
        self._max_retry = max_retry

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_val:
            print(f"Got an exception: {exc_type} - {exc_val if exc_val else ''}")
        else:
            self._notify_success()
        return True


class Retry:
    def __init__(self, max_retry):
        self._max_retry = max_retry
        self._success = False

    def __iter__(self):
        for i in range(self._max_retry):
            yield SwallowException(self.succeed, i)
            if self._success is True:
                print("Finished success..")
                break
        else:
            raise RetryError("too many attempts")

    def succeed(self):
        self._success = True


for attempt in Retry(3):
    with attempt:
        1 / 0

```

This combine an iterator that yield a context manager.
The context manager catch all exceptions and if no exceptions are thrown, notify the Iterator.
