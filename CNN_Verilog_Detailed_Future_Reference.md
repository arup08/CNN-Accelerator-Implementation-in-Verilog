# CNN in Verilog — Detailed RTL Future Reference

> **Purpose:** This document is a practical reference for understanding the supplied CNN Verilog implementation at both the **neural-network level** and the **RTL/hardware-timing level**.
>
> The key idea is to keep three things separate while reading the code:
>
> 1. **CNN mathematics** — convolution, pooling, fully connected layers, ReLU, argmax.
> 2. **Data storage** — image buffers, feature-map BRAMs, pooling BRAMs, weight/bias memory.
> 3. **Hardware scheduling** — FSM states, counters, BRAM latency, MAC valid/clear signals, and pipeline alignment.

---

# 1. Complete CNN Architecture

The supplied design follows this high-level flow:

```text
                    INPUT IMAGE
                     32 × 32
                        │
                        ▼
             ┌────────────────────┐
             │  5×5 Convolution   │
             │     4 filters      │
             └─────────┬──────────┘
                       │
                       ▼
                 4 × 28 × 28
                feature maps
                       │
                       ▼
                    ReLU /
                    scaling
                       │
                       ▼
                 4 × 28 × 28
                       │
                       ▼
             ┌────────────────────┐
             │     2×2 Max Pool   │
             └─────────┬──────────┘
                       │
                       ▼
                 4 × 14 × 14
                       │
                       ▼
                  FLATTEN
                       │
                       ▼
                     784
                    values
                       │
                       ▼
             ┌────────────────────┐
             │      FC1           │
             │    784 → 32        │
             └─────────┬──────────┘
                       │
                       ▼
                 32 activations
                       │
                       ▼
             ┌────────────────────┐
             │      FC2           │
             │     32 → 10        │
             └─────────┬──────────┘
                       │
                       ▼
                  10 logits
                       │
                       ▼
                   ARGMAX
                       │
                       ▼
                 CLASS 0…9
```

---

# 2. Tensor Dimensions

| Stage | Dimensions | Number of values |
|---|---:|---:|
| Input | `32 × 32` | 1024 |
| One convolution kernel | `5 × 5` | 25 |
| Number of filters | 4 | 4 |
| Convolution output | `4 × 28 × 28` | 3136 |
| One pooled feature map | `14 × 14` | 196 |
| Four pooled maps | `4 × 14 × 14` | 784 |
| Flattened vector | `784` | 784 |
| FC1 output | `32` | 32 |
| FC2 output | `10` | 10 |
| Final result | one class | 1 |

## Why is convolution output 28×28?

The convolution uses a `5×5` kernel with valid convolution and no padding:

\[
32-5+1=28
\]

Therefore:

\[
32×32 \rightarrow 28×28
\]

With four filters:

\[
28×28×4
\]

---

# 3. Why Pooling Produces 14×14

A `2×2` max-pooling operation with stride 2 reduces each spatial dimension by 2:

\[
28/2=14
\]

Therefore:

\[
4×28×28
\rightarrow
4×14×14
\]

Each feature map has:

\[
14×14=196
\]

values.

There are four feature maps:

\[
4×196=784
\]

This `784` is exactly the number of inputs to FC1.

---

# 4. The Most Important Data-Flow Picture

```text
                 CONVOLUTION
                     │
                     ▼
        ┌─────────────────────────┐
        │ Filter 0: 28×28         │
        │ Filter 1: 28×28         │
        │ Filter 2: 28×28         │
        │ Filter 3: 28×28         │
        └────────────┬────────────┘
                     │
                     ▼
                  POOLING
                     │
                     ▼
        ┌─────────────────────────┐
        │ Pool 0: 14×14 = 196     │
        │ Pool 1: 14×14 = 196     │
        │ Pool 2: 14×14 = 196     │
        │ Pool 3: 14×14 = 196     │
        └────────────┬────────────┘
                     │
                     ▼
                  FLATTEN
                     │
                     ▼
      ┌──────────────────────────────┐
      │ x[0]   ...   x[195]          │ ← pool 0
      │ x[196] ...   x[391]          │ ← pool 1
      │ x[392] ...   x[587]          │ ← pool 2
      │ x[588] ...   x[783]          │ ← pool 3
      └──────────────┬───────────────┘
                     │
                     ▼
                    FC1
                 784 → 32
                     │
                     ▼
               32 activations
                     │
                     ▼
                    FC2
                  32 → 10
                     │
                     ▼
                10 logits
                     │
                     ▼
                  ARGMAX
```

---

# 5. There Is No Physical Flattening Block

A very important RTL concept:

> **The supplied implementation does not need to copy all 784 pooled values into a new 784-element memory just to flatten them.**

Instead, the flattening happens through the **address-generation logic**.

The four pool memories are logically treated as one vector:

```text
Pool BRAM 0
address 0…195
      │
      ├──► flattened x[0…195]

Pool BRAM 1
address 0…195
      │
      ├──► flattened x[196…391]

Pool BRAM 2
address 0…195
      │
      ├──► flattened x[392…587]

Pool BRAM 3
address 0…195
      │
      └──► flattened x[588…783]
```

Therefore:

```text
flattened index
       │
       ▼
which pool BRAM?
       +
local BRAM address
```

---

# 6. CNN State-Machine Overview

The controller can be understood as a sequence of phases:

```text
                 ┌──────────┐
                 │ S_IDLE   │
                 └────┬─────┘
                      │
                      ▼
              ┌──────────────┐
              │ S_LOAD_IMG   │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_CONV_PREP  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_CONV_LOAD  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_CONV_WAIT  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │S_CONV_STORE  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_POOL_CALC  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_POOL_STORE │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_FC1_PREP   │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_FC1_LOAD   │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_FC1_STORE  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_FC2_PREP   │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_FC2_LOAD   │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ S_FC2_STORE  │
              └──────┬───────┘
                     │
                     ▼
                ┌────────┐
                │ S_DONE │
                └────────┘
```

---

# 7. `S_IDLE`

## Purpose

The design waits here before processing a new image.

Conceptually:

