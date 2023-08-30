# Asyncio Queue poison pills.

In an asyncio setup like this:

```python
import asyncio
import random
 
async def worker(name, queue: asyncio.Queue):
    while True:
        sleep_for = await queue.get()
        await asyncio.sleep(sleep_for)
        queue.task_done()
        print(f'worker - {name} has slept for {sleep_for:.2f} seconds')
 
 
async def main():
    nr_worker = 3
    queue = asyncio.Queue()
 
    for _ in range(20):
        sleep_for = random.uniform(0.05, 1.0)
        queue.put_nowait(sleep_for)
 
    async with asyncio.TaskGroup() as tg:
        for idx in range(nr_worker):
            tg.create_task(worker(idx, queue))
 
asyncio.run(main())
```

The main process block forever.

But if we put some poisoning pills one for every worker, like this:

```python
import asyncio
import random
 
_poison_pill = object()
 
 
async def worker(name, queue: asyncio.Queue):
    while True:
        sleep_for = await queue.get()
        # break if we are poisoned.
        if sleep_for is _poison_pill:
            break
        await asyncio.sleep(sleep_for)
        queue.task_done()
        print(f'worker - {name} has slept for {sleep_for:.2f} seconds')
 
 
async def main():
    nr_worker = 3
    queue = asyncio.Queue()
 
    for _ in range(20):
        sleep_for = random.uniform(0.05, 1.0)
        queue.put_nowait(sleep_for)
 
    # Put some pills into the queue for every worker.:
    for _ in range(nr_worker):
        queue.put_nowait(_poison_pill)
 
    async with asyncio.TaskGroup() as tg:
        for idx in range(nr_worker):
            tg.create_task(worker(idx, queue))
 
asyncio.run(main())
```

Not blocking anymore.