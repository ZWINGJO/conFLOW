# Customizing and launchpad

*Four places, and none of them complains when it is missing. The work item simply shows the text block again.*

## 1 · SWFVMD1: once per system

The link between work item and intent is **not** maintained per workflow, but once for the generic conFLOW task `TS00388601`. The entries are **dynamic**: they read the container elements set by conFLOW.

Transaction `SWFVMD1` → task `TS00388601` → task visualization `INTENT`:

| Parameter | Value | Dynamic |
| --- | --- | --- |
| `SEMANTIC_OBJECT` | `{&/C09/CFL_VISU_SEMANTIC_OBJECT&}` | ✓ |
| `ACTION` | `{&/C09/CFL_VISU_ACTION&}` | ✓ |
| `QUERY_PARAM00` | `CFLQueryObject00={&/C09/CFL_VISU_QUERY_OBJ00&}` | ✓ |
| `QUERY_PARAM01` … `05` | same pattern with `…OBJ01` … `…OBJ05` | ✓ |

{% hint style="info" %}
**Often this already exists.** If a conFLOW app is already attached to a work item in the system, the entries are there. Check: `SE16` → `SWFVGTP` with `TASK = TS00388601`. If the rows above are present, there is nothing to do here.
{% endhint %}

{% hint style="danger" %}
**SWFVMD1 takes precedence over SWFVISU.** The runtime reads `SWFVGT`/`SWFVGTP` first and SWFVISU only afterwards. Maintaining SWFVISU while SWFVMD1 has an entry has no effect — and there is no error message.
{% endhint %}

## 2 · Semantic object

Transaction `/UI2/SEMOBJ` → new entry:

| Field | Value |
| --- | --- |
| Semantic object | `ZCFLOrderPromiseV2` — **exactly** as the value of `VISU` |
| Description | any |

Underscores are allowed, a hyphen is not — it separates semantic object and action.

## 3 · Target mapping, catalog and role

Launchpad Designer (`/UI2/FLPD_CUST`) → your own catalog, e.g. `Z_CFL_00500` → **target mapping, no tile**. The app is never started from the launchpad, only from the work item.

| Field | Value |
| --- | --- |
| Semantic object | `ZCFLOrderPromiseV2` |
| Action | `openInInbox` |
| Application type | SAPUI5 Fiori App |
| Title | any |
| URL | `/sap/bc/ui5_ui5/sap/zcfl_00500_uiv2` (the BSP application) |
| ID | `zcfl00500v2` — the `sap.app.id` from `manifest.json` |
| Device types | Desktop · Tablet · Phone |
| Allow additional parameters | ✓ |
| Parameter | name `openMode` · **Mandatory ✓** · **Value** `embedIntoDetailsNestedRouter` · Default Value **empty** |

{% hint style="danger" %}
**`openMode` belongs in the *Value* column, not in *Default Value*.** My Inbox queries the intent six times, once per mode, and expects **exactly one** match. *Value* acts as a filter and lets exactly one mode match. *Default Value* does not filter, so all six match. My Inbox treats six matches like none: it silently falls back to the text block.
{% endhint %}

{% hint style="info" %}
**Why `embedIntoDetailsNestedRouter`.** The name is misleading — the app does not need a router. The mode switches two things:

**1.** the **"Show Details"** button, which shows the inbox's notes, attachments and object links on the right. It only exists in this mode, which is why you don't rebuild attachments.

**2.** **reuse** of the app when the user jumps to another work item. For this the app must provide two methods, see [The app](app.md).
{% endhint %}

**Role:** add the catalog to a PFCG role and **generate the profile**. Without the role the intent does not resolve. This is the most common cause of "it works for me, but not for you".

## 4 · Register the OData service

`/IWFND/MAINT_SERVICE` → *Add Service* → system alias `LOCAL` → technical service name `ZCFL_00500_V2_SRV` → *Add*.

Then authorize the OData service in the same role (`S_SERVICE`), otherwise the app loads but receives no data.

## 5 · After every app upload: the caches

| What | Why |
| --- | --- |
| `/UI5/APP_INDEX_CALCULATE` | without this run the server keeps delivering the old version, **even in an incognito window** |
| `/UI2/INVALIDATE_CLIENT_CACHES` | cache buster of the launchpad |
| Browser: DevTools → Application → **Clear site data** | UI5 stores views in IndexedDB, and "clear cache" does not touch it |

After a change to the SEGW model, additionally: `/IWFND/MAINT_SERVICE` → select the service → *Clear metadata cache*.

{% hint style="info" %}
**Diagnosis:** incognito shows what the server delivers, the normal window shows the cache. If **both** are outdated, the app index is missing. If only **one** is outdated, it is stuck in browser storage.
{% endhint %}