```text
                S_IDLE
                   │
          pixel_valid_in?
             /          \
           NO            YES
           │              │
           │              ▼
           │        S_LOAD_IMG
           │
           └────── stay idle
```

The important idea is that the CNN does not begin processing random data. It waits for the input-valid handshake.

---

# 8. `S_LOAD_IMG`

The incoming pixels are written to:

```text
img_buf[0…1023]
```

because:

\[
32×32=1024
\]

The `pixel_count` counter identifies where the next incoming pixel goes.

Conceptually:

```text
pixel 0  → img_buf[0]
pixel 1  → img_buf[1]
pixel 2  → img_buf[2]
...
pixel 1023 → img_buf[1023]
```

After all pixels are stored:

```text
S_LOAD_IMG
      ↓
S_CONV_PREP
```

---

# 9. Image Buffer Addressing

A 2-D image is stored linearly:

```text
2-D:

pixel(0,0) pixel(0,1) ... pixel(0,31)
pixel(1,0) pixel(1,1) ... pixel(1,31)
...
pixel(31,0) ...           pixel(31,31)
```

becomes:

```text
img_buf[0]
img_buf[1]
...
img_buf[31]

img_buf[32]
...
```

The normal mapping is:

\[
address=row×32+col
\]

This is important when understanding convolution addresses.

---

# 10. `S_CONV_PREP`

Before starting a new convolution output, the MAC accumulator must be reset.

Conceptually:

```text
new convolution output
        │
        ▼
clear accumulator
        │
        ▼
start 25 multiply-accumulate operations
```

For every output pixel and filter:

\[
Y_f(r,c)=
\sum_{k_r=0}^{4}
\sum_{k_c=0}^{4}
X(r+k_r,c+k_c)W_f(k_r,k_c)
\]

There are:

\[
5×5=25
\]

multiply-accumulate terms per output.

---

# 11. Why Four MAC Engines?

The design has four MAC datapaths:

```text
             ┌──── MAC 0 ────► Filter 0
             │
             ├──── MAC 1 ────► Filter 1
Input window ┤
             ├──── MAC 2 ────► Filter 2
             │
             └──── MAC 3 ────► Filter 3
```

This matches the four convolution filters.

Each MAC accumulates the dot product for one filter.

This is different from FC1/FC2, where the same MAC datapath is reused.

---

# 12. `S_CONV_LOAD`

This is where the design starts requesting:

- image samples
- convolution weights

The key complication is that the memory does not necessarily return data in the same clock cycle as the address request.

Therefore there are two concepts:

```text
REQUEST
   │
   ▼
memory latency
   │
   ▼
CONSUME
```

The RTL therefore has separate request-side and consume-side counters.

---

# 13. Request Counters vs Consume Counters

The convolution code has variables such as:

```text
req_kr
req_kc
req_filt
```

and:

```text
cons_kr
cons_kc
cons_filt
```

They do NOT represent the same clock event.

### Request counters

Answer:

> "Which data am I asking memory for now?"

### Consume counters

Answer:

> "Which data has arrived and should now be sent to the MAC?"

This distinction is essential for understanding the pipeline.

---

# 14. Why `conv_img_pipe` Exists

A memory pipeline introduces delay.

Conceptually:

```text
Cycle N:
    address = A
       │
       ▼
     BRAM
       │
       │ latency
       ▼
Cycle N+1 / N+2:
    data = memory[A]
```

Therefore the design stores image values in pipeline registers such as:

```text
conv_img_pipe[0]
conv_img_pipe[1]
```

so the correct image sample lines up with the corresponding weight.

---

# 15. Convolution Pipeline

Think of it as an assembly line:

```text
Clock N
┌───────────────────────┐
│ request image/weight  │
└──────────┬────────────┘
           │
Clock N+1  ▼
┌───────────────────────┐
│ memory returns data   │
└──────────┬────────────┘
           │
Clock N+2  ▼
┌───────────────────────┐
│ feed MAC              │
└──────────┬────────────┘
           │
           ▼
       accumulator
```

This is why the code contains timing counters that may look unrelated to the CNN formula.

---

# 16. `mac_vld`

`mac_vld` tells the MAC:

> "The operands currently presented to you are valid. Perform the MAC operation."

Conceptually:

```text
mac_a = valid image value
mac_b = valid weight
mac_vld = 1
             │
             ▼
         MAC performs
             │
             ▼
     accumulator updated
```

If `mac_vld=0`, the current cycle should not be interpreted as a valid multiplication.

---

# 17. `mac_clr`

`mac_clr` clears the accumulator.

This is necessary before starting a new independent dot product.

For example:

```text
Output A:
    clear
    + product 0
    + product 1
    ...
    + product 24

Output B:
    clear
    + product 0
    + product 1
    ...
```

Without clearing, output B would incorrectly include output A.

---

# 18. `mac_acc`

`mac_acc` is the running sum:

\[
MAC =
a_0b_0+a_1b_1+a_2b_2+\cdots
\]

At an intermediate point:

```text
mac_acc
   =
product0
+ product1
+ product2
+ ...
```

It is intentionally wider than the original operands to reduce overflow risk during accumulation.

---

# 19. `S_CONV_WAIT`

After the last convolution product is launched/consumed, the controller must allow the pipeline and MAC result to settle.

Therefore a separate wait state exists.

The mental model is:

```text
last input/weight
       │
       ▼
pipeline still contains data
       │
       ▼
S_CONV_WAIT
       │
       ▼
final accumulator result
```

Do not interpret this state as "doing nothing." It is mainly a **timing/drain state**.

---

# 20. `S_CONV_STORE`

The convolution result is combined with its bias.

Conceptually:

\[
Z=MAC+b
\]

Then the activation/scaling path is applied.

The overall flow is approximately:

```text
MAC result
    │
    ▼
+ convolution bias
    │
    ▼
ReLU
    │
    ▼
scaling / right shift
    │
    ▼
feature-map BRAM
```

The resulting value is stored for later pooling.

---

# 21. ReLU

ReLU is:

\[
ReLU(x)=
\begin{cases}
x,&x>0\\
0,&x\leq0
\end{cases}
\]

Hardware interpretation:

