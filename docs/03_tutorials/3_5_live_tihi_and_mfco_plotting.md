# 3.5 Live TIHI and MFCO plotting

Install Matplotlib in the Python environment used for this optional tutorial. The primary headless acquisition templates need no plotting dependency.

```sh
# Show the plotter's distinct pulse, histogram and analysis options first.
python python/examples/tihi_mfco_matplotlib.py --help
# Save plots and CSV without opening a live plot window.
python python/examples/tihi_mfco_matplotlib.py --home . --duration-sec 10 --save-plots --save-csv --no-live-window
```

This plotter uses internal test pulses by default; choose `--no-test-pulses` for external START/STOP signals. Read its options before changing pulse cycles, channel delay, bin width or coincidence window. Pulse timing and delay must fit the native profile, and TIHI limits must fit the API/capabilities.

TIHI measures START-to-STOP intervals; a nonzero peak requires a defined input relationship. Same-phase pulses do not establish a nonzero physical peak. MFCO analysis-channel selection filters returned pattern bins in software; it is not hardware input gating. Distinguish an exact selected-channel pattern from patterns containing those channels plus others.

Keep plots as a view of the data, not a replacement for completion/quality checks and saved numerical results. For a reusable recording application start with the [processed template](index.md#processed-and-raw-template-anatomy).

[Tutorials](index.md)
