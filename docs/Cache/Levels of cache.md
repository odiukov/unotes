### Quick breakdown
##### Modern CPUs have multiple levels of cache

- **L1 cache** → fastest (~1–4 CPU cycles)
    
- **L2 cache** → still fast (~4–12 cycles)
    
- **L3 cache** → slower (~12–50 cycles)
    
- **RAM** → _much_ slower (~100–300 cycles)
##### When you access a variable:

1. CPU checks if it’s already in **L1 cache** (fastest).
       
2. If not, it checks **L2**, then **L3**.
       
3. If it’s nowhere in cache → [[Cache miss]] → fetch from RAM (slow).