```text
negative?
 /       \
YES       NO
 │         │
 ▼         ▼
 0         x
```

This is implemented using signed arithmetic.

---

# 22. Why `>>6`?

The RTL defines:

```verilog
RELU_SHIFT = 6
```

A right shift by 6 corresponds approximately to:

\[
x>>6 \approx \frac{x}{64}
\]

It is a **fixed-point scaling operation**.

It is not another neural-network layer.

The conceptual path is:

```text
wide signed MAC result
          │
          ▼
        bias
          │
          ▼
        ReLU
          │
          ▼
         >>6
          │
          ▼
       8-bit value
```

The shift is useful because accumulated fixed-point values can be much larger than the desired activation representation.

---

# 23. Convolution Output Memory

The four convolution filters produce four feature maps.

Conceptually:

```text
conv feature BRAM 0 → 28×28
conv feature BRAM 1 → 28×28
conv feature BRAM 2 → 28×28
conv feature BRAM 3 → 28×28
```

Each map contains:

\[
28×28=784
\]

values.

These memories are later read by the pooling stage.

---

# 24. `S_POOL_CALC`

Pooling processes a `2×2` window.

For example:

```text
a b
c d
```

The maximum is:

\[
max(a,b,c,d)
\]

The RTL does not need four comparators simultaneously. It can maintain a running maximum:

```text
pool_max = a

pool_max = max(pool_max,b)

pool_max = max(pool_max,c)

pool_max = max(pool_max,d)
```

---

# 25. Pooling Counters

Important pooling variables include:

```text
pool_row
pool_col
pool_filt
pool_sub_r
pool_sub_c
```

Think of them as two nested coordinate systems.

### Output coordinate

```text
pool_row
pool_col
```

identifies where the pooled result will be stored.

### Window coordinate

```text
pool_sub_r
pool_sub_c
```

identifies which of the four values inside the `2×2` window is currently being examined.

---

# 26. Pooling Picture

Suppose the convolution map is:

```text
┌─────┬─────┬─────┬─────┐
│ a   │ b   │ e   │ f   │
│ c   │ d   │ g   │ h   │
├─────┼─────┼─────┼─────┤
│ i   │ j   │ ... │ ... │
│ k   │ l   │ ... │ ... │
└─────┴─────┴─────┴─────┘
```

The first pooling window is:

```text
a b
c d
```

and produces:

```text
max(a,b,c,d)
```

The next window is:

```text
e f
g h
```

because stride = 2.

---

# 27. `S_POOL_STORE`

Once the four values have been examined:

```text
pool_max
    │
    ▼
pool BRAM
```

The address corresponds to the pooled output position.

Each pool BRAM stores:

\[
14×14=196
\]

values.

So:

```text
pool[0] → 196 values
pool[1] → 196 values
pool[2] → 196 values
pool[3] → 196 values
```

---

# 28. Transition to FC1

After pooling finishes:

```text
4 × 14 × 14
        │
        ▼
     784 values
        │
        ▼
      FC1
```

There is an important conceptual transition here:

### Before FC1

Data is spatial:

```text
map → row → column
```

### Inside FC1

Data is treated as a vector:

```text
x[0], x[1], ..., x[783]
```

---

# 29. FC1 — The Mathematical Operation

FC1 has:

```text
784 inputs
32 output neurons
```

For neuron `j`:

\[
S_j=\sum_{i=0}^{783}x_iW_{j,i}
\]

Then:

\[
Z_j=S_j+b_j
\]

Then:

\[
A_j=ReLU(Z_j)>>6
\]

Therefore:

```text
784 inputs
      │
      ├──× W[0][i] → sum → bias → ReLU → >>6 → fc1_act[0]
      ├──× W[1][i] → sum → bias → ReLU → >>6 → fc1_act[1]
      ├──× W[2][i] → sum → bias → ReLU → >>6 → fc1_act[2]
      │
      │             ...
      │
      └──× W[31][i]→ sum → bias → ReLU → >>6 → fc1_act[31]
```

---

# 30. FC1 Does NOT Need 32 MAC Units

A hardware-efficient implementation can reuse one MAC:

```text
             ┌────────────────┐
784 inputs ─►│                │
784 weights ─►│    MAC 0      │
             └───────┬────────┘
                     │
                     ▼
                  neuron 0
                     │
                     ▼
                  neuron 1
                     │
                     ▼
                    ...
                     │
                     ▼
                  neuron 31
```

This costs more clock cycles but saves hardware.

The supplied implementation follows this resource-sharing idea for FC1.

---

# 31. `fc1_neuron`

```text
fc1_neuron = current FC1 output neuron
```

Range:

```text
0…31
```

Example:

```text
fc1_neuron = 7
```

means:

> The MAC is currently calculating neuron 7.

---

# 32. `fc1_input`

```text
fc1_input = current input element being multiplied
```

Range:

```text
0…783
```

Example:

```text
fc1_neuron = 7
fc1_input  = 250
```

means:

> Calculate neuron 7 using flattened input 250.

The corresponding mathematical term is:

\[
x_{250}W_{7,250}
\]

---

# 33. FC1 Weight Memory Layout

The FC1 weights are arranged conceptually as:

```text
Neuron 0:
W[0][0]   ... W[0][783]

Neuron 1:
W[1][0]   ... W[1][783]

...

Neuron 31:
W[31][0]  ... W[31][783]
```

The address calculation is:

```verilog
FC1_W_BASE + fc1_neuron*784 + fc1_input
```

For example:

```text
neuron = 2
input  = 10
```

gives:

```text
FC1_W_BASE + 2×784 + 10
```

---

# 34. FC1 Flattened Input Mapping

The four pool BRAMs are mapped to the 784-element FC1 input.

```text
fc1_input 0…195
       ↓
pool 0

fc1_input 196…391
       ↓
pool 1

fc1_input 392…587
       ↓
pool 2

fc1_input 588…783
       ↓
pool 3
```

Local addresses are:

```text
pool 0: fc1_input
pool 1: fc1_input - 196
pool 2: fc1_input - 392
pool 3: fc1_input - 588
```

This is the **logical flattening mechanism**.

---

# 35. Why `fc1_input - 2`?

