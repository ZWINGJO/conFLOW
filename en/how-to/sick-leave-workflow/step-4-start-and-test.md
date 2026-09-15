# Step 4: Start and test the workflow

The workflow is configured and the BAdI class is implemented. In this last step you set up the trigger, fill the container with business data and test the whole process.

---

## 4.1 Overview

<figure><img src="../../.gitbook/assets/folie32.png" alt="Starting the workflow overview"><figcaption><p>From the trigger to the running workflow</p></figcaption></figure>

---

## 4.2 Trigger options

There are several ways to trigger a conFLOW workflow. The choice depends on the use case:

| Option | When to use | How |
| --- | --- | --- |
| **Report / program** | Demos, tests, manual triggering | Call `start_workflow_int( )` directly |
| **Application BAdI** | Automatically when a document changes | Check and start in the save exit of the application object |
| **Status management** | Automatically on a status change | SAP status management (BSVW) raises a BOR event, the SWETYPV type linkage triggers the workflow |

{% hint style="warning" %}
**Protection against multiple starts:** `start_workflow_int( )` does **not** check whether a workflow is already running for the same object. Call `check_open_workflow( )` first — otherwise parallel instances are created for the same document.
{% endhint %}

<figure><img src="../../.gitbook/assets/folie33.png" alt="Trigger option 1"><figcaption><p>Option A: start the workflow from a report or program</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/folie34.png" alt="Trigger option 2"><figcaption><p>Option B: start the workflow from an application BAdI</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/folie35.png" alt="Trigger option 3"><figcaption><p>Option C: workflow via status management and BOR event</p></figcaption></figure>

---

## 4.3 Fill the container

The **conFLOW container** (`/C09/CFL_S04`) stores any business data as key-value pairs. It is written with the framework methods:

```abap
" write a value
/c09/cfl_cl_workflow_0101=>set_attribut_value(
  iv_element = 'BELEGNR'
  iv_id      = ls_cfl_s03-id
  it_value   = VALUE #( ( lv_belegnr ) ) ).

" read a value
DATA(lt_val) = /c09/cfl_cl_workflow_0101=>get_attribut_value(
                 iv_element = 'BELEGNR'
                 iv_id      = ls_cfl_s03-id ).
```

{% hint style="info" %}
**Best practice:** put attribute names as constants into a separate class `ZCL_CFL_CONST_<nnnnn>`. This class is shared by the BAdI class, the background steps and, if applicable, the UI — one source for all attribute names.
{% endhint %}

<figure><img src="../../.gitbook/assets/folie36.png" alt="Container"><figcaption><p>SET_ATTRIBUT_VALUE and GET_ATTRIBUT_VALUE: the conFLOW container</p></figcaption></figure>

---

## 4.4 Follow-up logic for parallel steps

With parallel work items, several agents decide. The BAdI method `GET_AFTER_EXECUTION_WORKITEM` is called after each individual decision and receives the context: which agent decided how, and whether further work items are still open.

This lets you, for example, calculate a summary after the last decision or trigger a follow-up action.

<figure><img src="../../.gitbook/assets/folie37.png" alt="GET_AFTER_EXECUTION_WORKITEM"><figcaption><p>Follow-up logic after parallel decisions</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/folie38.png" alt="Parallel tasks result"><figcaption><p>Merging and further processing</p></figcaption></figure>

---

## 4.5 Checklist: first test

Before the workflow goes live, check the following:

| Check | Where |
| --- | --- |
| Workflow instance was created | `/C09/CFL_S01`: a row with your `wf_definition` and `instid` |
| Work items were created | `/C09/CFL_S03`: rows with the `id` from `S01` |
| Agent is correct | SWIA: open the work item, check the agent |
| Container is filled | `/C09/CFL_S04`: attributes with the `id` from `S01` |
| Decision works | Open the work item in SBWP or Fiori My Inbox and decide |
| Next step is correct | After the decision: next step in `S01-gen_stat` |
| Workflow ends | `S01-wf_end = 'X'` after the last step |

{% hint style="success" %}
**The workflow runs.** From here on, the process is functionally complete. Further refinements — better texts, differentiated agent determination, Fiori display, a conMOBILE app — are enhancements, not prerequisites.
{% endhint %}
