# 2.9 System control

## System enable / disable

`enable_system(True)` controls the system data path. Connection readiness, output selection, individual measurement enables and file saving are separate controls. An already-running firmware may remain stopped after connection; explicitly enable the system when starting your intended measurement.

The primary processed/raw templates prepare settings and sinks while quiet, then enable/start the required path. Shutdown explicitly quiets it again. Do not add an implicit enable to a library-load or identity-only action.

## Peripheral reset

`reset_peripherals(True)` asserts reset and `reset_peripherals(False)` releases it. A complete deliberate reset needs both stages. Reset is not a replacement for the native output barrier or a mandatory step for every acquisition.

## Global acquisition stop

`request_global_stop_all_modes()` requests stopping measurement engines. Check its return and any expected terminal result; do not assume a command return has already delivered all final packets. Keep the required output and savers alive while waiting for terminal results, then select quiet output and finalize files.

For C/C++ see [system control](../07_c_api/7_5_system_control.md). Use the full template cleanup so one error does not skip all remaining retirement steps.

[Device operation](index.md)