This is a very important timing detail.

The code uses delayed indexing such as:

```verilog
fc1_input - 2
```

The subtraction is NOT part of:

\[
784\rightarrow32
\]

It compensates for memory/pipeline latency.

Think:

```text
Cycle N:
request x[i] and W[i]

        │
        ▼
   BRAM latency

Cycle N+1:
data moving through pipeline

        │
        ▼

Cycle N+2:
x[i] and W[i] are available to MAC
```

So if the current request counter says `i+2`, the data being consumed may correspond to `i`.

Therefore:

```text
current counter - 2
```

is used to recover the original logical input index.

---

# 36. FC1 Timing Example

```text
Logical index:

i = 0
i = 1
i = 2
i = 3
...

Request stream:

W0/X0 → W1/X1 → W2/X2 → W3/X3
              │
              │ pipeline delay
              ▼
MAC:

       W0/X0 → W1/X1 → W2/X2 → W3/X3
```

The exact number of delay cycles is determined by the RTL memory/MAC pipeline. In the supplied code, the FC1 data selection accounts for this with the `-2` offset.

---

# 37. `S_FC1_PREP`

This state initializes the FC1 calculation.

Conceptually:

```text
fc1_neuron = 0
fc1_input  = 0
MAC = 0
```

Then the first FC1 neuron begins.

---

# 38. `S_FC1_LOAD`

This state repeatedly performs the equivalent of:

```text
read flattened input
read corresponding weight
wait for latency
send input × weight to MAC
```

For neuron `j`:

```text
fc1_input = 0
fc1_input = 1
fc1_input = 2
...
fc1_input = 783
```

After the final input:

```text
MAC contains approximately:

Σ x[i] × W[j][i]
```

---

# 39. `S_FC1_STORE`

Once the dot product is complete:

```text
MAC
 │
 ▼
read bias[j]
 │
 ▼
MAC + bias
 │
 ▼
ReLU
 │
 ▼
>>6
 │
 ▼
fc1_act[j]
```

Then:

```text
fc1_neuron++
```

and the next neuron begins.

---

# 40. `fc1_act`

The array:

```verilog
fc1_act[0:31]
```

stores the output of FC1.

After FC1 completes:

```text
fc1_act[0]
fc1_act[1]
...
fc1_act[31]
```

are all ready.

This is the complete input vector for FC2.

---

# 41. FC1 to FC2 Picture

```text
                  POOL
                   │
                   ▼
             784 flattened values
                   │
                   ▼
          ┌─────────────────┐
          │       FC1       │
          │                 │
          │ 784 inputs      │
          │      ↓          │
          │ 32 neurons      │
          └────────┬────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ fc1_act[0]           │
        │ fc1_act[1]           │
        │ ...                  │
        │ fc1_act[31]          │
        └──────────┬───────────┘
                   │
                   ▼
               32 values
                   │
                   ▼
          ┌─────────────────┐
          │       FC2       │
          │                 │
          │ 32 inputs       │
          │      ↓          │
          │ 10 classes      │
          └────────┬────────┘
                   │
                   ▼
           10 signed logits
                   │
                   ▼
                ARGMAX
```

---

# 42. FC2 — Mathematical Operation

FC2 has:

```text
32 inputs
10 outputs
```

For class `k`:

\[
L_k=
\sum_{i=0}^{31}
A_iW_{k,i}+b_k
\]

where:

```text
A_i = fc1_act[i]
```

So:

```text
32 activations
       │
       ▼
weighted sum
       │
       ▼
+ class bias
       │
       ▼
class logit
```

---

# 43. `fc2_class`

```text
fc2_class = current output class
```

Range:

```text
0…9
```

Example:

```text
fc2_class = 4
```

means:

> The hardware is calculating the score for class 4.

---

# 44. `fc2_input`

```text
fc2_input = current FC1 activation
```

Range:

```text
0…31
```

Example:

```text
fc2_class = 4
fc2_input = 17
```

means:

> Calculate class 4 using `fc1_act[17]` and weight `W[4][17]`.

---

# 45. FC2 Weight Address

The conceptual layout is:

```text
Class 0:
W[0][0] … W[0][31]

Class 1:
W[1][0] … W[1][31]

...

Class 9:
W[9][0] … W[9][31]
```

The address is:

```verilog
FC2_W_BASE + fc2_class*32 + fc2_input
```

---

# 46. FC2 Hardware Reuse

The same MAC can be reused:

```text
Class 0
  │
  ├── 32 products
  ▼
logit 0

Class 1
  │
  ├── 32 products
  ▼
logit 1

...

Class 9
  │
  ├── 32 products
  ▼
logit 9
```

This avoids needing:

```text
10 × 32
```

parallel multipliers.

---

# 47. FC2 Pipeline Alignment

The FC2 code also contains an offset such as:

```verilog
fc1_act[fc2_input-2]
```

Again:

> The `-2` is a pipeline/memory alignment correction.

It means the current control counter and the data currently available at the MAC are not describing exactly the same clock/request index.

Conceptually:

```text
fc2_input request
       │
       ▼
weight address
       │
       ▼
BRAM latency
       │
       ▼
weight data
       │
       ├──────────┐
       │          │
       ▼          ▼
 delayed       weight
 fc1_act       data
       │          │
       └────┬─────┘
            ▼
           MAC
```

---

# 48. FC2 Has No ReLU

This is a major difference.

## FC1

```text
MAC
 ↓
bias
 ↓
ReLU
 ↓
>>6
 ↓
fc1_act
```

## FC2

```text
MAC
 ↓
bias
 ↓
fc2_logit
```

The final FC2 value must remain signed because the classifier compares scores.

---

# 49. Why FC2 Logits Are Signed

Suppose the outputs are:

```text
class 0 = -20
class 1 = 35
class 2 = -7
class 3 = 18
...
```

The correct class is determined by the largest numerical value:

```text
35
```

Therefore negative values must be preserved.

If FC2 applied ReLU:

```text
-20 → 0
-7  → 0
```

the original score information would be lost.

---

# 50. `fc2_logit`

The RTL stores ten scores:

```text
fc2_logit[0]
fc2_logit[1]
...
fc2_logit[9]
```

