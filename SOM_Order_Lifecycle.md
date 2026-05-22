# ServiceNow Order Management - Order Lifecycle

## Overview

This document describes the complete lifecycle of an order in ServiceNow Sales and Order Management (SOM), from creation to closure.

---

## Order Lifecycle Flow

```
Order Creation → Order Submission → Fulfillment Approval → Order Decomposition → Task Execution → Order Closure
```

---

## 1. Order Creation

### Navigation
- **Sales and Order Management → Orders → New**
- Or via API / Agent Assist

### What Happens
- A new **Customer Order** record is created (table: `sn_ind_tmt_orm_order`)
- Order state: **Draft**
- Key fields filled: Account, Contact, Order Type (Add/Modify/Disconnect), Channel, Shipping/Billing Address

### Order Types
| Type | Purpose |
|------|---------|
| Add | New products/services |
| Modify | Change existing products |
| Disconnect | Remove products/services |

---

## 2. Adding Line Items

### What Happens
- Products are added from the **Sales Catalog**
- Each product creates an **Order Line Item** (table: `sn_ind_tmt_orm_order_line_item`)
- Line items reference a **Product Offering** which links to a **Product Specification**

### Bundle Orders
For bundle products:
```
Parent Line Item (Bundle)
  ├── Child Line Item 1 (component product)
  ├── Child Line Item 2 (component product)
  └── Child Line Item 3 (component product)
```

### Product Configuration
- If a product is configurable, the **CPQ Configurator** opens
- Users select characteristics (e.g., Contract Duration, Plan Type)
- Configuration rules enforce compatibility and dependencies

---

## 3. Order Submission

### What Happens
- Order state changes: **Draft → In Progress**
- Order goes through validation (mandatory fields, pricing checks)
- Pricing is calculated based on price lists and pricing rules

---

## 4. Fulfillment Approval

### What Happens
- Order may go through an approval workflow (if configured)
- Once approved, the order moves to **decomposition**

---

## 5. Order Decomposition

### When It Triggers
Product Orders are **NOT** created when you add line items to the order. They are created **only after** the order is submitted and decomposition runs.

```
Order submitted → State changes to "In Progress" → Decomposition runs → Product Orders created
```

The trigger is the order state changing to **"In Progress"** (after submission/fulfillment approval). Before that, only the order and line items exist in the system.

### What Happens
The system breaks down the order into fulfillable units:

```
Customer Order
  └── Order Line Item (e.g., Fiber Broadband Bundle)
        └── Product Order (PO) - one per specification
              ├── Service Order (if service specs exist)
              └── Resource Order (if resource specs exist)
```

### How Decomposition Works
1. System reads the **Specification Relationships** defined in the Product Catalog
2. **Decomposition Rules** determine which service/resource specs are needed based on characteristics
3. **Product Orders** are created for each specification in the hierarchy
4. **Order Tasks** are created based on:
   - **Specification Category** → triggers specific request definitions
   - **Subflows** in Flow Designer define the task creation logic

### Example
```
Order Line Item: VoIP Phone Unlimited
  └── Product Order: "Product Order for VoIP Phone Spec"
        ├── Order Task: "Document Verification"
        ├── Order Task: "Select SIM Number"
        ├── Order Task: "Pair SIM and Mobile Number"
        ├── Order Task: "Initiate SIM Shipment"
        └── Order Task: "Service Activation for 4G Mobile"
```

### Key Tables
| Table | Purpose |
|-------|---------|
| `sn_ind_tmt_orm_order` | Customer Order |
| `sn_ind_tmt_orm_order_line_item` | Order Line Items |
| `sn_ind_tmt_orm_product_order` | Product Orders |
| `sn_ind_tmt_orm_order_task` | Order Tasks |
| `sn_ind_tmt_orm_mobile_order_task` | Mobile-specific Order Tasks |

---

## 6. Task Execution (Fulfillment)

### What Happens
- Fulfillment agents work on order tasks
- Tasks follow a sequence (based on dependencies)
- Each task moves through states: Open → In Progress → Closed Complete

### Task Types
| Task | Triggered By |
|------|-------------|
| Document Verification | OOTB for mobile plans |
| Select SIM Number | "Mobile Plan" specification category |
| Pair SIM and Mobile Number | OOTB subflow |
| Initiate SIM Shipment | OOTB subflow |
| Service Activation | OOTB subflow |

