This directory contains the hardware implementation of the processor. There are
three directories:
- core/
  The GPGPU. The top level module is 'nyuzi'. Configurable options (cache size,
  associativity, number of cores) are in core/config.sv
- fpga/
  Components of a quick and dirty system-on-chip test environment. These
  are not part of the Nyuzi core, but are here to allow testing on FPGA.
  Includes an SDRAM controller, VGA controller, AXI interconnect, and other
  peripherals like a serial port. (Documentation is
  [here](https://github.com/jbush001/NyuziProcessor/wiki/FPGA-Test-Environment)).
  The makefile for the DE2-115 board target is in fpga/de2-115.
- testbench/
  Support for simulation, including mock peripherals.

This project uses Emacs [Verilog Mode](http://www.veripool.org/wiki/verilog-mode)
to automatically generate wire definitions and resets. If you have Emacs installed,
type 'make autos' from the command line to update the definitions in batch mode.

When building for simulation, the preprocessor macro `SIMULATION` is defined.
This is used in the code to disable portions of code during synthesis. If you
are creating a simulation project for another toolchain, make sure this is
defined.

This design uses parameterized memories (FIFOs and SRAM blocks) in the modules
core/sram_1r1w.sv, core/sram_2r1w.sv, and core/sync_fifo.sv. By default, these
instantite simulator versions, which are not synthesizable (at least not
efficiently).

- For Altera parts, the build files define the preprocessor macro
  `VENDOR_ALTERA, which will use the megafunctions ALTSYNCRAM and SCFIFO.
- If you want to use this with a different vendor, create a `VENDOR_xxx define and
  add a new section that uses the appropriate module.
- For tools that generate memories using a separate memory compiler, running
  `make core/srams.inc` will generate an include file with all used memory
  sizes in the design. You can tweak the script tools/misc/extract_mems.py to
  change the module names or parameter formats.

This project uses [Verilator](http://www.veripool.org/wiki/verilator) for
simulation by default. Typing make in this directory compiles an executable
'nyuzi_vsim' in the bin/ directory. It accepts the following command
line arguments (Verilog prefixes arguments with a plus sign):

|          Argument               | Meaning        |
|---------------------------------|----------------|
| +bin=*hexfile*                  | Load this file into simulator memory at address 0. Each line contains a 32-bit little endian hex encoded value. |
| +trace                          | Print register and memory transfers to standard out.  The cosimulation tests use this to verify operation. |
| +statetrace                     | Write thread states each cycle into a file called 'statetrace.txt', read by visualizer app (tools/visualizer). |
| +memdumpfile=*filename*         | Write simulator memory to a binary file at the end of simulation. The next two parameters must also be specified for this to work |
| +memdumpbase=*baseaddress*      | Base address in memory to start dumping (hexadecimal) |
| +memdumplen=*length*            | Number of bytes of memory to dump (hexadecimal) |
| +autoflushl2                    | Copy dirty data in the L2 cache to system memory at the end of simulation before writing to file (used with +memdump...) |
| +profile=*filename*             | Periodically write the program counters to a file. Use with tools/misc/profile.py |
| +block=*filename*               | Read file into virtual block device, which it exposes as a virtual SD/MMC device.<sup>1</sup>
| +randomize=*\[1\|0\]*              | Randomize initial register and memory values. Used to verify reset handling. Defaults to on.
| +randseed=*seed*                | If randomization is enabled, set the seed for the random number generator.
| +dumpmems                       | Dump the sizes of all internal FIFOs and SRAMs to standard out and exit. Used by tools/misc/extract_mems.py |
| +jtag_port=*port*               | Opens a socket waiting for a connection on the given port. Commands received here will be sent over JTAG. See sim_jtag.sv for more details |

The amount of RAM available in the testbench is hard coded to 16MB. To alter
it, change MEM_SIZE in testbench/verilator_tb.sv.

The simulator exits when all threads halt by writing to the appropriate control
register.

To write a waveform trace, set the environment variable DUMP_WAVEFORM
and rebuild:

    make clean
	 cd ..
    DUMP_WAVEFORM=1 cmake .
	 cd hardware
	 make

The simulator writes a file called `trace.vcd` in
"[value change dump](http://en.wikipedia.org/wiki/Value_change_dump)"
format in the current working directory. This can be with a waveform
viewer like [GTKWave](http://gtkwave.sourceforge.net/).
Waveform files get enormous quickly. Even running a minute of simulation will
produce gigabytes of trace data. A trick to reduce the size is to use
verilator pragmas to selectively disable tracing for modules. Working in
hardware/testbench/soc_tb.sv, start by disabling all tracing:

    +/*verilator tracing_off*/
     module soc_tb(

Then, selectively re-enable it for modules you are interested in by adding a
pragma above the module instantiation:

    /*verilator tracing_on*/
    my_module module_i_want_to_trace(.*);
    /*verilator tracing_off*/

The timescale is set to 1 ns by default, which simulates a 1 GHz clock speed.

### Support for VCS:

Template scripts have been added to support building and running with
[VCS](https://www.synopsys.com/verification/simulation/vcs.html).
The VCS scripts are located in the scripts/ directory.

vcsbuild.pl requires no arguments and builds the model. It will create an
executable named simv and two support directories csrc and simv.daidir all located
in the scripts/ directory. vcsbuild.pl uses TOP.sv in the hardware/testbench
directory as the testbench top. TOP.sv in the VCS build is analogous to
verilator_main.cpp in the verilator build.

vcsrun.pl will run simulation. It accepts plus arguments in the same way as the
verilor_model. Any plus argument that is Verilog specific should work. It also
supports +randomize=*\[1\|0\]* and +randseed=*seed*

vcs.config in the build/ directory is used to configure the paths for VCS and Verdi
for both scripts (edit the file according to your site's installation).

The Makefile in the hardware/ directory can be used to build the VCS model by
executing:

    % make vcsbuild

Waveform dumping can be enabled by executing:

    % make clean
    % DUMP_WAVEFORM=1 make vcsbuild

If waveform dumping is enabled in VCS, the simulator writes a file called
`trace.fsdb` which can be opened with Verdi.

Several apps can be found in the software/apps/ directory: doom, hello_world,
mandelbrot, plasma, quakeview, rotozoom, sceneview. The Makefile for doom,
hello_world, mandelbrot, quakeview, and sceneview support simulation with
Verilator by executing:

    % ./run_verilator

Similarly, the same apps support simulation with VCS by executing:

    % ./run_vcs
