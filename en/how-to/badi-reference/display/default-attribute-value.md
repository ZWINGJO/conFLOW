# `default_attribute_value`

| | |
|---|---|
| **When** | The workflow framework asks for the default attribute of the instance - when building object lists, attachments and anywhere an object should "name itself". |
| **Out** | RESULT  a DATA REFERENCE to the value |

**THE ONE LINE THAT IS THE SAME IN EVERY IMPLEMENTATION**

Of all 26 hooks, this is the only one that was filled in every production implementation examined - and every time with exactly the same line. Copy it and you have it right.

**WHY GET REFERENCE OF AND NOT AN ASSIGNMENT**

```abap
       RESULT is REF TO DATA. The framework dereferences it later.
       A local variable would be long gone by then - hence the
       reference to the instance data of the framework, which lives
       the whole time.
```

**PITFALL** If you return a reference to a METHOD-LOCAL variable, you get no error, but garbled data later. Always reference MS_INSTANCES-INSTANCE->MS_DATA.

## The code

```abap
GET REFERENCE OF /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid INTO result.
```
