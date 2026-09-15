# BC master data distribution

## The requirement

Master data — material masters, conditions, organizational units — must be kept consistent in a distributed system landscape. Changes in the leading system must be checked, approved and distributed to the receiving systems. Distribution errors must be detected and handled.

## What conFLOW does here

The workflow models the approval and distribution process: a master data change is entered, the responsible business department approves it, and a background step performs the technical distribution. If distribution fails, conFLOW creates a work item for the Basis administrator with the relevant error information.

Messages from background steps are automatically logged in the application log (SLG1) and can be evaluated there — without a custom Z table.

### User story

**Daily routine:** **Roman** maintains master data and rules for business processes. He usually receives the changes from various business departments by email. They are often extensive changes that he works through as a team. When everyone is done with a change, it is approved and the objects are distributed automatically to all SAP landscapes.

**conFLOW for master data distribution:** **Roman** receives an email saying that a different customs tariff number should be used for a certain material group. He creates a work order for himself, which also starts the workflow, with himself as the first agent. If he couldn't finish it today, he would pass the workflow on to his colleague (including all the notes he has saved to the task in the workflow). Once the new rule is maintained, he sends the workflow on for approval. This is done by **Jürgen**, who is responsible, checks everything once more and then approves. After that, several sequential work items run automatically and distribute the created objects from the master data system to the production, test and development systems.

### Workflow

<figure><img src="../.gitbook/assets/folie14.png" alt="Workflow BC master data distribution"><figcaption><p>Process flow: approval and distribution (diagram labels in German)</p></figcaption></figure>
