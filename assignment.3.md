## ECE 565 Programming Assignment Part 3 Fall 2025
We are moving to the ARM ISA for these tasks. You will have to rebuild gem5 to use the ARM target ISA.

For this assignment you should focus on the *MinorCPU* Model located in /src/cpu/minor. The *MinorCPU* currently models a simple pipelined.  (Hint: Use the -h flag with your Python script to find out how to specify the CPU model you use.)
    One caveat of this CPU model is that each pipeline stage/component inherits from an abstracted resource object. As with every resource, there is the contingency of encountering a structural hazard. The occurrence of a structural hazard is dependent on the width of the corresponding stage. Thus in order to model one outstanding instruction per stage, you have to adjust the width of each stage to be 1 (*hint: look inside MinorCPU.py*).
    
A nice overview of MinorCPU can be found here: https://www.gem5.org/documentation/general_docs/cpu_models/minor_cpu

For this part and the following part (split execution stage) you should use the ARM build of gem5. Make sure you are building the ARM version using the command line:

```console
scons-3 USE_HDF5=0 -j `nproc` ./build/ECE565-ARM/gem5.opt
```

1. **Examine CPU types (ARM)**

    There are several different types of CPUs that gem5 supports: atomic, timing, out-of-order, inorder and kvm. Let's talk about the timing and the inorder cpus. The timing CPU (also known as SimpleTimingCPU) executes each arithmetic instruction in a single cycle, but requires multiple cycles for memory accesses. Also, it is not pipelined. So only a single instruction is being worked upon at any time. The inorder cpu (also known as Minor) executes instructions in a pipelined fashion. It has the following pipe stages: fetch1, fetch2, decode and execute.

    Take a look at the file MinorCPU.py. In the definition of MinorFU, the class for functional units, we define two quantities opLat and issueLat. From the comments provided in the file, understand how these two parameters are to be used. Also note the different functional units that are instantiated as defined in class MinorDefaultFUPool.

    Assume that the issueLat and the opLat of the FloatSimdFU can vary from 1 to 6 cycles and that they always sum to 7 cycles. For each decrease in the opLat, we need to pay with a unit increase in issueLat. Which design of the FloatSimd functional unit would you prefer? Provide statistical evidence obtained through simulations of the unannotated daxpy. For these experiments, please use the pre-compiled arm binary for daxpy that's provided at `/home/yara/mithuna2/gem5-Fall2022/daxpy-armv7-binary`. (For all parts of the homework that use the ARM ISA, we are unable to cross-compile on ECN's x86 machines. As such, you **must** use precompiled binaries.)
    
    If you wish - you can use the following new configuration file that provides an example of how to extend the MinorCPU with options for op/issue latency. Other students have reportedly used other approaches instead of inheritance. Some have reported that they successfully copied the files and modified the copies. As long as you can (1) implement a new kind of CPU, and (2) implement the opLat/issueLat varying in the new CPU, you will have met the requirements. You are not bound to a specific method.
    
    ```python
    # -*- coding: utf-8 -*-
    # Copyright (c) 2015 Mark D. Hill and David A. Wood
    # All rights reserved.
    #
    # Redistribution and use in source and binary forms, with or without
    # modification, are permitted provided that the following conditions are
    # met: redistributions of source code must retain the above copyright
    # notice, this list of conditions and the following disclaimer;
    # redistributions in binary form must reproduce the above copyright
    # notice, this list of conditions and the following disclaimer in the
    # documentation and/or other materials provided with the distribution;
    # neither the name of the copyright holders nor the names of its
    # contributors may be used to endorse or promote products derived from
    # this software without specific prior written permission.
    #
    # THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
    # "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
    # LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
    # A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
    # OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
    # SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
    # LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
    # DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
    # THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
    # (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
    # OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
    #
    # Authors: Jason Power, updates by Tim Rogers
    
    """ CPU based on MinorCPU with options for a simple gem5 configuration script

    This file contains a CPU model based on MinorCPU that allows for a few options
    to be tweaked. 
    Specifically, issue latency, op latency, and the functional unit pool.

    See src/cpu/minor/MinorCPU.py for MinorCPU details.

    """

    from m5.objects import MinorCPU, MinorFUPool
    from m5.objects import MinorDefaultIntFU, MinorDefaultIntMulFU
    from m5.objects import MinorDefaultIntDivFU, MinorDefaultFloatSimdFU
    from m5.objects import MinorDefaultMemFU, MinorDefaultFloatSimdFU
    from m5.objects import MinorDefaultMiscFU

    class MyFloatSIMDFU(MinorDefaultFloatSimdFU):

    # From MinorDefaultFloatSimdFU
    # opLat = 6

    # From MinorFU
    # issueLat = 1

        def __init__(self, options=None):
            super(MinorDefaultFloatSimdFU, self).__init__()

            if options and options.fpu_operation_latency:
                self.opLat = options.fpu_operation_latency

            if  options and options.fpu_issue_latency:
                self.issueLat = options.fpu_issue_latency


    class MyFUPool(MinorFUPool):
        def __init__(self, options=None):
            super(MinorFUPool, self).__init__()
            # Copied from src/mem/MinorCPU.py
            self.funcUnits = [MinorDefaultIntFU(), MinorDefaultIntFU(),
                            MinorDefaultIntMulFU(), MinorDefaultIntDivFU(),
                            MinorDefaultMemFU(), MinorDefaultMiscFU(),
                            # My FPU
                            MyFloatSIMDFU(options)]


    class MyMinorCPU(MinorCPU):
        def __init__(self, options=None):
            super(MinorCPU, self).__init__()
            self.executeFuncUnits = MyFUPool(options)

    ```
    
    To simulate workloads with the MinorCPU, you may use the following command (assuming that the daxpy-armv7-binary has been placed in `<gem5root>/part1`: 
    
    ```console
    ./build/ECE565-ARM/gem5.opt configs/example/se.py --cpu-type=MinorCPU --maxinsts=1000000 --l1d_size=64kB --l1i_size=16kB --caches --l2cache -c part1/daxpy-armv7-binary
    ```
    
    The command includes several options which are worth getting familiar with. E.g., options to change the CPU types, options to limit simulation to a given number of instructions, and options to include/configure caches in the simulation.
   

2. **Degrade Branch Prediction (ARM)**

The MinorCPU already implements a few branch predictor modules, including a tournament predictor and a simpler Branch Target Buffer (BTB). The pipeline timing enables you to figure out at the EX stage whether or not the branch prediction was correct. What you need to do is implement an option that will allow you to not only enable/disable the branch predictor, but degrade its accuracy to different levels as well.

Note that the pipeline already handles the squashing of instructions fetched from the wrong path. On a misprediction, the pipeline will initiate calls to update the corresponding branch predictor’s entry with the correct target address.

It would be functionally correct to squash the pending instructions if the predicted target is correct, but it would not be functionally correct to consider the branch correctly predicted when the predicted target does not match the actual branch target, and still consider it to be correct. For this part of the assignment, you need to implement a feature that degrades the branch predictor’s accuracy by treating some correctly predicted branches as incorrect. You will need to generate a random number between 0 and 1, and for a specified accuracy of X%, if the generated number is greater than X%, consider it a misprediction. Note that this "accuracy" is not the branch predictor’s accuracy, but the percentage of it’s original accuracy. That is, 100% would be equal to the default branch predictor’s accuracy, 50% would be half of the original predictor’s accuracy, and 0% would be an always incorrect predictor – it always predicts the wrong thing.

For this, you will need to run the benchmark simulations for those three "accuracies:"

* 100%
* 50%
* 0%

3. **Split Execution Stage (ARM)**

For this part, we want to be able to split up the Execution Unit into two stages. Modern pipelines employ deeper pipelines in order to increase the clock frequency. Instead of having a more complex stage that requires additional cycles, the corresponding stage is split into smaller stages, where each requires fewer cycles. However, as you know there is a trade-off for every design decision. What you need to do is split up the EX stage into EX1 + EX2 accordingly. The simplest way to do this is to create a "dummy" stage in front of the current execute that does nothing but pass the input tot EX stage along.


4. **Submission instructions**
    
The main deliverable of this assignment that will be graded is a report (and not the code). 

Answer the following questions via BrS:

1) Which (Oplat, IssueLat) pair will will produce the highest IPC? Why? (Max 20 words)
2) Is there a case that (Oplat = 1, IusseLat = 6) will perform best? why? (Max 20 words)
3) Any observations from splitting the EX stage (Max 30 words)

Your report (maximum 2 pages) for part 3 should include:
1) bar-graphs with benchmarks on the X-axis and IPC on the Y-axis to show the result of the Branch Prediction Degradation
2) graphs that support your observations (from splitting the EX stage)

What you need to submit in the end:
1) A maximum 4 pages report (part 1 + 2 + 3) that ONLY includes graphs. No text.
2) Answer the short questions listed above via BrS.