### How Tasks are Linked to Products
```
Product Specification → Specification Category → Request Definition → Order Task
```
- **Specification Category** (e.g., "Mobile Plan") determines which tasks get created
- **Request Definitions** (table: `sn_ind_request_definition`) define the task template
- **Subflows** in Flow Designer orchestrate the task creation during decomposition

---

## 7. Order Closure (State Rollup)

### Expected Flow (Simple/Non-Bundle Orders)
```
All Order Tasks close (state 5)
  → Product Order auto-closes (state 4)
    → Order Line Item auto-closes (state: completed)
      → Customer Order auto-closes (state: completed)
```
This works automatically via OOTB business rules and flows.

### Bundle Orders (Parent-Child Line Items)
```
All Order Tasks close
  → Child Product Orders close
    → Child Line Items close (manually or via rollup)
      → Parent Product Order needs to close
        → Parent Line Item closes
          → "Update Child Line State" BR cascades to children
            → Customer Order closes
```

### Key Business Rules for State Rollup
| Business Rule | Table | Purpose |
|---------------|-------|---------|
| Update Child Line State | `sn_ind_tmt_orm_order_line_item` | When parent line item reaches terminal state, cascades to children |
| Update Order Line Item State | `sn_ind_tmt_orm_order` | Updates line item state on order state change |
| Update OL state for new, reject | `sn_ind_tmt_orm_order` | Updates order line state on order state change |

### Bundle Rollup Gap
- The "Update Child Line State" rule only cascades **downward** (parent → children)
- There is no OOTB rule that rolls **upward** (all children complete → parent auto-completes)
- For bundle orders, the **parent Product Order** may require manual closure

---

## 8. Order States

### Customer Order States
| State | Description |
|-------|-------------|
| Draft | Order being created |
| In Progress | Order submitted, being fulfilled |
| Completed | All fulfillment done |
| Cancelled | Order cancelled |

### Order Line Item States
| State | Description |
|-------|-------------|
| Draft | Line item added |
| In Progress | Being fulfilled |
| Completed | All tasks done |
| Cancelled | Line item cancelled |

### Product Order / Task States
| State | Value | Description |
|-------|-------|-------------|
| Open | 1 | Not started |
| In Progress | 2 | Being worked on |
| Closed Complete | 3/4 | Successfully completed |
| Closed Incomplete | 4/5 | Closed but not fulfilled |

---

## 9. Key Configuration Points

| Configuration | Where | Purpose |
|--------------|-------|---------|
| Product Specifications | Product Catalog Management → Product Specifications | Define products |
| Specification Categories | Product Catalog Management → Specification Category | Group specs for fulfillment logic |
| Specification Relationships | On the specification record (related list) | Define decomposition hierarchy |
| Decomposition Rules | Order Management → Administration | Conditional decomposition |
| Request Definitions | Spoke Selector → Request Definitions | Task templates |
| Subflows | Flow Designer | Orchestrate task creation/sequencing |
| Business Rules | System Definition → Business Rules | State management and rollup |
| Compatibility Rules | Compatibility Management | Product compatibility/dependencies |

---

## 10. Visual Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    ORDER LIFECYCLE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CREATE → SUBMIT → APPROVE → DECOMPOSE → FULFILL → CLOSE   │
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Order   │───▶│  Line    │───▶│ Product  │              │
│  │          │    │  Items   │    │  Orders  │              │
│  └──────────┘    └──────────┘    └────┬─────┘              │
│                                       │                      │
│                                       ▼                      │
│                                  ┌──────────┐               │
│                                  │  Order   │               │
│                                  │  Tasks   │               │
│                                  └──────────┘               │
│                                       │                      │
│                                       ▼                      │
│                              TASK COMPLETION                  │
│                                       │                      │
│                                       ▼                      │
│                              STATE ROLLUP                    │
│                         (Tasks → PO → OLI → Order)          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## References

- ServiceNow Docs: [Configure telecommunications order fulfillment](https://docs.servicenow.com/bundle/paris-telecommunications-management/page/product/tmt-telecom-order-mgt/task/configure-order-management.html)
- ServiceNow Docs: [Customer order decomposition](https://docs.servicenow.com/bundle/paris-telecommunications-management/page/product/tmt-telecom-order-mgt/concept/order-mgt-order-decomposition.html)
- ServiceNow Docs: [Specification relationships and decomposition rules](https://docs.servicenow.com/bundle/paris-telecommunications-management/page/product/tmt-telecom-order-mgt/task/order-mgt-specification-rels.html)
