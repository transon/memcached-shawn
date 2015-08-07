## Data Structure ##
```
lock  // less complex lock
wl    // write counter(0/1)
rl    // read counter (0+)
waitQ // waiting for the lock
holdQ // holding the lock
```

## Interfaces ##

fthread lock was designed based on less complex lock (eg: spinlock), so fthread lock is a more complex r/w lock.


---


  * Request the lock for write

> If there hasn't threads holding the lock for read & write, in the other words, the counter of write lock and read lock are zero, then it can get the lock for write, assign wl = 1, push the thread into holdQ; but if there has threads holding the lock for read/write, so it can't hold the lock, need push it into waitQ, then waiting for other thread resume it.


  * Request the lock for read

> If there hasn't thread holding the write lock, the increment the rl and push it into holdQ, but if there has thread holding the write lock, it have to be push into waitQ, waiting for other thread resume it.

  * Free the lock

> If the free lock is used to write, then assign wl = 0, but if the free lock is used to read, then decrement the read counter; start checking the waitQ, to check if any thread waiting for the lock for read/write, the following steps just likes request the lock for write or read (resume more thread for read).
