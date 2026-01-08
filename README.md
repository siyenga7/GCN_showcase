**GCN Module**

**1. Introduction and architecture, high level block diagram**

![gcn_architecture](https://github.com/user-attachments/assets/fbfe5d65-88d1-49fe-84cc-476ccc634cc8)

**2. Initial Design Decisions**

**2.1 Parallel Multipliers**

Design Decision: I chose to implement 32 parallel multipliers for matrix
multiplication. This decision was based on the need to balance
performance with area and power consumption. By increasing the number of
parallel multipliers, matrix multiplication efficiency can be improved
significantly. However, more multipliers also increase area and power
consumption, so a tradeoff had to be considered.

Reasoning: After evaluating the computation load and the number of
elements in the matrices, I decided on 32 multipliers to achieve a
balanced throughput while optimizing for area efficiency. This allows
for parallel computation while not excessively increasing chip area.
This will also be explained in next section

**2.2 Number of Matrix Blocks (Block-wise approach)**

Design Decision: The matrix multiplication operation was broken into
blocks of 32 elements i.e 3 blocks for multiplying 96 elements, which
allows for easier scaling, pipelining, and more efficient use of the
hardware resources. This block-based approach uses a weight stationary
method for multiplication where 1 weight column is stored in scratchpad
and 6 rows of feature matrix input are multiplied with it. This approach
reduces off chip memory access which is very essential when considering
ASIC design. Block-wise approach also makes sure that the on chip
memory(scratch-pad) is small considering the cost of manufacturing on
chip memory is way higher than off chip memory.

Reasoning: The 32 block size was chosen after evaluating the optimal
trade-off between parallelism and resource constraints. This size
strikes a balance between not oversizing the memory requirement and
achieving high throughput for larger matrices.  
  
For example when the block size was varied from 96 to 6 (factors of 96),
significant reduction in area and power was observed. I tested all block
sizes and decide to choose 32 because it met the 100ns latency
constraint easily with significant overhead. This was optimal size
considering it required balance amount of pipelining. Smaller blocks can
be chosen but they come with downside of excessive pipelining to meet
latency constraints

**2.3 COO matrix design decision**

I did not opt for conversion of COO matrix to adjacency matrix because I
found an efficient way to use COO data and transformed matrix to
directly write into fm_wm_adj matrix with the use of only adders. This
decision was taken because conversion to adjacency matrix made no sense
since it would be a binary sparse matrix, instead an algorithm was
chosen to directly use COO data for creating final fm_wm_adj matrix.

Algorithm:- Since the Adjacency matrix is completely binary and sparse,
The final multiplied output will use only values from FM_WM memory. For
example Adjacency matrix row is \[00010\], when this row is multiplied
to FM_WM memory it gives output as whatever data FM_WM memory had at the
transposed location. Another example with Adj row being \[000101\] now
for this we will need an adder too. If COO data is 1 & 2 it means the
value of adj matrix row 0 and column 1 will be 1 and also at column 0
and row 1 it will be 1. This gives us enough information to choose fm_wm
memory element, the final step is to use an adder that takes present
value of output memory and adds data from fm_wm memory from new COO data
it received.  
  
An important change from milestone 1 is that I pipelined my COO fsm a
bit more to reduce the number of cycles required to complete
computation, this helped me increase Clock period further while being
under latency constraints

**2.4 Data Flow and Control**

Design Decision: two control FSMs (finite state machines), one for
transform block and another for COO block were implemented to manage
data flow between the feature matrix, weight matrix, and the multiplier
modules. It controls when to start, read, and store results, keeping the
data in sync. The FSM of transform block has 4 main stages and a few
final stages which imitate the 4 stages but are only used at the end
when all counters reset back to 0 but there's still more data to be
fetched multiplied and stored. These stages are required due to
pipelining and a 2 cycle delay introduced by sending the read address
output and getting a new data in.

Reasoning: The FSM approach was chosen due to its simplicity in handling
state transitions and ease of implementing sequencing and
synchronization. It also provides better control over timing and ensures
the module\'s correct functionality. Another key design decision was to
reduce switching activity of signals in every stage of FSM. This was
achieved using smart pipelining decisions, which helped keeping signals
at either 0 or 1 for most of the time of computation, this drastically
reduced switching activity which plays an important role in power
analysis.  
  
Clock Gating was added at synthesis stage using Design compiler---where
the clock signal to certain portions of the circuit is disabled when not
needed. Since the clock is a major source of switching activity (and
hence dynamic power consumption), turning it off selectively reduces
unnecessary toggling and saves power.

2.5 Total latency and simulation of post APR Verilog netlist  

<img width="548" alt="image" src="https://github.com/user-attachments/assets/01d6ab05-6baa-43a5-a1ca-346b19d476f4" />

**Tot_latency = 0.000096602 ms** at **Clock Frequency of 833.33 MHz**

**Earlier design had a latency of 82ns which was increased to
96.6ns(still \<100ns) -- This decision was taken to squeeze out more
power optimisation from the design by reducing clock frequency but not
crossing the 100ns requirement**

**2.6 Power Analysis**

Design Decision: Timing and power analysis were performed using Innovus
power analysis option to ensure that the design meets both speed and
power efficiency constraints. The design was optimized for low power
consumption without sacrificing performance. Clock gating using design
compiler was added which reduced the power consumption by 3 times.

**Total Power: 1.079mW**

**Total power reduced from 1.24mW to 1.079mW (13% decrease in power)
compared to earlier design.**

<img width="395" alt="image" src="https://github.com/user-attachments/assets/4ef9ea69-c259-4f23-bafa-8da657461dac" />


**2.7 Area**

Standard cells + Filler cells: **23518.123** um\^2

<img width="482" alt="image" src="https://github.com/user-attachments/assets/0b645cce-4bb6-46c4-8c41-f7b796eda45b" />


<img width="320" alt="image" src="https://github.com/user-attachments/assets/06d2a609-8441-4020-b5b5-665f32c49211" />


X dim -- 154.512 um, Y dim -- 152.28 um

X and Y dimension can easily be calculated from floorplan information
which excludes power rings and gives accurate core size information.

**2.8 Innovus Density - 74.184%**

<img width="524" alt="image" src="https://github.com/user-attachments/assets/e2b68c01-7099-460b-894b-36257c1759bb" />


**2.9 Number of Gates**

GCN Gates = 23465**

Cells = 9242**

**2.10 DRC and LVS on calibre**

<img width="524" alt="image" src="https://github.com/user-attachments/assets/d817579d-8e15-4c0f-a6ce-195d02a7168c" />


**Waivable DRC violations present in the design**

<img width="524" alt="image" src="https://github.com/user-attachments/assets/8af44ccf-a75a-48a7-9732-a5686c057d69" />


**These short circuit warnings for LVS make sense considering that
during synthesis the synthesis tool warned that read address has 13 pins
out of which most were shorted to logic 0, this happens because read
address changes only from 512 to 517(for feature) and 0 to 2(for weight)
which means most of the pins are at logic 0 through out the computation.
Since read address is an output port, the synthesis tool did not remove
the logic 0 pins( don't touch attribute applied)**

**2.11 Worst Hold and Setup paths**

<img width="524" alt="image" src="https://github.com/user-attachments/assets/7552d39f-c538-45db-8423-7552663cc45d" />


**As it is evident that the slack is exactly 0, after post route when
the optDesign command was run it managed to fix this path which had
almost negligible negative slack, the slack was so negligible that
innovus showed it as '-0.000' but after running the optimisation it
fixed this.**

<img width="524" alt="image" src="https://github.com/user-attachments/assets/d788e931-4fb8-4823-9d83-788899f64e95" />


**2.12 Geometry and Connectivity**

<img width="524" alt="image" src="https://github.com/user-attachments/assets/9c161d09-d610-4d09-80f0-ff14c4dcb941" />


<img width="524" alt="image" src="https://github.com/user-attachments/assets/97ca8f53-b17d-48fb-8e07-b8dca3287551" />


**3 Future Improvements and comments on current design**

This is a highly optimised design, with minimal switching activity,
smaller on chip memory, clock gating and highly efficient pipelining.
The entire goal of this project was to reduce power as much as possible.

I had two parameters that I could change freely for power optimisation

- Alpha(Activity factor) -- If signals toggle very less this parameter
  will have a very low value , drastically reducing power

- Frequency -- finding the right balance between meeting latency and
  keeping power low by choosing the right frequency since Freq is
  directly proportional to dynamic power

Throughout the designing process one thing that I noticed was bottleneck
at FM_WM memory. The COO block cannot start before the FM_WM memory is
complete. If the COO data is stored in a memory we can pipeline both
fsms in such a way that it utilises the vector multiplication output as
soon as it is generated without having to store it in a larger memory
block(Since COO data is 6\*2\*3 registers while FM_WM memory is 6\*3\*5
registers)  
  
Another optimisation would be addition of zero skipping technique, I
noticed that feature and weight matrix both have a lot of zero values,
this technique will help reduce power further by skipping multiplication
completely for 0 valued inputs.
