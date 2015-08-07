这里主要通过图形+代码的方式展示mem$d是如何设计hash的，预计填充时间是：2010/9/28 ~ 2010/9/30

  * item structure
```
/**
 * Structure for storing items within memcached.
 */
typedef struct _stritem {
    struct _stritem *next;
    struct _stritem *prev;
    struct _stritem *h_next;    /* hash chain next */
    rel_time_t      time;       /* least recent access */
    rel_time_t      exptime;    /* expire time */
    int             nbytes;     /* size of data */
    unsigned short  refcount;
    uint8_t         nsuffix;    /* length of flags-and-length string */
    uint8_t         it_flags;   /* ITEM_* above */
    uint8_t         slabs_clsid;/* which slab class we're in */
    uint8_t         nkey;       /* key length, w/terminating null and padding */
    void * end[];
    /* if it_flags & ITEM_CAS we have 8 bytes CAS */
    /* then null-terminated key */
    /* then " flags length\r\n" (no terminating null) */
    /* then data with terminating \r\n (no terminating null; it's binary!) */
} item;
```

  * alloc memory for primary hash table, its size is hashsize(hashpower)
```
/* how many powers of 2's worth of buckets we use */
static unsigned int hashpower = 16;

#define hashsize(n) ((ub4)1<<(n))
#define hashmask(n) (hashsize(n)-1)

// alloc memory for primary hash table
void assoc_init(void) {
    primary_hashtable = calloc(hashsize(hashpower), sizeof(void *)); //  2^16 = 65536
    if (! primary_hashtable) {
        fprintf(stderr, "Failed to init hashtable.\n");
        exit(EXIT_FAILURE);
    }
}
```

  * insert the hashed key into hashtable, it maybe insert into old hashtable, maybe insert into new primary hashtable.
```
// find the item according to key
item *assoc_find(const char *key, const size_t nkey) {
    uint32_t hv = hash(key, nkey, 0);
    item *it;
    unsigned int oldbucket;

    if (expanding &&
        (oldbucket = (hv & hashmask(hashpower - 1))) >= expand_bucket)
    {
        it = old_hashtable[oldbucket]; // moving bucket from oldbucket to primarybucket now
    } else {
        it = primary_hashtable[hv & hashmask(hashpower)];
    }

    item *ret = NULL;
    int depth = 0;
    while (it) {
        if ((nkey == it->nkey) && (memcmp(key, ITEM_key(it), nkey) == 0)) {
            ret = it;
            break;
        }
        it = it->h_next;
        ++depth;
    }
    MEMCACHED_ASSOC_FIND(key, nkey, depth);
    return ret;
}

/* grows the hashtable to the next power of 2. */
static void assoc_expand(void) {
    old_hashtable = primary_hashtable; // expanding the primary hash table now

    primary_hashtable = calloc(hashsize(hashpower + 1), sizeof(void *));
    if (primary_hashtable) {
        if (settings.verbose > 1)
            fprintf(stderr, "Hash table expansion starting\n");
        hashpower++;
        expanding = true;
        expand_bucket = 0;
        pthread_cond_signal(&maintenance_cond);
    } else {
        primary_hashtable = old_hashtable;
        /* Bad news, but we can keep running. */
    }
}

/* Note: this isn't an assoc_update.  The key must not already exist to call this */
int assoc_insert(item *it) {
    uint32_t hv;
    unsigned int oldbucket;

    assert(assoc_find(ITEM_key(it), it->nkey) == 0);  /* shouldn't have duplicately named things defined */

    hv = hash(ITEM_key(it), it->nkey, 0);
    if (expanding &&
        (oldbucket = (hv & hashmask(hashpower - 1))) >= expand_bucket)
    {
        it->h_next = old_hashtable[oldbucket]; // h_next used the link list in bucket array
        old_hashtable[oldbucket] = it; // insert to the head
    } else {
        it->h_next = primary_hashtable[hv & hashmask(hashpower)];
        primary_hashtable[hv & hashmask(hashpower)] = it;
    }

    hash_items++;
    if (! expanding && hash_items > (hashsize(hashpower) * 3) / 2) {
        assoc_expand();
    }

    MEMCACHED_ASSOC_INSERT(ITEM_key(it), it->nkey, hash_items);
    return 1;
}
```

  * delete the hashed key from hashtable
