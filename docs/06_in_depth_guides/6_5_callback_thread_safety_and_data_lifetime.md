# 6.5 Callback thread safety and data lifetime

Callbacks may run on native worker threads or synchronously during an API call. Connection registration can invoke its callback inline, and field-update progress callbacks run within the calling operation. Do not assume a UI thread. Return promptly; transfer plotting, disk export and expensive analysis to application work queues, and protect shared state appropriately.

## C and C++

Most fixed data callbacks receive a by-value record. Its fields can be copied into your own storage, but a pointer to that local argument becomes invalid when the callback returns. By-value delivery does not extend a C local variable's lifetime. Keep the registered function and `user_data` alive until native device destruction completes.

**The C versioned configuration-dump callback is a pointer-based exception.** It receives a borrowed view whose records pointer also expires when the callback returns. Copy the view's metadata and deep-copy its valid records into application-owned storage before returning. Copying only the top-level view leaves a dangling records pointer. Python's public wrapper performs an owned deep copy for this view.

```c
/* Fragment: the application owns slot and synchronizes it with its consumer. */
static void receive_cps(nexatom_cps_data_t data, void *user_data) {
    nexatom_cps_data_t *slot = (nexatom_cps_data_t *)user_data;
    *slot = data; /* Copy the record; never store &data for later use. */
}
```

This fragment requires application synchronization if another thread accesses `slot`; it is not a lock-free queue implementation. Use the packaged complete templates for practical ownership.

## Python

The public wrapper copies records before invoking Python handlers and retains ctypes callback functions. An owned Python record can be queued, but application concurrency and bounded-memory policy remain your responsibility. Do not replace the public wrapper with a raw callback that retains borrowed memory.

## Retirement and reentrancy

Clearing or replacing ordinary data registrations does not fence an invocation already selected by native. The Python binding retains retired callback references until native destruction. C/FFI callers must likewise keep handler code and user data alive through device destruction; garbage collection is not a native lifetime guarantee.

Logging has a separate process-global `unregister_log_callback` with explicit quiescence. Successful logging unregister permits releasing its resources; timeout/BUSY/failure does not. Do not apply that stronger logging contract to ordinary device callback setters.

Do not destroy/close a device, replace callbacks, unregister logging or synchronously wait for the same callback from within a native callback. Unsupported reentrant operations may be rejected. Keep a control thread responsible for start/stop/cleanup, and report cleanup errors as part of the run outcome.

[In-depth guides](index.md) · [Python callbacks](../05_api_reference/5_4_callback_types.md)
