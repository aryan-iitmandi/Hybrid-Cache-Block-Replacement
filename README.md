# Hybrid Cache – Block Replacement Policy

## Overview
Implemented a block replacement policy for a hybrid cache architecture combining 
volatile (SRAM) and non-volatile memory (NVM), simulated using the MacSim 
microarchitectural simulator. The goal was to reduce write amplification and 
extend NVM lifetime while maintaining cache performance.

## Key Contributions
- Designed and implemented a custom block replacement policy targeting 
  write-endurance limitations in NVM-based caches
- Achieved 45% reduction in power consumption compared to traditional 
  SRAM-only cache configurations
- Validated policy correctness across multiple workload traces using MacSim's 
  detailed processor simulation framework
- Analyzed cache behavior under HPC-oriented access patterns to optimize 
  replacement decisions

## Tech Stack
- **Language:** C/C++
- **Simulator:** MacSim (microarchitectural simulation framework)
- **Build System:** SCons
- **OS:** Linux (Ubuntu)

## Requirements
- Linux (Ubuntu recommended)
- GCC 4.4+ (C++11 support)
- SCons: `sudo apt-get install scons`
- zlib: `sudo apt-get install zlib1g-dev`
- Python: `sudo apt-get install python`

## Build & Run
```bash
./build.py
cd bin
./macsim
```
Ensure `params.in` and `trace_file_list` are present in the `bin/` directory.
