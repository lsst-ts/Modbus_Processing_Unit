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
* output to telemetry FIFO register values, etc.
* optional (user commanded) raw output to output FIFO

## Usage

Add the library to your project. Create MPU object, and execute its Run method.
You need to provide Command, Telemetry and Debug (Output) FIFOs, and port for
commands. See [Example.vi](Example.vi) for details.

## Command queue

Command queue uses own multiplex to clear MPU error lines, program MPU or
report port status. The command queue is checked for new command after run of
a programmed loop (which is empty on startup). If command is available, it is
executed, if not, programmed code is executed. This quarantees programming will
not mess with program execution, as both cannot happen at the same time.

| Code | Description                                            |
| ---- | -------------------------------------------------------|
| 0    | Reset error lines. If the program fails, error lines will be set. See unit telemetry for how to get current error lines status. |
| 1    | Program MPU. Followed by single byte - program length, and the program itself. See below for supported instructions.            |
| 2    | Report port telemetry to Telemetry FIFO.                                 |
| 3    | Reset unit telemetry counters.                                          |
| 4    | Sets write and read timeouts to the following 2 16bit big endian values. |

## Supported instructions

| In. | Description                                             |
| --- | ------------------------------------------------------- |
| 0   | Stop executing and wait for a new instructions in FIFO. |
| 1   | Write the next byte to the port.                        |
| 2   | Wait for given ms.                                      |
| 3   | Read from the port and store data into output array.    |
| 4   | Loop (jump to some instruction, usually 0 to repeat the sequence; with looping, a check for commanding FIFO will be provided, and if that contains new instructions, instruction stack will be replaced with those new instructions). |
| 5   | Check readout [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check), assumes CRC are two last bytes read. If CRC doesn't match, error 6000 is reported and FPGA processing stops. |
| 6   | Write data to output FIFO and forgots content of output memory.     |
| 20  | Write multiple bytes. Followed by number of bytes and paylod (bytes to transfer). |
| 30  | Write byte(s) to telemetry. Instruction shall be followed with number of bytes and output array offset. |
| 100 | Set write and read timeouts. Shall follow with 2 16 bit big endian values for write and read timeout in ms. |
| 255 | Stops application loop, exit FPGA application.          |

### Telemetry

Telemetry command (30) dumps U8 values. Offset is calucalted from 0. This can
be used to periodically dump measured values.

## Unit telemetry

After command 2 on command FIFO, unit telemetry is reported. The telemetry
contains those values (all in big endian):


| Offset  | Description                                                      |
| ------- | ---------------------------------------------------------------- |
| 0-1     | Current Instruction Pointer (IP).                                |
| 2-9     | Out counter. Bytes send so far.                                  |
| 10-17   | In counter. Bytes received so far.                               |
| 18-25   | Out timeouted counter. Number of output writes timeouted so far. |
| 26-33   | In timeouted counter. Number of input reads timeouted so far.    |
| 34-35   | IP on error. Set to IP of last failed instruction.               |
| 36-37   | Port write timeout value (in ms).                                |
| 38-39   | Port read timeout value (in ms).                                 |
| 40      | Error status. 1 for error, 0 for no errors.                      |
| 41-42   | Error code.                                                      |
| 43-44   | Data Modbus CRC-16.                                              |

* output counter - how many bytes were written to the port
* input counter - how many bytes were received on the port
* output timeout - how many writes timeouted
* input timeout - how many reads timeouted

## Unit error lines - error handling

Current error state and code can be found in the unit telemetry - see above.

The following strategy is used for error handling:

* Errors inside operational part, which handles port communication etc., are
  connected. That means if error occurred in some preceding step, the operation
  will not be carried (as its input error wire is active)
* Loops where error can signal problems must stop when error occurs
* Error is being cleared on MPU Multiplex commands 0 and 1, e.g. either
  explicitly (on command 0) on implicitly (on command 1, before executing new
  commands; this makes sense, as when you command MPU, you would like to see
  responses regardless of previous problems).
* Error lines aren't connected in MPUMultiplex handling. This is to prevent
  error triggered inside MPU unit (when serial communication is handled) to
  stop new commanding.

That allows new commands to be accepted regardless of any errors, which is
intended behavior (otherwise commands to reset errors or query telemetry to see
the errors would not be accepted).

### Example

When following is passed to input FIFO, the processing unit will:

1. call for function 58 from address 1, CRC is 0x3380
2. waits 100 milliseconds for response
3. reads 18 bytes from device
4. check CRC
5. writes two first bytes from payload (indices 2 and 3; first two bytes are
   address and function) to telemetry at addrress 11
6. waits 500 milliseconds
7. closes loop (call again function 58,..)

The loops get repeated as long as new instructions aren't available on
commanding FIFO, or an error occurs.

0. 17  *sends 17 bytes as commands/data*
1. 20 4 1 58 0x33 0x80
2. 2 100
3. 3 18
4. 5
5. 30 18 0
7. 2 500
8. 4

Assuming device response is (CRC is 0x6676, hex numbers are prefixed with 0x):

1 58 0x11 0x12 0x13 0x14 0x15 0x16 0x17 0x18 0x19 0x20 0x21 0x22 0x23 0x24 83 4

This will be also available in Telemetry FIFO

This is implemented in unit test "README.md".

## Future (currently unsupported) instructions:

* verify response address and function (to make sure device is responding to what we asked for)
* optionally (if we need it) conditional branching can be provided

This shall fit most of the single use Modbus devices, usually read-only
devices. Shall enable to execute monitoring (readout of Modbus
registers/values) inside FPGA, and use CPU only to get data from telemetry
memory where those will be stored. For debugging, raw traffic can be commanded
and replies can be stored.

## Error handling

On errors, something has to be written into telemetry and processing ought
us
ually stop; we might devise options to handle errors better, but that shall
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

### Custom error values

* **-21** cannot write data - write timeouted
* **-22** cannot read data - read timeouted
* **-23** invalid CRC

# Unit Tests

The "MPU Test" project contains unit tests for FPGA IP. Just open it and run
tests (context menu, Run item) from "DEN Unit Tests.lvtest". DEN stays for
Desktop Execution Node, used to execute simulated FPGA on host PC.
