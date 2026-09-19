# `default_attribute_value`

| | |
|---|---|
| **When** | The workflow framework asks for the default attribute of the instance - when building object lists, attachments and anywhere an object should "name itself". |
| **Out** | RESULT  a DATA REFERENCE to the value |

**THE ONE LINE THAT IS THE SAME IN EVERY IMPLEMENTATION**

Of all 26 hooks, this is the only one that was filled in every production implementation examined - and every time with exactly the same line.

WITH c06t-OBJTEXT: LEAVE IT EMPTY

The framework then supplies the document key itself, from the right instance, BEFORE this hook. That is why the line below only sets RESULT if it is still empty.

**WHY GET REFERENCE OF AND NOT AN ASSIGNMENT**

```abap
       RESULT is REF TO DATA. The framework dereferences it later.
       A local variable would be long gone by then - hence the
       reference to the instance data of the framework, which lives
       the whole time.
```

**PITFALL** If you return a reference to a METHOD-LOCAL variable, you get no error, but garbled data later. Hence the reference to

**MS_INSTANCES.**

**CAUTION MS_INSTANCES**

MS_INSTANCES is class-wide and overwritten by every FIND_BY_LPOR. With several work items in one session it returns the key of the object created last - one more reason for c06t-OBJTEXT.

## The code

```abap
IF result IS INITIAL.
  GET REFERENCE OF /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid INTO result.
ENDIF.
```
