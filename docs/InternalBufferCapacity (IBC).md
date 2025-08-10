**Default value:** The initial capacity of a dynamic buffer is defined by the type that the buffer stores. By default, the capacity defaults to the number of elements that fit within 128 bytes.

- **If you exceed IBC:** Unity allocates **external memory** for the buffer (on the heap), which:
    - Loses cache locality.
    - Adds GC-like memory allocation overhead (though DOTS manages it in native memory).
    - Can hurt performance for frequent access.

- **If you set IBC too high:**
    - Every entity reserves that many elements worth of space **inside the chunk**, even if you store fewer items.
    - Wastes chunk memory, reducing how many entities fit per chunk and hurting overall performance.

- **Rule of thumb:**
    - Small, frequently accessed buffers → keep IBC high enough so they almost never spill.
    - Large or rarely accessed buffers → keep IBC small to avoid wasting chunk space.