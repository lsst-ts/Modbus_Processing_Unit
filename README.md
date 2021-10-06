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

## Usage

Add the library to your project. Create MPU object, and execute its Run method.
You need to provide Command, Telemetry and Debug (Output) FIFOs, and port for
commands. See [Example.vi](Example.vi) for details.

## Supported instructions

| Instruction | Description                                             |
| ----------- | ------------------------------------------------------- |
| 0           | Stop executing and wait for a new instructions in FIFO. |
| 1           | Write the next byte to the port.                        |
| 2           | Wait for given ms.                                      |
| 3           | Read from the port and store data into output array.    |
| 4           | Loop (jump to some instruction, usually 0 to repeat the sequence; with looping, a check for commanding FIFO will be provided, and if that contains new instructions, instruction stack will be replaced with those new instructions). |
| 5           | Check readout [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check), assumes CRC are two last bytes read. If CRC doesn't match, error 6000 is reported and FPGA processing stops. |
| 6           | Write data to output FIFO.                              |
| 20          | Write multiple bytes. Followed by number of bytes and paylod (bytes to transwer). |
| 30          | Write single byte to telemetry. Instruction shall be followed with telemetry address offset and output array offset. |
| 31          | Write two bytes to telemetry. Instruction shall be followed with telemetry address offset and output array offset (of the first byte). |
| 32          | Write four bytes to telemetry. Instruction shall be followed with telemetry address offset and output array offset (of the first byte). |
| 33          | Write eight bytes to telemetry. Instruction shall be followed with telemetry address offset and output array offset (of the first byte). |
| 255         | Stops application loop, exit FPGA application. |

### Telemetry

Telemetry commands (30-34) writes big endian numbers. Telemetry uses [Common
FPGA HealthAndStatus](https://github.com/lsst-ts/Common_FPGA_HealthAndStatus)
update FIFO.

### Example

When following is passed to input FIFO, the processing unit will:

1. call for function 58 from address 1, CRC is 0x3380
2. waits 100 milliseconds for response
3. reads 18 bytes from device
4. check CRC
5. writes two first bytes from payload (indices 2 and 3; first two bytes are
   address and function) to telemetry at addrress 11
6. writes eight bytes from payload indices 4-11 to telemetry at address 22
7. writes four bytes (indices 12-15) to telemetry at address 67
8. waits 500 milliseconds
9. closes loop (call again function 58,..)

The loops get repeated as long as new instructions aren't available on
commanding FIFO, or an error occurs.

0. 23  *sends 23 bytes as commands/data*
1. 20 4 1 58 0x80 0x33
2. 2 100
3. 3 18
4. 5
5. 31 11 2
6. 33 22 4
7. 32 67 12
8. 2 500
9. 4

Assuming device response is (CRC is 0x6676, hex numbers are prefixed with 0x):

1 58 0x11 0x12 0x13 0x14 0x15 0x16 0x17 0x18 0x19 0x20 0x21 0x22 0x23 0x24 0x76 0x66

The following commands will be send to telemetry FIFO:

* set address 11 to 0x1112
* set address 22 to 0x1314151617181920
* set address 67 to 0x21222324

This is implemented in unit test 14.

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
