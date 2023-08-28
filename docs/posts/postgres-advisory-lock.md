# Postgres advisory lock.

A typical deadlock in a Django app.

```python
def test_deadlock_without_transaction_advisory_lock(transactional_db):
    import threading
 
    from django.db import transaction
 
    from myapp.models import Book
 
    lock_id = "foo.bar"
 
    book1 = Book.objects.create(name="Book-1")
    book2 = Book.objects.create(name="Book-2")
 
    caught = None
    event = threading.Event()
 
    def update_1(e, _lock_id):
        nonlocal caught
 
        try:
            with transaction.atomic():
                e.wait()
                Book.objects.filter(id=book1.pk).update(name="Name-1")
                Book.objects.filter(id=book2.pk).update(name="Name-2")
        except Exception as exc:
            caught = exc
 
    def update_2(e, _lock_id):
        nonlocal caught
 
        try:
            with transaction.atomic():
                e.wait()
                Book.objects.filter(id=book2.pk).update(name="Name3")
                Book.objects.filter(id=book1.pk).update(name="Name4")
        except Exception as exc:
            caught = exc
 
    thread_a = threading.Thread(target=update_1, args=(event, lock_id))
    thread_a.start()
    thread_b = threading.Thread(target=update_2, args=(event, lock_id))
    thread_b.start()
 
    event.set()
    thread_a.join()
    thread_b.join()
 
    assert caught is not None
```

Avoid a deadlock with an advisory lock.

```python
from enum import Enum
import contextlib
import zlib
from typing import Optional, Callable



class LockQuery(Enum):
    TRANSACTION_ADVISORY_LOCK_BASE = "SELECT pg_advisory_xact_lock({lock_id})"
    TRANSACTION_ADVISORY_LOCK_TUPLE_FORMAT = (
        "SELECT pg_advisory_xact_lock({lock_id_1}, {lock_id_2})"
    )


    
def generate_crc_32_lock_id(lock_id: str) -> int:
    return zlib.crc32(lock_id.encode("utf-8"))


@contextlib.contextmanager
def transaction_advisory_lock(
    lock_id: str,
    using: Optional[str] = None,
    key_generator: Optional[Callable[[str], int]] = generate_crc_32_lock_id,
):
    """
    Obtains an exclusive transaction-level advisory lock.
    """
    using = using or DEFAULT_DB_ALIAS

    lock_id = key_generator(lock_id)

    match lock_id:
        case int():
            query = LockQuery.TRANSACTION_ADVISORY_LOCK_BASE.value
            command = query.format(lock_id=lock_id)
        case [int() as l1, int() as l2]:
            query = LockQuery.TRANSACTION_ADVISORY_LOCK_TUPLE_FORMAT.value
            command = query.format(lock_id_1=l1, lock_id_2=l2)
        case _:
            raise ValueError("Unsupported param format.")

    connection = connections[using]
    if not connection.in_atomic_block:
        raise RuntimeError("Must be inside a transaction.")

    cursor = connection.cursor()
    cursor.execute(command)
    try:
        yield
    finally:
        cursor.close()

```
