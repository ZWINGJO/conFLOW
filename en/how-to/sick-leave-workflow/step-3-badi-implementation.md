# Step 3: BAdI implementation

The Customizing defines the process flow — the BAdI class fills it with business logic. In this step you create the BAdI implementation and implement the most important hooks.

---

## 3.1 Overview

Every conFLOW workflow has exactly one BAdI class that implements the interface `/C09/CFL_IF_BADI_0101`. The class is restricted to the workflow number by a **filter** — so it only applies to "its" workflow.

{% hint style="info" %}
**All hooks always exist.** The interface defines numerous methods. Leave the ones you don't need empty — just do nothing in them. The framework does not check whether a hook contains code.
{% endhint %}

<figure><img src="../../.gitbook/assets/folie19.png" alt="BAdI overview"><figcaption><p>The BAdI class: one class per workflow definition</p></figcaption></figure>

---

## 3.2 Create the enhancement implementation

In transaction **SE80** (or ADT), create an enhancement implementation for the enhancement spot `/C09/CFL_ES_BADI_0101`. The decisive step is the **filter**: set `wf_definition = '<your number>'` so that the class only applies to your workflow.

<figure><img src="../../.gitbook/assets/folie20-1.png" alt="Enhancement implementation"><figcaption><p>SE80: BAdI implementation with a filter on the workflow number</p></figcaption></figure>

---

## 3.3 Dynamic step determination

If the next step should not come from Customizing (`/C09/CFL_C02`) but be calculated at runtime, set the type of the approval step to **"BADI"**. conFLOW then calls the BAdI method `GET_STATUS_DYNAMIC` and expects the next `gen_stat` as the return value.

{% hint style="warning" %}
**Caution:** When `GET_STATUS_DYNAMIC` is active, the transitions in `/C09/CFL_C02` are **ignored** for this step. All routing logic then lives in ABAP code. Use this mode only if the decision really has to be dynamic.
{% endhint %}

<figure><img src="../../.gitbook/assets/folie21-1.png" alt="Dynamic step determination"><figcaption><p>Type "BADI": dynamic step determination at runtime</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/folie22-1.png" alt="Dynamic step determination detail"><figcaption><p>GET_STATUS_DYNAMIC: the code determines the next step</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/folie23-1.png" alt="Dynamic step determination example"><figcaption><p>Example: branching based on document data</p></figcaption></figure>

---

## 3.4 Parallel steps

For a parallel step, assign **several agent keys** (`gen_stat_user`) to one approval step in Customizing. conFLOW creates a work item for each, and the workflow waits until all of them have decided.

{% hint style="info" %}
**Veto, first decision or majority** are now set on the step without code, with the column *Decision rule* in Approval steps. Details: [Technical documentation, parallel agent paths](../../technical-documentation.md#parallel-agent-paths). The BAdI is only needed for rules of your own.
{% endhint %}

<figure><img src="../../.gitbook/assets/folie24.png" alt="Parallel steps"><figcaption><p>Parallel processing: several agent keys per step</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/folie25.png" alt="Parallel steps detail"><figcaption><p>Parallel work items in the inbox</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/folie26.png" alt="Parallel steps result"><figcaption><p>Merging the parallel results</p></figcaption></figure>

---

## 3.5 Dynamic agent determination with GET\_ACTORS

If `user_badi = 'X'` is set in Customizing (`/C09/CFL_C03`), conFLOW calls the BAdI method `GET_ACTORS`. There you determine the agent at runtime — for example based on organizational unit, document data or a PFCG role.

{% hint style="warning" %}
**Mind the format:** actor strings must always start with an object type prefix: `US` for user, `S` for position, `AC` for role. A bare user name without a prefix is ignored.
{% endhint %}

<figure><img src="../../.gitbook/assets/folie27.png" alt="GET_ACTORS"><figcaption><p>GET_ACTORS: determine agents dynamically at runtime</p></figcaption></figure>

---

## 3.6 Application log (SLG1)

Messages from background steps (`ET_BAPIRET2`) are logged automatically in the **application log** (SLG1). You configure object and subobject in the node **General parameters** (`/C09/CFL_C08`). The messages can be evaluated there without a custom Z table.

<figure><img src="../../.gitbook/assets/folie28.png" alt="Application log"><figcaption><p>Messages in the application log (SLG1)</p></figcaption></figure>

---

## 3.7 Replace mail placeholders with GET\_DATASOURCE\_MAIL

The SO10 texts for email notifications contain placeholders (`§{...}`). The method `GET_DATASOURCE_MAIL` supplies the replacement values — typically document data read from the conFLOW container or from the SAP document.

{% hint style="info" %}
**Reference implementation:** the standard class `/C09/CFL_CL_BADI_0101` contains an example of `GET_DATASOURCE_MAIL` that can serve as a starting point.
{% endhint %}

<figure><img src="../../.gitbook/assets/folie29.png" alt="GET_DATASOURCE_MAIL"><figcaption><p>Replace placeholders in SO10 texts dynamically</p></figcaption></figure>

---

## 3.8 Navigate to the business object with EXECUTE\_DEFAULT\_METHOD

When the agent wants to jump from the work item to the business object (double-click in SAP GUI), conFLOW calls the method `EXECUTE_DEFAULT_METHOD`. There you set the parameter and call the transaction:

```abap
SET PARAMETER ID 'ANR' FIELD lv_belegnr.
CALL TRANSACTION 'VA03' AND SKIP FIRST SCREEN.
```

<figure><img src="../../.gitbook/assets/folie30.png" alt="EXECUTE_DEFAULT_METHOD"><figcaption><p>Navigation from the work item to the SAP transaction</p></figcaption></figure>

---

## 3.9 Further options

The BAdI interface offers many more hooks — for most workflows the ones shown above are enough. Further hooks such as `GET_AFTER_EXECUTION_WORKITEM`, `GET_BEFORE_DECISION_WORKITEM` or `GET_OBJECT_INFO` enable follow-up logic, button control and adjustments to the Fiori display.

<figure><img src="../../.gitbook/assets/folie31.png" alt="Further options"><figcaption><p>Further BAdI hooks for special requirements</p></figcaption></figure>
