## Thread-safe

**Property of code that can be safely executed by multiple threads simultaneously** without causing race conditions or data corruption.

In Unity DOTS, [[Job]] system ensures thread safety through:
- **[[Job Safety System]]** - validates no conflicting read/write access
- **[ReadOnly] attribute** - marks data as read-only for parallel access
- **Job dependencies** - ensures jobs with conflicting access run sequentially

**Thread-safe operations:**
- Multiple jobs reading same data (`[ReadOnly]`)
- Jobs accessing different data in parallel
- Properly dependency-chained jobs

**Not thread-safe:**
- Multiple jobs writing to same data
- Reading while another job writes
- Without proper job dependencies

**See also:** [[Job Safety System]], [[ReadOnly and Optional|[ReadOnly]]], [[Job]]
