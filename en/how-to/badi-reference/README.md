# Reference: all 26 BAdI methods

Anyone building a conFLOW workflow starts by creating a class that
implements `/C09/CFL_IF_BADI_0101`. The interface requires
**26 methods**. Their names are all similar, their signatures say
little about what they are for, and you don't need most of them.

The problem is that at the start you don't know which ones.

This chapter is a **complete, activatable example class** with all
26 hooks. Each one has the same header: when it runs, what it
receives, what it may change, and whether you need it at all.

**9 hooks are intentionally empty.** That is not something left
for someone to fill in - it is the answer. Their header says what they
are meant for and how to tell whether your own case belongs there. An
example that fills all 26 methods would be more convenient to
read and wrong in substance.

## What you find here

| | |
|---|---|
| `ZCL_CFL_WORKFLOW_00900` | the reference class - all 26 hooks, one background step, eleven helpers |
| `ZCL_CFL_CONST_00900` | the constants class - every key from Customizing in one place |
| `ZCL_CFL_GET_ACTORS` | agent determination, central instead of in the workflow class |

All three are syntax-checked against a real system. There are no
dependencies outside the conFLOW standard.

## How to read it

From the beginning, if you are building your first workflow - the hooks
are arranged in six groups along the process, not in the order of the
interface.

Via the [map](basics/map.md), if you are looking for a specific hook.

## The example process

An **approval process** - the case conFLOW is built for in nine out of
ten projects. Here it runs on an SAP standard document, so the data
part doesn't take any attention and the focus stays on the workflow.
See [The process](basics/process.md).
