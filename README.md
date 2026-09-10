# FPGA-Based CNN Accelerator

A **Verilog-based CNN accelerator** designed for **FPGA hardware** to perform image classification efficiently.

## Overview

The CNN processes a **32×32 grayscale image** through:

**Input → Convolution → ReLU → Max Pooling → Flatten → FC1 → FC2 → Argmax → Output**

## Architecture

- **Convolution:** 4 parallel **5×5 filters**, producing **4×28×28 feature maps**
- **Max Pooling:** **2×2** pooling reduces maps to **4×14×14**
- **Flatten:** Converts the pooled maps into **784 logical inputs**
- **Fully Connected:** **FC1: 784→32**, **FC2: 32→10**
- **Classification:** Argmax selects the predicted class

## Hardware Design

- **BRAM-based** storage for input, feature maps, pooled data, weights, and biases
- **MAC-based** computation for convolution and fully connected layers
- **FSM-based control** for sequencing and memory addressing
- **Fixed-point arithmetic** for hardware-efficient computation
- **Pipeline-aware control** for correct timing and data alignment

## Goal

Develop a **resource-efficient FPGA CNN accelerator** capable of end-to-end image classification with predictable latency and efficient on-chip data reuse.

## Technologies

**Verilog HDL | FPGA | Vivado | BRAM | MAC | FSM | Fixed-Point Arithmetic | RTL Design**

## Future Scope

- Higher computation parallelism
- Deeper CNN architectures
- Quantization and lower-bit computation
- Real-time image/video classification
- Edge-AI and embedded vision applications
