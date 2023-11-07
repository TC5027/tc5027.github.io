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
