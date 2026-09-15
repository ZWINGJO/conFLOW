# Why this example

## The problem it solves

A conFLOW workflow starts with a class that implements
`/C09/CFL_IF_BADI_0101`. When you create it, the development
environment generates 26 empty method bodies.

Then you face three questions, and the signature answers none of them:

1. **Which hooks does my workflow need?** The names only help so
   much - `GET_AFTER_EXECUTION` and `GET_AFTER_EXECUTION_WORKITEM`
   sound the same and aren't.
2. **What may I change in a hook?** Some parameters are `CHANGING` and
   are ignored anyway. Others only take effect in one of the two UIs.
3. **What happens if I leave one out?** Usually nothing visible - and
   that is exactly the problem. A missing hook rarely causes an error.
   It causes a UI that shows something other than intended.

## What the standard provides

The package `/C09/CONFLOW_BACKEND_0101` contains
**`/C09/CFL_CL_BADI_0101`** - an example implementation of the BAdI. It
contains fragments from real projects: agent determination via the
organizational structure, archive attachments, an example for
`GET_STATUS_DYNAMIC`.

Most of the fragments are **commented out** and come from different
installations. They are useful for ideas; as a template to copy they
don't work, because they refer to objects that only existed in the
respective project.

This reference adds what is missing: a version that is **complete,
activatable and consistent**, with a statement on every hook - including
the ones that stay empty.

## What is different here

**One continuous process instead of individual examples.** All hooks
refer to the same document and the same container attributes. You can
read from top to bottom.

**Standard dependencies only.** The class calls
`/C09/CFL_CL_WORKFLOW_0101` and `/C09/CFL_CL_HELPER_0101` - both are
part of the delivery. There are no custom helper classes except the two
that come with it.

**Syntax-checked against a real system.** Not "should work", but
compiled.

**The empty hooks are content.** For six of 26, the right answer is
"leave it empty". Why is stated with each one.

## What is not covered here

**Customizing.** This reference describes the class, not the tables
`/C09/CFL_C01` to `C10`. Without maintained Customizing no workflow
runs, however good the class is - the
[installation order](../reference/installation.md) says what belongs
together.
