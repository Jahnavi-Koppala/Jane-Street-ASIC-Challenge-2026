


## High Level Analysis:

First and foremost I went about viewing puzzle.gds in [TinyTapeout Online GDS Viewer](https://gds-viewer.tinytapeout.com/) and picking the layers and cells/instances apart. 

### Initial Observations:
1. This chip design is not purely combinational. It is sequential since it has 92 D flip-flops (with 12 of 92 belonging to the output generator section).
2. The cells are arranged in clusters. Given the clue that the circuit is physically arranged to hint at its functionality, I assumed that each cluster of cells has it's own function (perhaps a counter? a shift register?).
3. INTERNAL_3 and INTERNAL_7 are not visible to the naked eye in TinyTapeout (On further inspection this turned out to be an easter egg).



VIA_*, INTERNAL_*, decap_*, diode_*, tapvpwrvgnd_* are ignored as they are related to physical implementation / power / reliability rather than the logical function we are trying to understand.


How to load and read a gds file using gdstk: https://heitzmann.github.io/gdstk/gettingstarted.html#
This helped me understand the cells, polygons and labels

GDS Layers Info:
https://skywater-pdk.readthedocs.io/en/main/rules/layers.html

The Example input:
Viewing the example_inputs.vcd in Surfer and counting the number of posedges when enable=1 makes it obvious that the input is serial 121 bit.

## Constructing a Gate-level Netlist from GDS

The cell names use SKY130 standard cells and it's labels have locations of pins relative to the cell. This is enough information to extract a gate-level netlist from the GDS by tracing the metal conductivity along and between the layers. (this problem is much simpler as there's no need to look at the transistor layers)

As the pins are on layer 67 (except I, O[7:0]) and the routing layers on top of it, we will be analysing these metal and via layers (67 to 72).

I wrote an algorithm in python that would print the recovered netlist with synthesized cell names and net(node) names.

1.
2.
3.


NOTE: I have observed that some pins have multiple coordinates. Fortunately, all the duplicate instances of the same pin mapped to the same net.



## SAT Solver:

Result:

The 121-bit input for which success goes high: `0000000101010000100000000000010101010000000000001010000001000001000000100000101000010000000100000010000010010001010000000`

There are more 0s than 1s and there are 22 1s. Between 121 and 22, 11 is a common denominator. That gave me the idea to view the input in windows of 11-bits.

```
00000001010
10000100000
00000001010
10100000000
00001010000
00100000100
00001000001
01000010000
00010000001
00000100100
01010000000
```

Every row and column has 2 1s and no 1s touch each other. This observation combined with the output for success=1 "(* TWO STARS *)" made me realise that this is a Star Battle Board.

# Analyzing the layout

I have verified that this is indeed a start battle board from the layout. 

The identification of the detailed functionality of the layout was based on backward tracing from the success logic, grouping flip-flops by shared fan-in structure, examining their dependence on position signals, deriving the state-transition equations, and finally interpreting the state accepted by the comparator logic. This provided a direct functional explanation from the gate-level circuit rather than relying only on the expected rules of the puzzle.

11 x 2-bit saturating column counters: The column-position counter determines which one of the 11 column counters gets the input. When input=1 the appropriate column counter advances.

11 x 2-bit saturating region counters: Both column-position and region-position counters determine which one of the 11 region counters gets the input. When input=1 the appropriate column counter advances.

MOD-11 Column position counter: Stores the column position of the current input bit. It advances once per enabled serial input bit.

MOD-11 Row position counter: Stores the row position of the current input bit. Unlike the previous counter, this one advances when the column counter reaches its terminal state.

2-bit reusable saturating counter: At the terminal state of the MOD-11 Column position counter checks whether the current row count including the current bit equals exactly 2; resets the two-bit counter for the next row. Since the input arrives row-major, there's no need to use 11 different counters.

Invalid-row sticky flag: Flag is set true when the above counter at terminal column position is not equal to two.

8-bit population counter: Counts the number of 1s. After reading 121-bit input, the total number of 1s is stored in this counter and finally checked if the total is exactly 22. The population counter acts as a useful global consistency check.

12-bit shift register: Stores the previous 12 input bits for adjacency check. (12 bits of history is required as that's the farthest a neighbour can be)

Adjacency-error sticky flag: Flag is set true when the two stars are touching.

End of input sticky flag: Sticky flag indicating that the final cell of the 11×11 board has been consumed. This happens when both MOD-11 Column position counter and MOD-11 Row position counter reach their terminal states.

Final checker: We have 3 DFFs

One is a completion latch; also starts the post-check output phase

alternate result state; it distinguishes a failed adjacency case when the other major count constraints pass

success
    stores the actual fully-valid result

The set condition for success is 


## Solving for Regions


I did it by isolating the 11 two-bit region counters that feed the final region-validity condition. The netlist indeed contains paired resettable state elements for these region counters, and their combined validity signal feeds the final checker. I then reset the circuit and, for each of the 121 board positions, supplied a board containing exactly one star at that position. Whichever region counter incremented tells us which region that cell belongs to. Repeating this for all 121 cells reconstructs the complete region map.

Using rows 1–11 and columns 1–11, with serial input interpreted row-by-row, I get:

        Columns
        1  2  3  4  5  6  7  8  9 10 11

Row  1  D  D  D  D  D  J  J  A  B  B  I
Row  2  D  D  F  D  D  J  A  A  B  B  I
Row  3  D  D  F  J  J  J  J  A  A  B  I
Row  4  D  D  F  J  E  E  E  I  A  A  I
Row  5  F  D  F  J  E  I  I  I  I  I  I
Row  6  F  F  F  J  E  E  E  I  H  H  H
Row  7  J  J  J  J  J  J  E  I  H  K  K
Row  8  J  C  C  C  E  E  E  I  H  K  K
Row  9  J  C  C  G  I  I  I  I  H  K  K
Row 10  J  J  C  G  G  I  I  I  H  H  H
Row 11  J  C  C  G  I  I  I  I  I  I  I


## Easter Eggs:
1. PER ARENAM AD ASTRA: While going through each layer and cell/instance type view in tinytapeout, INTERNAL_3 and INTERNAL_7 where inconspicuous (probably an auxiliary layer). After extracting it's polygons and decoding it's Morse Code, I got PER ARENAM AD ASTRA
2. LEAP SECOND --The date mentioned in example_inputs.vcd file. Sat Dec 31 23:59:60 2016.
3. THE NIGHT SKY AWAITS -- The example input VCD reads
4. Output generator: EMPTY SKY, BIG BANG, TWO"NOT TOUCH

## Tools
Icarus Verilog, z3 solver, TinytapeOut, Surfer


