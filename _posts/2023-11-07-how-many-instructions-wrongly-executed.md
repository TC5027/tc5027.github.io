---
layout: post
title: "How many instructions can be wrongly executed ?"
date: 2023-11-07
---

Not so long ago, I heard about a strange behaviour of the branch predictor unit in AMD CPUs, making it easy to wrongly execute instructions but up to how many ?

## About CPU

Let's consider this gcc output.

```
foo:
cmp edi,10
jle .L2
lea eax, [rdi+6]
ret
.L2:
cmp edi, 5
mov eax, 5
cmovle eax, edi
ret
```

The theory behind CPUs is that there is a pipeline system for processing the instructions. What happens if we have edi=9 ?
```
fetch     |cmp edi, 10|jle .L2    |lea eax, [rdi+6]|ret             <-not the following instruction
decode    |x          |cmp edi, 10|jle .L2         |lea eax, [rdi+6]<-not the following instruction
execute   |x          |x          |cmp edi, 10     |jle .L2
write-back|x          |x          |x               |cmp edi, 10
```

Due to the ```jle .L2``` instruction a jump is performed and in our sketch, fetching instructions linearly yields to this situation where the incoming instructions are not the one expected due to the lack of anticipation of the jump, a pipeline hazard.

To solve this, there is a dedicated component inside CPUs, the BPU. It takes as input an instruction pointer and outputs a prediction on if there is a branch at this address, and if yes where does it leads. This happens very early at the fetch state.

## The experiment

```
mov $CACHE_LINE_ADDR, %rsi
clflush (%rsi)
sfence
lfence
xor %rdi, %rdi
jz END_LABEL
nop
...
nop
mov (%rsi), %rax
END_LABEL:

measure CACHE_LINE_ADDR access time
```

This program helps us identify if the branch predictor failed in its prediction. What happens is we flush a line out of cache, the ```xor %rdi, %rdi``` sets ZF flag to 1 which implies the conditional jump to go to ```END_LABEL```.

* If the branch predictor predicted correctly then ```mov (%rsi), %rax``` is not executed and the access time is high because it was previously evicted from cache
* If the branch predictor predicted wrongly + there is not too many ```nop```, then ```mov (%rsi), %rax``` is executed and then backrolled, but not its side effect of caching, so the access time is lower

Considering our sketch pipeline this could never happen as the backroll should happen before the execution step but in reality there is more agresive speculative executions in CPUs to get the best perf.

Adding ```nop```s, we can deduce how many instructions can be wrongly executed due to branch misprection before being backrolled.

## AMD BTB

Inside BPU there is a cache called Branch Target Buffer, storing the previously taken branches' destination. For AMD's BTB the behaviour is if there is no entry in the BTB for a conditional branch, it will always be predicted as not taken.

The idea is to first fill the whole BTB with unconditional branches so that there is no entry left, and then run our experiment, the branch will be predicted as not taken and the access time at the end should be low.

## Result

I rented a vps with AMD EPYC 7543 and using [KTF](https://github.com/KernelTestFramework/ktf) (simple kernel suited for this kind of experiments) I managed to execute wrongly 8 instructions with a misprediction rate of 99%