Think of them as:

```text
        CLASSIFICATION SCORES

class 0 ─────► logit[0]
class 1 ─────► logit[1]
class 2 ─────► logit[2]
...
class 9 ─────► logit[9]
```

The prediction is:

\[
\operatorname{argmax}_k L_k
\]

---

# 51. Argmax

Argmax means:

> Find the index of the largest value.

Example:

```text
Class       Logit

0             12
1             31
2             -4
3             74   ← maximum
4             18
5             42
6              5
7             20
8             11
9             63
```

Result:

```text
class = 3
```

---

# 52. `argmax_val`

This register stores:

```text
largest score encountered so far
```

After class 0:

```text
argmax_val = logit[0]
```

After class 1:

```text
argmax_val = max(logit[0], logit[1])
```

Continue until class 9.

---

# 53. `argmax_class`

This register stores the class corresponding to `argmax_val`.

Example:

```text
argmax_val   = 74
argmax_class = 3
```

means:

```text
highest score = 74
class = 3
```

---

# 54. Why Class 9 Needs Special Treatment

A sequential RTL implementation commonly has a subtle issue with nonblocking assignments.

For example:

```verilog
fc2_logit[9] <= new_logit;
```

does not make the new value immediately visible everywhere else in that same clocked block.

Therefore the code explicitly compares the newly calculated class-9 result with the running maximum.

Conceptually:

```text
new class-9 logit
       │
       ▼
compare with argmax_val
       │
   ┌───┴───┐
   ▼       ▼
greater   smaller
   │       │
   ▼       ▼
class 9   old winner
```

---

# 55. `mac_pipe_cnt`

This variable is best understood as a:

> **datapath timing/alignment counter**

It is not simply "the number of MACs performed."

It helps determine when pipeline events should happen.

For example:

```text
request data
     │
     ▼
pipeline delay
     │
     ▼
consume data
     │
     ▼
finish MAC
     │
     ▼
read bias
     │
     ▼
store result
```

`mac_pipe_cnt` helps the FSM distinguish these timing points.

---

# 56. Why a Counter Is Needed

Suppose a bias is requested from BRAM:

```text
Cycle N:
    bias address issued

Cycle N+1:
    memory still processing

Cycle N+2:
    bias data available
```

The FSM must not use the bias at cycle N.

A timing counter allows the controller to wait for the correct cycle.

---

# 57. MAC Datapath

The conceptual MAC block is:

```text
             operand A
                 │
                 ▼
              ┌─────┐
              │ MUL │
              └──┬──┘
                 │
                 ▼
             product
                 │
                 ▼
             ┌─────┐
             │ ADD │◄──── accumulator
             └──┬──┘
                 │
                 ▼
             accumulator
```

With control:

```text
mac_clr
   │
   ▼
clear accumulator

mac_vld
   │
   ▼
accept current A/B
```

---

# 58. MAC Signals — Quick Reference

| Signal | Meaning |
|---|---|
| `mac_a` | First multiplication operand |
| `mac_b` | Second multiplication operand |
| `mac_vld` | Current operands are valid |
| `mac_clr` | Clear accumulator |
| `mac_acc` | Current accumulated result |
| `mac_pipe_cnt` | Timing/alignment counter |

---

# 59. Main CNN Variables — Quick Reference

## Input

| Variable | Role |
|---|---|
| `pixel_in` | Incoming pixel |
| `pixel_valid_in` | Incoming pixel valid |
| `pixel_count` | Input-pixel position |
| `img_buf` | Stored input image |

## Convolution

| Variable | Role |
|---|---|
| `conv_out_row` | Output row |
| `conv_out_col` | Output column |
| `conv_filt` | Current filter |
| `req_kr` | Requested kernel row |
| `req_kc` | Requested kernel column |
| `req_filt` | Requested filter |
| `cons_kr` | Consumed kernel row |
| `cons_kc` | Consumed kernel column |
| `cons_filt` | Consumed filter |
| `conv_img_pipe` | Delayed image data |
| `conv_store_phase` | Result-store timing phase |

## Pooling

| Variable | Role |
|---|---|
| `pool_row` | Pool output row |
| `pool_col` | Pool output column |
| `pool_filt` | Current feature map |
| `pool_sub_r` | Row within 2×2 window |
| `pool_sub_c` | Column within 2×2 window |
| `pool_max` | Running maximum |
| `pool_addr_issued` | Pool memory timing/request tracking |

## FC1

| Variable | Role |
|---|---|
| `fc1_neuron` | Current FC1 neuron, 0…31 |
| `fc1_input` | Current flattened input, 0…783 |
| `fc1_act` | 32 FC1 activations |

## FC2

| Variable | Role |
|---|---|
| `fc2_class` | Current class, 0…9 |
| `fc2_input` | Current FC2 input, 0…31 |
| `fc2_logit` | Ten output scores |

## Classification

| Variable | Role |
|---|---|
| `argmax_val` | Largest score so far |
| `argmax_class` | Class associated with largest score |
| `result_reg` | Final predicted class |
| `result_valid_reg` | Indicates prediction is valid |

---

# 60. Weight Memory Organization

The supplied constants organize the weight memory into regions:

```verilog
CONV_W_BASE = 15'd0;
CONV_B_BASE = 15'd100;

FC1_W_BASE  = 15'd104;
FC1_B_BASE  = 15'd25192;

FC2_W_BASE  = 15'd25224;
FC2_B_BASE  = 15'd25544;
```

Conceptually:

```text
WEIGHT BRAM

0
│
├── CONV WEIGHTS
│   4 × 5 × 5 = 100
│
100
├── CONV BIASES
│   4
│
104
├── FC1 WEIGHTS
│   32 × 784 = 25,088
│
25,192
├── FC1 BIASES
│   32
│
25,224
├── FC2 WEIGHTS
│   10 × 32 = 320
│
25,544
├── FC2 BIASES
│   10
│
25,554
```

---

# 61. Why Memory Is Organized This Way

Using one large weight memory is convenient for FPGA implementation.

Instead of having separate memories:

```text
conv_weight_memory
fc1_weight_memory
fc2_weight_memory
...
```