```
/* returns the address of the item pointer before the key.  if *item == 0,
   the item wasn't found */

static item** _hashitem_before (const char *key, const size_t nkey) {
    uint32_t hv = hash(key, nkey, 0);
    item **pos;
    unsigned int oldbucket;

    if (expanding &&
        (oldbucket = (hv & hashmask(hashpower - 1))) >= expand_bucket)
    {
        pos = &old_hashtable[oldbucket];
    } else {
        pos = &primary_hashtable[hv & hashmask(hashpower)];
    }

    while (*pos && ((nkey != (*pos)->nkey) || memcmp(key, ITEM_key(*pos), nkey))) { // key not same
        pos = &(*pos)->h_next;
    }
    return pos;
}

void assoc_delete(const char *key, const size_t nkey) {
    item **before = _hashitem_before(key, nkey);

    if (*before) {
        item *nxt;
        hash_items--;
        /* The DTrace probe cannot be triggered as the last instruction
         * due to possible tail-optimization by the compiler
         */
        MEMCACHED_ASSOC_DELETE(key, nkey, hash_items);
        nxt = (*before)->h_next;
        (*before)->h_next = 0;   /* probably pointless, but whatever. */
        *before = nxt;
        return;
    }
    /* Note:  we never actually get here.  the callers don't delete things
       they can't find. */
    assert(*before != 0);
}
```
![http://memcached-shawn.googlecode.com/files/deleteitem.jpg](http://memcached-shawn.googlecode.com/files/deleteitem.jpg)

  * Start the assoc maintenance thread which move bucket(rehash) from old primary hashtable(old hashtable to new primary hashtable), it will do rehash in batch mode, the number of buckets was decided by hash\_bulk\_move. When we want the thread to move buckets, using pthread\_cond\_signal to tell her.

```
static volatile int do_run_maintenance_thread = 1;

#define DEFAULT_HASH_BULK_MOVE 1
int hash_bulk_move = DEFAULT_HASH_BULK_MOVE;

/* current thread used to maintain the hash table, when need expand the hash table
 * we have to rehash the item using the new hash table size (hashsize(hashpower))
 */
static void *assoc_maintenance_thread(void *arg) {

    while (do_run_maintenance_thread) {
        int ii = 0;

        /* Lock the cache, and bulk move multiple buckets to the new
         * hash table. */
        pthread_mutex_lock(&cache_lock);

        for (ii = 0; ii < hash_bulk_move && expanding; ++ii) {
            item *it, *next;
            int bucket;

            // move the element in current operated bucket (expand_bucket)
            for (it = old_hashtable[expand_bucket]; NULL != it; it = next) {
                next = it->h_next; // hash table next pointer

                // calculate the new bucket entry and insert from head
                bucket = hash(ITEM_key(it), it->nkey, 0) & hashmask(hashpower);
                it->h_next = primary_hashtable[bucket];
                primary_hashtable[bucket] = it;
            }

            old_hashtable[expand_bucket] = NULL;

            expand_bucket++;
            if (expand_bucket == hashsize(hashpower - 1)) {
                expanding = false;
                free(old_hashtable);
                if (settings.verbose > 1)
                    fprintf(stderr, "Hash table expansion done\n");
            }
        }

        if (!expanding) {
            /* We are done expanding.. just wait for next invocation */
            pthread_cond_wait(&maintenance_cond, &cache_lock);
        }

        pthread_mutex_unlock(&cache_lock);
    }
    return NULL;
}

static pthread_t maintenance_tid;

int start_assoc_maintenance_thread() {
    int ret;
    char *env = getenv("MEMCACHED_HASH_BULK_MOVE");
    if (env != NULL) {
        hash_bulk_move = atoi(env);
        if (hash_bulk_move == 0) {
            hash_bulk_move = DEFAULT_HASH_BULK_MOVE;
        }
    }
    if ((ret = pthread_create(&maintenance_tid, NULL,
                              assoc_maintenance_thread, NULL)) != 0) {
        fprintf(stderr, "Can't create thread: %s\n", strerror(ret));
        return -1;
    }
    return 0;
}
```

  * Stop assoc maintenance thread
```
void stop_assoc_maintenance_thread() {
    pthread_mutex_lock(&cache_lock);
    do_run_maintenance_thread = 0;
    pthread_cond_signal(&maintenance_cond);
    pthread_mutex_unlock(&cache_lock);

    /* Wait for the maintenance thread to stop */
    pthread_join(maintenance_tid, NULL);
}
```

![http://memcached-shawn.googlecode.com/files/hashtable.jpg](http://memcached-shawn.googlecode.com/files/hashtable.jpg)