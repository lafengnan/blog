# Settlement System

## 0x1 Background

Since we will provide online loan products to customers, these products would be packaged by one equivalant partner named *SPV*, and without orgnizations. We should develop a new settlement service to support the sales. This new settlement service will be huge different with the one we have developed for fund products.  Here are the pieces of differences between traditional fund prouct and online finance product:

- No *TA* organs in our online product
- *SPV* takes the place of *TA* to act as a partner entity
- *SPV* does not hold on accounts, the data it collects are all from our side

## 0x2 Design

### Architecture

The architechture of settelment service looks like below diagram. 

![settlement_arch](/resources/settlement_arch.png)

As you can see, settlement service is a cross-over service that will interact with *account service*, *order service*, *payment service* , *mis service*, and *product service*, and the settlement result should be logged with *bookkeeper* module for persistence. 

### State Machine

To make the document clear and simple, I only put state transition diagram here. In fact audit process is the core of settlement service.

![settelment_state](/resources/settlement_state.png)

There are many events on the state transition diagram, but in fact the state only transits either by action of **APPROVE(01)** or **REJECT(1)**.

If the action is **APPROVE**, state will transit to next state.

If the action is **REJECT**, state will transit to REJECTED(-1). 

## 0x3 Settlement Flow

*TBD*

## 0x4 Appendix

## API Reference

We design and implemented several APIs to serve the settelement. Here is the first release of APIs. 

| API                          | URI (draft)                             | Method | Progress | Comments |
| ---------------------------- | --------------------------------------- | ------ | -------- | -------- |
| Create Report                | /settlement/dlc/report                  | POST   | Done     |          |
| Query audit journal          | /settlement/dlc/journal/:kind/:id       | GET    | Done     |          |
| Query Report (SPV to us)     | /settlement/dlc/report/:kind/in         | GET    | Done     |          |
| Query Report (us to SPV)     | /settlement/dlc/report/:kind/out        | GET    | Done     |          |
| Audit Report                 | /settlement/dlc/audit                   | POST   | Done     |          |
| Export Excel (SPV to us)     | /settlement/dlc/report/:kind/in/export  | GET    | TBD      |          |
| Export Excel (to SPV)        | /settlement/dlc/report/:kind/out/export | GET    | TBD      |          |
| Fix error record (to SPV)    | /settlement/dlc/report/:kind/out        | POST   | TBD      |          |
| Fix error record (SPV to us) | /settlement/dlc/report/:kind/in         | POST   | TBD      |          |