the design can use one address space:

```text
             Weight BRAM
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
       CONV      FC1        FC2
      region    region     region
```

The base address identifies the layer.

---

# 62. FC1 Memory Size

FC1 has:

\[
784
\]

inputs per neuron and:

\[
32
\]

neurons.

Therefore:

\[
784×32=25,088
\]

weights.

Then 32 biases are stored after them.

---

# 63. FC2 Memory Size

FC2 has:

\[
32
\]

inputs and:

\[
10
\]

classes.

Therefore:

\[
32×10=320
\]

weights.

Then 10 biases are stored after them.

---

# 64. Full FC1 Hardware Picture

```text
               POOL BRAMs
                   │
                   │ logical flattening
                   ▼
          ┌─────────────────┐
          │ 784-element     │
          │ logical vector  │
          └────────┬────────┘
                   │
                   │ x[i]
                   ▼
              ┌─────────┐
              │         │
Weight BRAM ─►│  MAC 0  │
              │         │
              └────┬────┘
                   │
                   ▼
              Σ x[i]W[j][i]
                   │
                   ▼
                  +bias
                   │
                   ▼
                 ReLU
                   │
                   ▼
                  >>6
                   │
                   ▼
              fc1_act[j]
                   │
                   │ j++
                   ▼
              next neuron
```

---

# 65. Full FC2 Hardware Picture

```text
              fc1_act[0…31]
                    │
                    ▼
             ┌─────────────┐
             │             │
Weight BRAM ─►   MAC 0     │
             │             │
             └──────┬──────┘
                    │
                    ▼
             Σ A[i]W[k][i]
                    │
                    ▼
                  +bias
                    │
                    ▼
               fc2_logit[k]
                    │
                    ▼
                 argmax
```

---

# 66. Why FC1 Takes Much More Work Than FC2

FC1:

\[
32×784=25,088
\]

multiplications.

FC2:

\[
10×32=320
\]

multiplications.

Therefore FC1 has:

\[
25,088/320 \approx 78.4
\]

times as many weight multiplications.

This is why FC1 dominates the fully connected computation.

---

# 67. Hardware Resource Sharing

The design trades:

```text
hardware resources
```

for:

```text
clock cycles
```

Instead of implementing all FC1 neurons in parallel:

```text
32 MACs
```

it can reuse one:

```text
MAC 0
```

Similarly, FC2 reuses the same MAC.

This is a common FPGA design strategy when resource utilization matters.

---

# 68. Why the RTL Looks More Complicated Than the CNN

The CNN equation is simple:

\[
y=\sum x_iw_i+b
\]

But hardware must answer:

```text
Where is x_i stored?
Where is w_i stored?
When does memory return x_i?
When does memory return w_i?
When can the MAC consume them?
When should the accumulator clear?
When should the bias be read?
When should the output be stored?
```

That is why the Verilog has:

```text
FSM states
counters
pipeline registers
valid signals
address registers
```

---

# 69. Reading the RTL as a Scheduler

A very useful mental model is:

> The FSM is a scheduler for the neural-network operations.

For example:

```text
S_FC1_PREP
    ↓
prepare

S_FC1_LOAD
    ↓
request + consume MAC data

S_FC1_STORE
    ↓
bias + activation + store
```

The states divide a mathematically continuous operation into clock-accurate hardware phases.

---

# 70. `S_FC1_LOAD` — What Is Actually Happening

For each input:

```text
1. Determine flattened input index.
2. Determine which pool BRAM contains it.
3. Generate pool BRAM address.
4. Generate FC1 weight address.
5. Wait for memory pipeline.
6. Pair returned input and weight.
7. Assert MAC valid.
8. Accumulate product.
9. Increment fc1_input.
10. Repeat.
```

At the end:

```text
mac_acc ≈ Σ x[i]W[j][i]
```

---

# 71. `S_FC1_STORE` — What Is Actually Happening

After all 784 products:

```text
1. Request/read bias.
2. Add bias to MAC result.
3. Apply signed ReLU.
4. Shift right by 6.
5. Store into fc1_act[fc1_neuron].
6. Advance to next neuron.
```

---

# 72. `S_FC2_LOAD` — What Is Actually Happening

For each class:

```text
1. Start with fc2_input = 0.
2. Read fc1_act[0] and W[class][0].
3. Accumulate.
4. Move to input 1.
5. Continue to input 31.
6. Final MAC = Σ fc1_act[i]W[class][i].
```

---

# 73. `S_FC2_STORE` — What Is Actually Happening

```text
MAC result
    │
    ▼
read class bias
    │
    ▼
MAC + bias
    │
    ▼
fc2_logit[class]
    │
    ▼
argmax comparison
```

If this is the final class:

```text
class 9
```

the final comparison produces the classification result.

---

# 74. Complete FC1 → FC2 Example

Assume the pooled vector is:

```text
x[0] ... x[783]
```

For FC1 neuron 2:

```text
fc1_neuron = 2

MAC =
x[0]   × W[2][0]
+x[1]  × W[2][1]
+x[2]  × W[2][2]
...
+x[783]× W[2][783]
```

Then:

```text
+ bias[2]
→ ReLU
→ >>6
→ fc1_act[2]
```

Now FC2 class 7:

```text
fc2_class = 7

MAC =
fc1_act[0]  × W[7][0]
+fc1_act[1] × W[7][1]
...
+fc1_act[31]× W[7][31]
```

Then:

```text
+ bias[7]
→ fc2_logit[7]
```

Finally:

```text
compare fc2_logit[7]
against current argmax
```

---

# 75. Important Difference Between Counters

This table is worth remembering:

| Counter | Question it answers |
|---|---|
| `pixel_count` | Which input pixel? |
| `conv_out_row` | Which convolution output row? |
| `conv_out_col` | Which convolution output column? |
| `conv_filt` | Which convolution filter? |
| `req_kr` | Which kernel row is being requested? |
| `req_kc` | Which kernel column is being requested? |
| `cons_kr` | Which kernel row is being consumed? |
| `cons_kc` | Which kernel column is being consumed? |
| `pool_row` | Which pooled output row? |
| `pool_col` | Which pooled output column? |
| `pool_filt` | Which feature map? |
| `pool_sub_r` | Which row of 2×2 window? |
| `pool_sub_c` | Which column of 2×2 window? |
| `fc1_neuron` | Which FC1 output? |
| `fc1_input` | Which FC1 input? |
| `fc2_class` | Which FC2 output/class? |
| `fc2_input` | Which FC2 input? |
| `mac_pipe_cnt` | Which timing/pipeline phase? |

