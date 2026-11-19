# Project 2 pin assignments

Copy the pin-assignment file produced by Genus `write_dft_atpg` into this directory, then make `top_loc.pinassign` and `top_los.pinassign` from it. These assignments describe the design's actual clocks, scan enable, scan inputs and outputs, and test constants. Confirm every name against the generated scan-chain report and netlist.

For this comparison, the test modes use the same design pins. LOC versus LOS is selected by the dynamic ATPG sequence filter in `scripts/run_modus.tcl`; separate assignment files are kept because the two test modes are built and checked independently. Do not put guessed port names in these files.
