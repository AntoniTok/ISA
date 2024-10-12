# Lab 1

## Task 1

### counter.sv

```verilog
module counter #(
    parameter WIDTH = 8
)(
    // Interface signals
    input logic clk,
    input logic rst,
    input logic en,
    output logic [WIDTH-1:0] count
);

always_ff @ (posedge clk)
    if (rst) 
        count <= {WIDTH{1'b0}};
    else 
        count <= count + {{WIDTH-1{1'b0}}, en};

endmodule
```

This verilog module describes a parameterized counter.

Module Declaration and Parameterization:

>module counter #(parameter WIDTH = 8): 

We declare a counter module with a parameter WIDTH which defines the bit-width of the counter, the default should be 8 bits, but this can be overridden when the module is instantiated.

Interface Signals : 

The clock signal : `input logic clk`

The synchronous reset signal : `input logic rst`

The enable signal, we use it to control when the counter increments : `input logic en`

The output signal count, which is a WIDTH-bit counter : `output logic [WIDTH-1:0] count`


Counting Logic:

> count <= count + {{WIDTH-1{1'b0}}, en};

If rst is low, the counter increments by 1 whenever en is high. The increment logic uses a conditional add operation.

>{{WIDTH-1{1'b0}}, en}

This concatenates WIDTH-1 zeros with the value of en, which results in adding 1 to count if en is high (1'b1), or adding 0 if en is low (1'b0).

Behaviour Explanation:

- The always_ff block is triggered on the positive edge of the clock 'posedge clk'.
- When rst is high, the counter is reset to all zeros.
- The counter increments by 1 when en is high. The increment happens only on the rising edge of the clock (posedge clk).
- The counter's width is parameterized, allowing us to choose the bit-width at instantiation time. The default is 8 bits, but this can be overwritten by using a different value when the module is instantiated.


### counter_tb.cpp

```cpp
#include "Vcounter.h"
#include "verilated.h"
#include "verilated_vcd_c.h"

int main(int argc, char **argv, char **env){
    int i;
    int clk;
    
    Verilated::commandArgs(argc, argv);
    //init top verilog instance
    Vcounter* top = new Vcounter;
    //init trace dump
    Verilated::traceEverOn(true);
    VerilatedVcdC* tfp = new VerilatedVcdC;
    top->trace (tfp, 99);
    tfp->open ("counter.vcd");

    top->clk = 1;
    top->rst = 1;
    top->en = 0;

    //run simulation for many clock cycles
    for(i=0; i<300; i++){

        //dump variables into VCD file and toggle clock
        for(clk=0; clk<2; clk++){
            tfp->dump(2*i+clk);     //unit is in ps
            top->clk = !top->clk;
            top->eval (); 
        }
        top->rst = (i<2) | (i==15);
        top->en = (i>4);
        if(Verilated::gotFinish()) exit(0);
    }
    tfp->close();
    exit(0);
}
```

The C++ code simulates a Verilog module 'Vcounter' using Verilator.


Setup:

`Vcounter.h` : Header file for the Vcounter Verilog module.

`verilated.h` : Verilator's main header file.

`verilated_vcd_c.h` : Header for generating VCD (Value Change Dump) files, which are used to record simulation waveforms.

For some reason these are all underlined as errors in VSCode but the simulation run without issue.


Initialization:

>Verilated::commandArgs(argc, argv);

Initializes Verilator with the command-line arguments.

>Vcounter* top = new Vcounter;

Creates an instance of the Vcounter module.

VCD Tracing:

>Verilated::traceEverOn(true); 

Enables VCD tracing.

>VerilatedVcdC* tfp = new VerilatedVcdC; 

Creates a trace object.

>top->trace (tfp, 99); 

Configures tracing for the Vcounter instance with depth 99 (trace detail level).

>tfp->open ("counter.vcd"); 

Opens a VCD file named counter.vcd to store the simulation waveforms.


Simulation Loop:


Initializes the inputs:

`top->clk = 1;` : Clock is initially high.

`top->rst = 1;` : Reset is initially high.

`top->en = 0;` : Enable is initially low.


We have a for loop that simulates 300 clock cycles in which:

`tfp->dump(2*i+clk);` dumps the current state into the VCD file.

The clock is toggled between high and low with `(top->clk = !top->clk;).`

`top->eval();` evaluates the design for the current clock state.

The reset signal is asserted for the first two cycles and on cycle 15. `top->rst = (i<2) | (i==15);`

Enable signal is activated after 4 cycles `top->en = (i > 4);`

`if (Verilated::gotFinish()) exit(0);` : This checks if there is a finish condition that should end the simulation early


In the End:

`tfp->close();` Closes the VCD file.

`exit(0);` Exits the simulation.


Time Scale:

The time axis in the waveform is displayed in picoseconds because of `tfp->dump(2*i+clk);`

`2 * i + clk` is used for the time stamp:

- i is the iteration counter, which increments every two clock edges (i.e. every full clock cycle).

- Multiplying by 2 and adding the clock gives the timing for both the rising and falling edges of the clock.

- Since Verilator’s default time unit is picoseconds, this translates to 1 time unit (1 i) being 2 ps.


### Challenges:

1 - Modify the testbench so that you stop counting for 3 cycles once the counter reaches 0x9, and then resume counting. You may also need to change the stimulus for rst.

Solution:

```cpp
#include "Vcounter.h"
#include "verilated.h"
#include "verilated_vcd_c.h"

int main(int argc, char **argv, char **env){
    int i;
    int clk;
    int pause_cycles = 0;  // Pause cycle tracker
    bool pause = false;   // Boolean value that checks for the pause state

    Verilated::commandArgs(argc, argv);
    //init top verilog instance
    Vcounter* top = new Vcounter;
    //init trace dump
    Verilated::traceEverOn(true);
    VerilatedVcdC* tfp = new VerilatedVcdC;
    top->trace (tfp, 99);
    tfp->open ("counter.vcd");

    top->clk = 1;
    top->rst = 1;
    top->en = 0;

    //run simulation for many clock cycles
    for(i=0; i<300; i++){

        //dump variables into VCD file and toggle clock
        for(clk=0; clk<2; clk++){
            tfp->dump(2*i+clk);  // Back to picoseconds scale
            top->clk = !top->clk;
            top->eval(); 
        }

        // Reset signal control for the first 2 cycles
        top->rst = (i < 2);

        if (top->count == 0x9 && !pause) {
            pause_cycles = 3;  // Initiate 3-cycle pause
            pause = true;     // Set the pause flag
        }

        if (pause_cycles > 0) {
            top->en = 0;       // Stop counting when paused
            pause_cycles--;    
        } else {
            top->en = 1;       // Resume counting once pause is finished
            pause = false;    
        }

        if (Verilated::gotFinish()) exit(0);
    }
    tfp->close();
    exit(0);
}
```
![Simulation](C:\Users\Antoni\Pictures\Screenshots\Screenshot 2024-10-11 185644.png)


2 - The current counter has a synchronous reset. To implement asynchronous reset, you can change line 11 of counter.sv to detect change in rst signal. (See notes.)

Solution:

For an asynchronous reset, the reset condition should be evaluated whenever the reset signal changes, independently of the clock.
This can be achieved by using the or keyword in the sensitivity list of the always_ff block, so that the block is triggered on either the rising edge of the clock or any change in the reset signal.

The always_ff block is now sensitive to both the positive edge of the clock and the positive edge of the reset signal so it will execute if either the clock rises or the reset goes high.

```verilog
module counter #(
    parameter WIDTH = 8
)(
    //interface signals
    input logic clk,
    input logic rst,
    input logic en,
    output logic [WIDTH-1:0] count
);

always_ff @ (posedge clk or posedge rst)  // Detecting rst for asynchronous reset
    if (rst) 
        count <= {WIDTH{1'b0}};  // Asynchronous reset
    else 
        count <= count + {{WIDTH-1{1'b0}}, en};

endmodule
```