---

# 76. A Simple Rule for Understanding Any Counter

Whenever you see a counter in the RTL, ask:

> **"What object does this counter index?"**

For example:

```text
fc1_neuron → neurons
fc1_input  → inputs
fc2_class  → classes
fc2_input  → FC1 activations
pool_row   → pooled rows
pool_col   → pooled columns
```

This is much easier than memorizing every line of Verilog.

---

# 77. Pipeline Counters Are Different

For something like:

```text
mac_pipe_cnt
```

do not ask:

> "Which neural-network object does this index?"

Instead ask:

> "Which clock/timing phase am I in?"

That distinction is extremely useful.

---

# 78. Signed Arithmetic

The design uses wider signed values for MAC accumulation and logits.

A useful representation is:

```text
input activation
       │
       ▼
8-bit value
       │
       ×
8-bit weight
       │
       ▼
wider signed product
       │
       ▼
wide signed accumulator
       │
       ▼
bias
       │
       ▼
activation/scaling
```

The wide accumulator is necessary because adding hundreds of products can require many more bits than one input or weight.

---

# 79. Why FC1 Activation Is 8-bit

After:

```text
MAC
+
bias
+
ReLU
+
>>6
```

the result is stored into:

```verilog
fc1_act[0:31]
```

with an 8-bit representation.

This makes FC2 cheaper because its input values are compact.

---

# 80. Why FC2 Output Remains Wider

FC2 produces signed classification scores.

Keeping a wider signed representation makes it possible to preserve:

```text
positive values
negative values
relative differences between classes
```

for the final argmax.

---

# 81. Debugging With Vivado Waveforms

When debugging this design in Vivado, do not inspect every signal at once.

Use groups.

## Group 1 — FSM

```text
state
```

## Group 2 — FC1

```text
fc1_neuron
fc1_input
fc1_act
```

## Group 3 — FC2

```text
fc2_class
fc2_input
fc2_logit
```

## Group 4 — MAC

```text
mac_a
mac_b
mac_vld
mac_clr
mac_acc
mac_pipe_cnt
```

## Group 5 — Memory

```text
weight address
weight data
pool addresses
pool data
```

---

# 82. Best Way to Debug FC1

At a selected clock cycle, ask five questions:

```text
1. Which neuron?
   fc1_neuron

2. Which input?
   fc1_input

3. Which weight address?
   FC1_W_BASE + fc1_neuron×784 + fc1_input

4. Which pool memory?
   determined by flattened index

5. Is this request or consumed data?
   account for pipeline delay
```

Then inspect:

```text
mac_a
mac_b
mac_vld
mac_acc
```

---

# 83. Best Way to Debug FC2

Ask:

```text
1. Which class?
   fc2_class

2. Which input?
   fc2_input

3. Which weight?
   FC2_W_BASE + fc2_class×32 + fc2_input

4. Which activation?
   fc1_act[fc2_input - pipeline delay]

5. Is MAC accumulating?
   mac_vld
```

After input 31:

```text
MAC + bias
→ fc2_logit[class]
```

---

# 84. Common Confusions

## Confusion 1: "Where is flatten?"

Answer:

```text
There is no large flattening operation.
```

The flattened vector is created logically by address sequencing.

---

## Confusion 2: "Why is there `-2`?"

Answer:

```text
Pipeline alignment.
```

It compensates for delayed BRAM/data availability.

---

## Confusion 3: "Why `>>6`?"

Answer:

```text
Fixed-point scaling by approximately 1/64.
```

It is not a CNN layer.

---

## Confusion 4: "Why only one MAC for FC1?"

Answer:

```text
Hardware resource sharing.
```

One MAC is reused for 32 neurons.

---

## Confusion 5: "Why no ReLU in FC2?"

Answer:

```text
FC2 produces signed class logits.
```

The largest signed score is selected.

---

# 85. CNN Mathematics vs RTL

This is perhaps the most important comparison.

| CNN concept | RTL implementation |
|---|---|
| Input image | `img_buf` |
| Convolution | MAC engines + image/weight addressing |
| Bias | bias memory + addition |
| ReLU | signed comparison |
| Scaling | right shift |
| Feature maps | feature BRAMs |
| Max pooling | `pool_max` comparisons |
| Flatten | address mapping |
| FC1 | repeated MAC operations |
| FC1 activations | `fc1_act` |
| FC2 | repeated MAC operations |
| Class logits | `fc2_logit` |
| Argmax | `argmax_val` + `argmax_class` |
| Prediction | `result_reg` |

---

# 86. Overall Hardware Architecture

```text
                         ┌───────────────┐
pixel_in ───────────────►│   IMG BUFFER  │
                         └───────┬───────┘
                                 │
                                 ▼
                       ┌─────────────────┐
                       │  CONVOLUTION    │
                       │  4 MAC engines  │
                       └────────┬────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ FEATURE BRAMs   │
                       └────────┬────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   MAX POOL      │
                       └────────┬────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   POOL BRAMs    │
                       │ 4 × 196 values  │
                       └────────┬────────┘
                                │
                                ▼
                         logical flatten
                                │
                                ▼
                       ┌─────────────────┐
                       │      FC1        │
                       │   MAC reused    │
                       │    784 → 32     │
                       └────────┬────────┘
                                │
                                ▼
                           fc1_act[31:0]
                                │
                                ▼
                       ┌─────────────────┐
                       │      FC2        │
                       │   MAC reused    │
                       │     32 → 10     │
                       └────────┬────────┘
                                │
                                ▼
                           fc2_logit[9:0]
                                │
                                ▼
                       ┌─────────────────┐
                       │     ARGMAX      │
                       └────────┬────────┘
                                │
                                ▼
                         predicted class
```

