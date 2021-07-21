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

## Supported instructions

| Instruction | Description                                             |
| ----------- | ------------------------------------------------------- |
| 0           | Stop executing and wait for a new instructions in FIFO. |
| 1           | Write the next byte to the port.                        |
| 2           | Wait for given ms.                                      |
| 3           | Read from the port and store data into output array.    |
| 4           | Loop (jump to some instruction, usually 0 to repeat the sequence; with  looping, a check for commanding FIFO will be provided, and if that contains new instructions, instruction stack will be replaced with those new instructions). |
| 5           | Check readout [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check), assumes CRC are two last bytes read. If CRC doesn't match, error 6000 is reported and FPGA processing stops. |
| 6           | Write data to output FIFO.                              |
| 20          | Write multiple bytes. Followed by number of bytes and paylod (bytes to transwer). |
| 30          | Write single byte to telemetry. Instruction shall be followed with telemetry address offset and output array offset. |
| 255         | Stops application loop, exit FPGA application. |

Future (currently unsupported) instructions:

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
be the first approach. It's left up to real-time (CPU application) to handle
errors (check telemetry library the unit isn't executing/writing data) and
decide what to do (usually, as this violates safety rules, CSC will be
commanded to transition into fault state).

Inconclusive list of errors:

* wrong/unknow instruction
* went past instruction array end
* unable to write to the port
* timeout reading data from the port
* invalid Modbus CRC/checksum (reported in Error output)

# Unit Tests

The "MPU Test" project contains unit tests for FPGA IP. Just open it and run
tests (context menu, Run item) from "DEN Unit Tests.lvtest". DEN stays for
Desktop Execution Node, used to execute simulated FPGA on host PC.
