# Hardening Macros

Using openlane, you can produce a GDSII from an RTL for macros, and then use these macros to create your chip. Check [this][4] for more details about chip integration.

In this document we will go through the hardening steps and discuss in some detail what considerations should be made when hardening your macro.

**NOTE:** For all the configurations mentioned in this documentation and any other openlane documentation, you can use the exploration script `run_designs.py` to find the optimal value for each configuration. Read more [here][6]. 

## Base Requirements:

You should start by setting the basic configuration file for your design. Check [this][5] for how to add your new design.

The basic configurations `config.tcl` should at least contain:
```tcl
set ::env(DESIGN_NAME) <the name of the design>

set ::env(VERILOG_FILES) <pointer to the verilog source(s) of the design>
set ::env(CLOCK_PORT) <list of clock ports in the design rtl>
set ::env(DESIGN_IS_CORE) 0 # Since you are hardening a macro.
```

If you are hardening the chip core. Then make sure to set `::env(DESIGN_IS_CORE)` to `1` and check [this][4] for more details about chip integration.

In case your design doesn't have a clock port, then you can either omit ::env(CLOCK_PORT) or give it an empty string.

These configurations should get you through the flow with the all other configurations using openlane default values, read about those [here][0]. However, in the coming sections we will take a closer look on how to determine the best values for most of the other configurations.

## Synthesis

The first decision in synthesis is determining the optimal synthesis strategy `::env(SYNTH_STRATEGY)` for your design. For that purpose there is a flag in the `flow.tcl` script, `-synth_explore` that runs a synthesis strategy exploration and reports the results in a table under `<run_path>/reports/`.

Then you need to consider the best values for the `SYNTH_MAX_FANOUT`. 

If your macro is huge (200k+ cells), then you might want to try setting `SYNTH_NO_FLAT` to `1`.

Other configurations like `SYNTH_SIZING`, `SYNTH_BUFFERING`, and other synthesis configurations don't have to be changed. However, the advanced user can check [this][0] documentation for more details about those configurations and their values.

## Static Timing Analysis

Static Timing Analysis happens multiple times during the flow. However, they all use the same base.sdc file. You can control the outcome of the static timing analysis by setting those values:

1. The clock ports in the design, explained in the base requirements section `CLOCK_PORT`. 

2. The clock period that you prefer the design to run with. This could be set using `::env(CLOCK_PERIOD)` and the unit is ns. It's important to note that the flow will use this value to calculate the worst and total negative slack, also if timing optimizations are enabled, it will try to optimize for it and give suggested clock period at the end of the run in `<run-path>/reports/final_summary_report.csv` This value should be used in the future to speed up the optimization process and it will be the estimated value at which the design should run.

3. The IO delay percentage from the clock period `IO_PCT`. More about that [here][0]. 

Other values are set based on the (PDK, STD_CELL_LIBRARY) used. You can read more about those configurations [here][0].

## Floorplan

During Floor plan, you have one of two options:

1. Let the tools determine the area relative to the size and number of cells. This is done by setting `FP_SIZING` to `relative` (the default value), and setting `FP_CORE_UTIL` as the core utilization percentage. Also, you can change the aspect ratio by changing `FP_ASPECT_RATIO`.

2. Set a specific DIE AREA by making `FP_SIZING` set to `absolute` and then giving the size as four coordinates to `DIE_AREA`.

You can read more about how to control these variables [here][0].

## IO Placement

## Placement

## Clock Tree Synthesis

## Power Grid (pdn)

Each macro in your design should configure the `common_pdn.tcl` for it's use. Generally, you want to announce that the design is a macro inside the core, and that it doesn't have a core ring. This is done by setting the following configs:

```tcl
set ::env(DESIGN_IS_CORE) 0
set ::env(FP_PDN_CORE_RING) 0
```

Pdngen configs for macros contain a special stdcell section, instead of the one used for the core in the `common_pdn.tcl`. The purpose of this is to prohibit the use of metal 5 in the power grid of the macros and use it exclusively for the core and top level.

If your macro contains other macros inside it. Then make sure to check the `macro` section and see if it requires any modifications for each them depending on their special configs. The following example is using default `connect` section and a different `straps` section:
```tcl
pdngen::specify_grid macro {
    orient {R0 R180 MX MY R90 R270 MXR90 MYR90}
    power_pins "vpwr vpb"
    ground_pins "vgnd vnb"
    straps {
	    met1 {width 0.74 pitch 3.56 offset 0}
	    met4 {width $::env(FP_PDN_VWIDTH) pitch $::env(FP_PDN_VPITCH) offset $::env(FP_PDN_VOFFSET)}
    }
    connect {{met1 met4}}
}
```

**WARNING:** only use met1 and met4 for and rails straps.

Refer to [this][3] for more details about the syntax in case you needed to create your own `pdn.tcl` and then point to it using `::env(PDN_CFG)`.

- The height of each macro must be greater than or eaqual to the value of `$::env(FP_PDN_HPITCH)` to allow at least two metal 5 straps on the core level to cross it and all the dropping of a via from met5 to met4 connecting the vertical straps of the macro to the horizontal straps of the core and so connect the power grid of the macro to the outer core ring.

If your design is a core, then check the power routing section in the [chip integration documentation][4]. However, the basic thing you'll need to do is to announce that your design is a chip core and that it should have a core ring. This is done by setting the following values:

```tcl
set ::env(DESIGN_IS_CORE) 1
set ::env(FP_PDN_CORE_RING) 1
```

## Routing

## Final Reports and Summaries


[0]: ./../configuration/README.md
[1]: ./OpenLANE_commands.md
[2]: ./advanced_readme.md
[3]: https://github.com/The-OpenROAD-Project/OpenROAD/blob/openroad/src/pdngen/doc/PDN.md
[4]: ./chip_integration.md
[5]: ./../designs/README.md
[6]: ./../regression_results/README.md5