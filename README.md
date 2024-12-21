# ARM7TDMI
EXPERIMENTAL repository for ARM7TDMI single-step tests

## Thanks to NanoBoyAdvance and fleroviux!
These tests are generated from fleroviux's excellent NanoBoyAdvance. Any errors in the test are likely errors in my code, not in NBA. Thanks to fleroviux for letting me use NBA for this! I will link both the original repository and my fork for creating these tests, in the near future.

## How to get .json files from this
You must run transcode_json.py after pulling the tests. This will translate the .json.bin format into .json format to easily work with. If you wish to use the binary representation, the .py file should document it fairly clearly, it's a very simple format.

## About
These LIKELY have bugs. Until tested, don't be surprised if you get a wrong result. This is normal for releasing these tests.

These tests currently consist of 20,000 tests per .json file, each one testing a category of encoding. There may be holes or inaccuracies. Only ARM is covered, THUMB will come soon. 

Each test has a list with 20000 entries that look like this (this is from hw_data_transfer_register.json):

```json
  {
    "initial": {
      "R": [
        4172568239,
        3218885584,
        3611667212,
        1565924574,
        126631669,
        898680852,
        2152366193,
        3510576945,
        2585370008,
        2084998222,
        1710718908,
        921484554,
        857686631,
        4076630577,
        496381836,
        3076961312
      ],
      "R_fiq": [
        1729278480,
        3279118443,
        1296478502,
        1311475728,
        2093184835,
        617130518,
        3124009755
      ],
      "R_svc": [
        1848569204,
        3393792212
      ],
      "R_abt": [
        2644803584,
        3237205884
      ],
      "R_irq": [
        4028872293,
        1667487871
      ],
      "R_und": [
        1540154790,
        1437581195
      ],
      "CPSR": 805306395,
      "SPSR": [
        268435671,
        3758096403,
        1879048400,
        268435483,
        268435667,
        805306512
      ],
      "pipeline": [
        1887645885,
        10559586
      ]
    },
    "final": {
      "R": [
        4172568239,
        3218885584,
        3611667212,
        1565924574,
        126631669,
        898680852,
        2152366193,
        3510576945,
        2585370008,
        2084998222,
        1710718908,
        921484554,
        857686631,
        0,
        0,
        3076961320
      ],
      "R_fiq": [
        1729278480,
        3279118443,
        1296478502,
        1311475728,
        2093184835,
        617130518,
        3124009755
      ],
      "R_svc": [
        1848569204,
        3393792212
      ],
      "R_abt": [
        2644803584,
        3237205884
      ],
      "R_irq": [
        4028872293,
        1667487871
      ],
      "R_und": [
        4076630577,
        496381836
      ],
      "CPSR": 805306395,
      "SPSR": [
        0,
        3758096403,
        1879048400,
        268435483,
        268435667,
        805306512
      ],
      "pipeline": [
        10629219,
        10698852
      ]
    },
    "transactions": [
      {
        "kind": 0,
        "size": 4,
        "addr": 3076961312,
        "data": 10629219,
        "cycle": 1,
        "access": 12,
      },
      {
        "kind": 0,
        "size": 4,
        "addr": 3076961316,
        "data": 10698852,
        "cycle": 2,
        "access": 12,
      }
    ],
    "opcodes": [
      1887645885,
      10559586,
      10629219,
      10698852,
      11047017
    ],
    "base_addr": 3076961306
  },
```

### The top-level keys are "initial," "final," "opcodes," "transactions," and "base_addr."

### base_addr
Is the location in RAM where opcode[0] is located. Just the base address of the test.

### Initial and final are the same format.
They contain the initial and final state of all relevant registers in the CPU. Initial is what the CPU is set to before executing the instruction, and final is what all the same registers contain afterward.

* R: R0-R15
* R_fiq: R8-R14 banked for FIQ
* R_svc, R_abt, R_irq, R_und: R13-R14 banked for svc, abt, IRQ, and und.
* CPSR register
* SPSR registers in this order: fiq, svc, abt, irq, und
* pipeline: The contents of the instruction pipeline, in order

Honestly I don't have an ARM7TDMI core yet and I'm not sure if this is the best way to represent this data. Let me know if I can make it better. Moving on...

### Transactions
Are memory transactions.

```json
"kind": 0,
"size": 4,
"addr": 3076961316,
"data": 10698852,
"cycle": 2,
"access": 12
```

* kind is 0 for an instruction read, 1 for a general read, and 2 for a write
* size is in bytes. so 1 = 1 byte, 2 = 16 bits, 4 = 32 bits.
* addr is the address
* data is the data
* cycle is the cycle number this transaction happened on (experimental, I'm not sure if I instrumented NBA correctly for this number)
* access is the access mask provided by NanoBoyAdvance. It is made by ORing these values together:

```cpp
    enum Access {
        Nonsequential = 0,
        Sequential = 1,
        Code = 2,
        Dma = 4,
        Lock = 8
    };
```

## Opcodes
This section is a doozy. Testing a 32-bit RISC processor isn't like testing an 8-bit processor. We can't just allocate a flat 4 gigs of RAM and go. (Well, maybe many of us can, but it's not a good idea). Instead, we have a list of transactions to search through and match our transactions against, and a list of opcodes that are given in different circumstances.

Here is what the opcodes in the list ARE:

* 0: is the opcode being tested
* 1: is ADC R1, R2, store in R2
* 2: is ADC R2, R3, store in R3
* 3: is ADC R3, R4, store in R4
* 4: is ADC R8, R9, store in R9

All of the opcodes other than the specific one being tested were chosen specifically so that (a) they would be very simple (no shifts etc.) and easy to rely on, and (b) they would have different side-effects. This should help you determine where something goes wrong.

Opcode 0 is loaded into pipeline slot 0, opcode 1 is in pipeline slot 1, and R15 is set to the address of opcode 2 (+8 from the about-to-execute instruction, ot +4 for THUMB).

The CPU is set up in this way, and runs 1 instruction.

Opcode 0 only should be executed.

### Putting it all together

When your CPU issues a read or write, you should look it up in the list of transactions and compare it. If it isn't found in the transactions, you should return opcode 4 as the value, as that will have a side effect one way or another.

In pseudocode that looks something like this...

```python
def read(addr, is_code):
    if not is_code:
        return lookup_transaction(addr)
    if (addr >= test.base_addr) and (addr <= (test.base_addr + 12)):
        diff = (addr - test.base_addr) / 4
        return test.opcodes[diff]
    else:
        return test.opcodes[4]
```

## Known Issues
* Some illegal edge cases (which can't even be produced by assembler) where write-back register is r15 and destination register is r15, may not have perfectly correct behavior

## Disclaimers

* The tests do not properly restrict read and write alignment, other than instructions. Let me know if it's important to change this
* The tests treat RAM as a 32-bit flat space with no memory-mapped registers. Unless you want to allocate 4GB RAM, I suggest you use our transaction-based method.
* The tests may have bugs, this is an in-development release v0.1
* There may be issues with the tests we don't know yet.
* As of yet, the correct number of cycles and transactions happening on the correct cycle are not recorded. The correct order is.

I hope you find it useful!
