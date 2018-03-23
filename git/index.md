# Git 

## 概览

```text

+--------------+       +-------------------+       +-----------------------------+
| working tree | ----> | index/stage/cache | ----> | append-only object database |
+--------------+       +-------------------+       +-----------------------------+
                                                                |
                                                                |
                          +-------------------------------------+                                                        
                          |
                          |
      +--------------+----+----------+----------------+
      |              |               |                |
      |              |               |                |
      v              v               v                v
  +------+        +------+        +--------+        +-----+
  | blob |        | tree |        | commit |        | tag |
  +------+        +------+        +--------+        +-----+
     |               |                 |                |
     |               |                 |                +---> contains a reference to another object and 
     |               |                 |                      can hold added meta-data related to another object
     |               |                 |
     |               |                 +---> links tree objects together into a history
     |               |
     |               +---> the equivalent of a directory
     |
     +---> the content of a file
```

## References

- [Git](https://en.wikipedia.org/wiki/Git)
- [Content-addressable storage](https://en.wikipedia.org/wiki/Content-addressable_storage)
