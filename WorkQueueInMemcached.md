# Memcached use workqueue to work together with libevent #

## Introduction ##

In mem$d, the main pthread(master) need to work together with worker pthreads(slave) and the worker pthreads will use libevent to manage the fds, but the interfaces are not MT safe in libevent, so mem$d use work queue to assign works to these worker pthreads.

## Details ##

  * The data structure and interfaces which associate with CQ in "threads.c".

```
       /* An item in the connection queue. */
       typedef struct conn_queue_item CQ_ITEM;
       struct conn_queue_item {
           int               sfd;
           enum conn_states  init_state;
           int               event_flags;
           int               read_buffer_size;
           enum network_transport     transport;
           CQ_ITEM          *next;
       };
```

```
       /* A connection queue. */
       typedef struct conn_queue CQ;
       struct conn_queue {
           CQ_ITEM *head;
           CQ_ITEM *tail;
           pthread_mutex_t lock;
           pthread_cond_t  cond;
       };
```

```
       /* Free list of CQ_ITEM structs */
       static CQ_ITEM *cqi_freelist;
       static pthread_mutex_t cqi_freelist_lock;
```


```
       /*
        * Initializes a connection queue.
        */
       static void cq_init(CQ *cq);

       /*
        * Looks for an item on a connection queue, but doesn't block if there isn't
        * one.
        * Returns the item, or NULL if no item is available
        */
       static CQ_ITEM *cq_pop(CQ *cq);
       
       /*
        * Adds an item to a connection queue.
        */
       static void cq_push(CQ *cq, CQ_ITEM *item);

       /*
        * Returns a fresh connection queue item.
        * Get it from freelist
        */
       static CQ_ITEM *cqi_new(void)

       /*
        * Frees a connection queue item (adds it to the freelist.)
        */
       static void cqi_free(CQ_ITEM *item);
       
```

  * The main thread use this interface to assign work to worker pthreads in "threads.c".

```
      /*
       * Dispatches a new connection to another thread. This is only ever called
       * from the main thread, either during initialization (for UDP) or because
       * of an incoming connection.
       */
      void dispatch_conn_new(int sfd, enum conn_states init_state, int event_flags,
                             int read_buffer_size, enum network_transport transport) {
          CQ_ITEM *item = cqi_new();
          int tid = (last_thread + 1) % settings.num_threads;

          LIBEVENT_THREAD *thread = threads + tid;

          last_thread = tid;

          item->sfd = sfd;
          item->init_state = init_state;
          item->event_flags = event_flags;
          item->read_buffer_size = read_buffer_size;
          item->transport = transport;

          cq_push(thread->new_conn_queue, item);

          MEMCACHED_CONN_DISPATCH(sfd, thread->thread_id);
          if (write(thread->notify_send_fd, "", 1) != 1) {
              perror("Writing to thread notify pipe");
          }
      }
```