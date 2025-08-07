CLHT
====

CLHT is a very fast and scalable concurrent, resizable hash table. It comes in two main variants: a lock-based version (CLHT-LB) and a lock-free version (CLHT-LF).

**Upsert Support in This Fork**
The original CLHT only supported *insert-only* semantics: calling `put(key, val)` on an existing key would fail. This fork adds true **upsert** (insert-or-overwrite) capabilities:

* **Lock-based variant**: bucket-level mutexes now guard a two-phase remove+put or direct overwrite of `val[j]` under lock, ensuring atomic replace without exposing intermediate state.
* **Lock-free variant**: before claiming an empty slot via CAS on the bucket `snapshot`, we scan for an existing valid slot and write the new `val` in place (with write fences). Readers continue to see either the old or new value, preserving lock-free consistency.

Original CLHT Design
--------------------

1. **Cache-line buckets**: update operations (`put`, `remove`) on a bucket complete with at most one cache-line transfer.
2. **Snapshot-based LF**: an 8-byte `snapshot_t` (version + map) coordinates wait-free `get` operations and atomic updates via CAS.
3. **Lock-based LB**: per-bucket locks serialize updates, with in-place writes or bucket chaining/resizing on overflow.

The performance capabilities have been maintained in this fork. For detailed benchmark results, please refer to the original work.

## Original Authors & License

* **Author**: Vasileios Trigonakis ([github@trigonakis.com](mailto:github@trigonakis.com))
* **Publication**: *Asynchronized Concurrency: The Secret to Scaling Concurrent Search Data Structures*, David, Guerraoui, Trigonakis (ASPLOS '15)
* **License**: MIT

Compilation
-----------

You can install the dependencies of CLHT with `make dependencies`.

CLHT requires the ssmem memory allocator (https://github.com/LPD-EPFL/ssmem).
Clone ssmem, do `make libssmem.a` and then copy `libssmem.a` in `CLHT/external/lib` and `include/smmem.h` in `CLHT/external/include`.

Additionally, the sspfd profiler library is required (https://github.com/trigonak/sspfd).
Clone sspfd, do `make` and then copy `libsspfd.a` in `CLHT/external/lib` and `include/sspfd.h` in `CLHT/external/include`.

You can compile the different variants of CLHT with the corresponding target. For example:
`make libclht_lb_res.a` will build the `clht_lb_res` version.

The compilation always produces the `libclht.a`, regardless of the variant that is built.

Various parameters can be set for each variant in the corresponding header file (e.g., `include/clht_lb_res.h` for the `clht_lb_res` version).

To make a debug build of CLHT, you can do `make target DEBUG=1`.

Using CLHT
----------

To use CLHT you need to include the `clht.h` file in your source code and then link with `-lclht -lssmem` (ssmem is used in the CLHT implementation).

The following functions can be used to create and use a new hash table:
  * `clht_t* clht_create(uint32_t num_buckets)`: creates a new CLHT instance.
  * `void clht_gc_thread_init(clht_t* hashtable, int id)`: initializes the GC for hash table resizing. Every thread should make this call before using the hash table.
  * `void clht_gc_destroy(clht_t* hashtable)`: frees up the hash table
  * `clht_val_t clht_get(clht_hashtable_t* hashtable, clht_addr_t key)`: gets the value for a give key, or return 0
  * `int clht_put(clht_t* hashtable, clht_addr_t key, clht_val_t val)`: inserts a new key/value pair (if the key is not already present)
  * `clht_val_t clht_remove(clht_t* hashtable, clht_addr_t key)`: removes the key from the hash table (if the key is present)
  * `void clht_print(clht_hashtable_t* hashtable)`: prints the hash talble
  * `const char* clht_type_desc()`: return the type of CLHT. For example, CLHT-LB-RESIZE.


For full details, refer to the original repository: [http://lpd.epfl.ch/site/ascylib](http://lpd.epfl.ch/site/ascylib)
