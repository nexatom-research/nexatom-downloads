# Nexatom Downloads & SDK Documentation

Welcome to the official repository for [Nexatom Research & Instruments](https://www.nexatom.in). This repository hosts the public releases, SDK binaries, and documentation for our precision scientific instruments, including the **UTT810 Universal Time Tagger**.

## About Nexatom

Nexatom mobilizes research knowledge into practical systems for precision light, timing, and scientific measurement. We develop indigenous high-technology solutions for precision lasers, time tagging, and custom scientific instrumentation used by research groups and industry teams.

## The NexatomTT SDK (UTT810)

The NexatomTT SDK provides host-side control of the UTT810 time tagger over USB 3.0. It features a native C-library and a Python wrapper for seamless integration into custom data pipelines, automated experiments, and live analysis (such as TIHI and MFCO plotting).

## [Documentation](https://nexatom-research.github.io/nexatom-downloads)

* **[Getting Started](docs/01_getting_started/index.md)**
    * [Installation Guide](docs/01_getting_started/1_1_installation.md)
        - [System requirements](docs/01_getting_started/1_1_installation.md#system-requirements)
        - [FTDI D3XX driver installation](docs/01_getting_started/1_1_installation.md#ftdi-d3xx-driver-installation)
        - [SDK package contents](docs/01_getting_started/1_1_installation.md#sdk-package-contents)
        - [Verifying the installation](docs/01_getting_started/1_1_installation.md#verifying-the-installation)
        - [SDK home discovery](docs/01_getting_started/1_1_installation.md#sdk-home-discovery)
    * [Quick Start Workflow](docs/01_getting_started/1_2_quick_start.md)
        - [Device lifecycle](docs/01_getting_started/1_2_quick_start.md#device-lifecycle)
        - [Realtime CPS and telemetry streaming](docs/01_getting_started/1_2_quick_start.md#realtime-cps-and-telemetry-streaming)
        - [Raw time-tag capture and CSV export](docs/01_getting_started/1_2_quick_start.md#raw-time-tag-capture-and-csv-export)
        - [Live TIHI and MFCO plotting](docs/01_getting_started/1_2_quick_start.md#live-tihi-and-mfco-plotting)
    * [Programming Languages](docs/01_getting_started/1_3_programming.md)
        - [C / C++ integration](docs/01_getting_started/1_3_programming.md#c-cpp-integration)
        - [Python integration](docs/01_getting_started/1_3_programming.md#python-integration)
        - [FFI / Foreign language bindings](docs/01_getting_started/1_3_programming.md#ffi-foreign-language-bindings)
    * Firmware Management (Drafting...)

## Downloads

You can find the latest releases and binaries in the [Releases](https://github.com/nexatom-research/nexatom-downloads/releases) section.

## Support

If you encounter a bug, need to report an issue with the SDK, or have questions about integrating the API, please open an **[Issue](https://github.com/nexatom-research/nexatom-downloads/issues)** on this repository.

For custom instrumentation discussions or hardware support, please [contact us directly](https://www.nexatom.in/).

## License

This SDK and its documentation are provided under the Nexatom End-User License Agreement. See the `LICENSE.txt` file included in your release download for full terms. Third-party components (such as FTDI and MinGW runtimes) remain under their respective licenses.

---

**Last Updated:** Sep 8, 2026
