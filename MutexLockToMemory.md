# How to use mutex lock during store key/value #

## Introduction ##

  * thread\_mutex\_t cache\_lock; (Lock for cache operations (item**_, assoc_**))
  * tatic pthread\_mutex\_t slabs\_lock = PTHREAD\_MUTEX\_INITIALIZER; (Access to the slab allocator is protected by this lock)


## Details ##

store\_item
```
    pthread_mutex_lock(&cache_lock);     // Lock for cache operations (item_*, assoc_*)
    ret = do_store_item(item, comm, c);
    pthread_mutex_unlock(&cache_lock);
```
-- do\_store\_item
```
       new_it = do_item_alloc(key, it->nkey, flags, old_it->exptime, 
                it->nbytes + old_it->nbytes - 2 /* CRLF */);

       ........

       do_item_link(it);
```
> -- do\_item\_alloc
```
          item *it = slabs_alloc(ntotal, id);
          
```
> > -- slabs\_alloc
```
                 pthread_mutex_lock(&slabs_lock);
                 ret = do_slabs_alloc(size, id);        // alloc memory for item from slab
                 pthread_mutex_unlock(&slabs_lock);
```

> -- do\_item\_link
```
          assoc_insert(it); // map key to 'item *' in hash table
```