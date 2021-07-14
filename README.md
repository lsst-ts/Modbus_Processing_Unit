# Modbus Processing Unit (MPU)

Provides a common modbus communication library (IP, Intellectual Property),
which can be left running in FPGA and provide telemetry. The major idea is to
create in FPGA a simple CPU (a Turing Machine if you want), designed for
reading out data from Modbus devices. The design includes:

* input FIFO with commands
* array storage for commands (instructions stacks), which can be updated from
  FIFO when one loop of the unit finished
* processing unit. That takes commands from instructions stacks, and executed
  them
* output array. The responses from Modbus devices are stored in it, as
  commanded (readout command)
* output to telemetry FIFO, with commands to update telemetry memory with
  values readout from device
* optional (user commanded) raw output to output FIFO

Those instructions are supported:

0. stop executing and wait for a new instructions in FIFO
1. write the next byte to the port
2. wait for given ms
3. read from the port and store data into output array
4. loop (jump to some instruction, usually 0 to repeat the sequence; with
   looping, a check for commanding FIFO will be provided, and if that contains
   new instructions, instruction stack will be replaced with those new
   instructions)
5. check read CRC, assumes cRC are two last bytes read
6. write data to output FIFO
255. stops application loop, exit FPGA application

Future (currently unsupported) instructions:

* check [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) of data
  stored in output array
* write something (transformed part of output array) to telemetry FIFO
* optionally (if we need it) conditional branching can be provided

This shall fit most of the single use Modbus devices, usually read-only
devices. Shall enable to execute monitoring (readout of Modbus
registers/values) inside FPGA, and use CPU only to get data from telemetry
memory where those will be stored. For debugging, raw traffic can be commanded
and replies can be stored.

## Error handling

On errors, something has to be written into telemetry and processing ought
usually stop; we might devise options to handle errors better, but that shall
be the first approach. It's left up to realtime (CPU application) to handle
errors (check telemetry library the unit isn't executing/writing data) and
decide what to do (usually, as this violates safety rules, CSC will be
commanded to transition into fault state).

Inconclusive list of errors:

* wrong/unknow instruction
* went past instruction array end
* unable to write to the port
* timeout reading data from the port
* invalid Modbus CRC/checksum

# Unit Tests

The "MPU Test" project contains unit tests for FPGA IP. Just open it and run
tests (context menu, Run item) from "DEN Unit Tests.lvtest". DEN stays for
Desktop Execution Node, used to execute simulated FPGA on host PC.