---

# 87. Control Architecture

The datapath is controlled by the FSM:

```text
                    ┌──────────────┐
                    │     FSM      │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
       Counters         Memories           MAC
          │                │                │
          └────────────────┼────────────────┘
                           │
                           ▼
                      CNN result
```

The FSM determines:

```text
what operation
when operation
which address
which counter
which MAC control
```

---

# 88. High-Level Clock Sequence

A single image can be viewed as:

```text
CLOCKS
──────────────────────────────────────────────►

LOAD IMAGE
████████

CONV PREP
        █

CONV LOAD
         █████████████████████████

CONV WAIT
                                 ██

CONV STORE
                                   ███████

POOL
                                          ███████████

FC1 PREP
                                                     █

FC1
                                                      ███████████████████████████████████

FC2 PREP
                                                                                       █

FC2
                                                                                        ███████████

DONE
                                                                                                   ███
```

The exact cycle counts depend on the RTL's memory and MAC implementation, but the phase ordering is the key concept.

---

# 89. The Most Important Mental Model

When reading this CNN Verilog, think:

```text
                    CNN
                     │
                     ▼
              mathematical task
                     │
                     ▼
              hardware schedule
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     memory        MAC          FSM
        │            │            │
        └────────────┼────────────┘
                     ▼
                 result
```

The neural network tells us **what** must be calculated.

The RTL tells us **when and where** each calculation happens.

---

# 90. One Complete End-to-End Example

Suppose the network receives one `32×32` image.

### Step 1 — Load

```text
1024 pixels
     ↓
img_buf[0…1023]
```

### Step 2 — Convolution

For each of four filters:

```text
5×5 window
     ↓
25 products
     ↓
MAC
     ↓
bias
     ↓
ReLU / scaling
     ↓
feature map
```

Output:

```text
4×28×28
```

### Step 3 — Pool

Each feature map:

```text
28×28
  ↓
2×2 max pool
  ↓
14×14
```

Output:

```text
4×14×14
```

### Step 4 — Flatten

Logical sequence:

```text
pool0 → 0…195
pool1 → 196…391
pool2 → 392…587
pool3 → 588…783
```

### Step 5 — FC1

For each of 32 neurons:

```text
784 multiplications
       ↓
     MAC
       ↓
    + bias
       ↓
     ReLU
       ↓
      >>6
       ↓
fc1_act[j]
```

### Step 6 — FC2

For each of 10 classes:

```text
32 multiplications
      ↓
     MAC
      ↓
   + bias
      ↓
fc2_logit[k]
```

### Step 7 — Argmax

```text
10 logits
   ↓
largest score
   ↓
class index
```

---

# 91. Final Cheat Sheet

```text
INPUT
32×32
= 1024

CONV
5×5 × 4 filters
= 4×28×28

POOL
2×2 stride 2
= 4×14×14

FLATTEN
4×14×14
= 784

FC1
784 → 32
= 25,088 weight multiplications

FC1 processing
MAC + bias + ReLU + >>6
→ 32 activations

FC2
32 → 10
= 320 weight multiplications

FC2 processing
MAC + bias
→ 10 signed logits

ARGMAX
10 logits
→ predicted class
```

---

# 92. Most Important Variables to Remember

If you forget everything else, remember these:

```text
pixel_count
    → input pixel position

conv_out_row / conv_out_col
    → convolution output position

conv_filt
    → convolution filter

pool_row / pool_col
    → pooling output position

pool_filt
    → pooling feature map

fc1_neuron
    → FC1 output neuron

fc1_input
    → FC1 input index, 0…783

fc1_act[]
    → 32 FC1 outputs

fc2_class
    → FC2 class, 0…9

fc2_input
    → FC2 input index, 0…31

fc2_logit[]
    → 10 class scores

argmax_val
    → largest score so far

argmax_class
    → class of largest score

mac_acc
    → running MAC sum

mac_vld
    → MAC operands valid

mac_clr
    → clear MAC accumulator

mac_pipe_cnt
    → pipeline/timing control
```

---

# 93. Final One-Paragraph Summary

The supplied Verilog CNN stores the input image in a buffer, computes four 5×5 convolution feature maps using MAC engines, applies bias/activation/scaling, performs 2×2 max pooling, and stores four 14×14 pooled maps. There is no separate physical flattening stage: the four pooled memories are accessed sequentially as a logical 784-element vector. FC1 then reuses a MAC datapath to calculate 32 neurons, each requiring 784 multiply-accumulate operations, followed by bias, ReLU, and `>>6` scaling. The resulting 32 `fc1_act` values become the inputs to FC2. FC2 reuses the MAC again to calculate ten class logits, each from 32 inputs plus a bias. Finally, `argmax_val` and `argmax_class` track the largest logit, producing the final predicted class. The FSM, counters, pipeline registers, and timing counters are primarily there to make all of these mathematical operations happen correctly despite synchronous memory and datapath latency.

---

# 94. Quick Reference: If You Open the Verilog Again

Read the code in this order:

```text
1. Parameters / dimensions
        ↓
2. Weight-memory base addresses
        ↓
3. MAC module
        ↓
4. BRAM declarations
        ↓
5. FSM state definitions
        ↓
6. Input loading
        ↓
7. Convolution
        ↓
8. Pooling
        ↓
9. FC1
        ↓
10. FC2
        ↓
11. Argmax
        ↓
12. Output/result logic
```

When you reach FC1, immediately identify:

```text
fc1_neuron
fc1_input
fc1_act
```

When you reach FC2, immediately identify:

```text
fc2_class
fc2_input
fc2_logit
```

When you see `-2`, think:

```text
PIPELINE ALIGNMENT
```

When you see `>>6`, think:

```text
FIXED-POINT SCALING ≈ /64
```

When you see `mac_pipe_cnt`, think:

```text
TIMING / PIPELINE CONTROL
```

When you see `pool_max`, think:

```text
RUNNING MAX OF 2×2 WINDOW
```

When you see `argmax_val`, think:

```text
LARGEST CLASS SCORE SO FAR
```

That mental mapping is enough to navigate most of the RTL without getting lost in the clock-by-clock details.
