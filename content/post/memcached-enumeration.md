---
title: "Memcached Enumeration"
date: 2019-02-04T20:43:23+01:00
draft: false
index: true
tags: ["cache", "linux"]
categories: ["pentest", "Enumeration"]
comments: true
highlight: true
---


 A server cache is an information technology for the temporary storage of data, to reduce server lag. I find a lot of those technologies in my daily work while doing penetration testing. Memcached is one of them and I'd like to talk about it and how to extract informations from it.

<!--more-->

### TL;DR
[Memcached](https://memcached.org/) is a distributed memory object caching system, is an in-memory key-value store for small chunks of arbitrary data (strings, objects) from results of database calls, API calls, or page rendering.

### Security

![alt text](/img/memcached-enumeration/memchached-tcp-port.png "Memcached TCP Port")

Memcached expose TCP port **11211** and bind it on localhost, however it's still possible to communicate with this port via SSRF or literary movement techniques.

### Protocol
Clients of memcached communicate with server through TCP connections. A simple raw commands can be performed to do various things with memcached.

![alt text](/img/memcached-enumeration/memcached-cheat-sheet.png "Memcached Cheat Sheet")

More Commands can be found [here.](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)

### Memcached Extractor
Now let's see how to extracts the slabs from a memcached instance and then finds the keys and values stored in those slabs.</br>
First, let's connect to the server using `netcat`

> `nc 127.0.0.1 11211`

After successfully connected, let's print memory statistics with :

> `stats slabs`

```text
STAT 16:chunk_size 2904
STAT 16:chunks_per_page 361
STAT 16:total_pages 1
STAT 16:total_chunks 361
STAT 16:used_chunks 0
STAT 16:free_chunks 361
STAT 16:free_chunks_end 0
STAT 16:mem_requested 0
STAT 16:get_hits 14
STAT 16:cmd_set 7
STAT 16:delete_hits 0
STAT 16:incr_hits 0
STAT 16:decr_hits 0
STAT 16:cas_hits 0
STAT 16:cas_badval 0
STAT 16:touch_hits 0
STAT 26:chunk_size 27120
STAT 26:chunks_per_page 38
STAT 26:total_pages 1
STAT 26:total_chunks 38
STAT 26:used_chunks 0
STAT 26:free_chunks 38
STAT 26:free_chunks_end 0
STAT 26:mem_requested 0
STAT 26:get_hits 7046
STAT 26:cmd_set 33
STAT 26:delete_hits 0
STAT 26:incr_hits 0
STAT 26:decr_hits 0
STAT 26:cas_hits 0
STAT 26:cas_badval 0
STAT 26:touch_hits 0
STAT active_slabs 2
STAT total_malloced 2078904
END
```

If you notice in our example we have two different values **16** and **26**. We will using those values to fetch for the key's names associated with them.

> `stats cachedump 16 0`

```text
ITEM stock [2807 b; 1549317135 s]
END
```

> `stats cachedump 26 0`

```text
ITEM users [24625 b; 1549317140 s]
END
```

As you can see now we have the key's names which are `stock` and `users`, now let's extract all the data associated with those keys with the command :

> `get stock`

```text
VALUE stock 0 2807
{"1": {"product": "Apples - Sliced / Wedge", "qty": 568}, "2": {"product": "Appetizer - Tarragon Chicken", "qty": 16}}
```

#### Sources: ####
* [lzone.de](https://lzone.de//cheat-sheet/memcached)
* [MSF](https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/gather/memcached_extractor.rb)
</br>
