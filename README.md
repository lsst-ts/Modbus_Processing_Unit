# Modbus Processing Unit (MPU)

Provides a common modbus communication library, which can be left running in
FPGA and provide telemetry. The major idea is to create in FPGA a simple CPU (a
Turing Machine if you want), narrowly designed into reading out data from
Modbus devices. The design is:

* input FIFO with commands
* array storage for commands (instructions stacks), which can be updated from
  FIFO when one loop of the library finished
* processing unit. That takes commands from instructions stacks, and executed
  them
* output array. There responses from Modbus devices are stored, as commanded
  (readout command)
* output to telemetry FIFO, with commands to update telemetry memory with
  values readout from device
* optional raw output to output FIFO

Those instructions are supported:

0. stop executing and wait for a new instructions in FIFO
1. write the next byte to the port
3. wait for given ticks/ms/us/..
4. read from the port and store data into output array
4. loop (jump to some instruction, usually 0 to repeat the sequence; with
   looping, a check for commanding FIFO will be provided, and if that contains
   new instructions, instruction stack will be replaced with those new
   instructions)
5. write data to output FIFO
x. check CRC of data stored in output array
x. write something (transformed part of output array) to telemetry FIFO
x. optionally (if we need it) conditional branching can be provided

This shall fit most of the single use Modbus devices, usually read-only
devices. Shall enable to execute monitoring (readout of Modbus
registers/values) inside FPGA, and use CPU only to get data from telemetry
memory where those will be stored. Optionally shall allow us to read from the
port and get back raw replies to check what is wrong with the device (enable
direct communication).

## Error handling

On errors, something has to be written into telemetry and processing ought usually stop; we might devise options to handle errors better, but that shall be the first approach. It's left up to realtime (CPU application) to handle errors (check telemetry library the unit isn't executing/writing data) and decide what to do (usually, as this violates safety rules, CSC will be commanded to transition into fault state).

Inconclusive list of errors:

* wrong/unknow instruction
* went past instruction array end
* unable to write to the port
* timeout reading data from the port
* invalid Modbus CRC/checksum
