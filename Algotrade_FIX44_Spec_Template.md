<div style="text-align: center; margin-top: 40px;">
  <img src="Algotrade - Logo-04.png" alt="Algotrade" style="width: 60%;">
</div>

<h1 style="text-align: center; border: none;">FIX 4.4 Trading API Specification</h1>

<div style="page-break-before: always;"></div>

## Table of Contents

- [0) Purpose & Background](#0-purpose--background)
- [1) Roles, Connectivity & Account Model](#1-roles-connectivity--account-model)
  - [1.1 Roles](#11-roles)
  - [1.2 Account Model — Single Connection, Multiple Sub-accounts](#12-account-model--single-connection-multiple-sub-accounts)
  - [1.3 Session Schedule & Sequence Numbers](#13-session-schedule--sequence-numbers)
  - [1.4 Reconnection & Failover](#14-reconnection--failover)
- [2) Session Layer (Admin)](#2-session-layer-admin)
  - [2.1 Standard Header](#21-standard-header-required-on-all-messages)
  - [2.2 Logon (35=A)](#22-logon-35a)
  - [2.3 Logout (35=5)](#23-logout-355)
  - [2.4 Mandatory FIX session support](#24-mandatory-fix-session-support)
- [3) Trading Messages (Application)](#3-trading-messages-application)
  - [3.1 New Order Single (35=D)](#31-new-order-single-35d)
    - [3.1.1 Required fields](#311-required-fields)
    - [3.1.2 Custom OrdType note](#312-custom-ordtype-note)
    - [3.1.3 TimeInForce (59) values](#313-timeinforce-59-values)
    - [3.1.4 Additional fields for derivatives](#314-additional-fields-for-derivatives)
    - [3.1.5 Example — Equity](#315-example--equity-limit-order-day)
    - [3.1.6 Example — Derivatives](#316-example--derivatives-futures-limit-order)
  - [3.2 Order Cancel Request (35=F)](#32-order-cancel-request-35f)
- [4) Execution Report (35=8) — Fields, States & Mapping](#4-execution-report-358--fields-states--mapping)
  - [4.1 Fields expected on all Execution Reports](#41-fields-expected-on-all-execution-reports)
  - [4.2 Trade-specific fields](#42-trade-specific-fields-required-when-a-fill-occurs)
  - [4.3 Reject fields](#43-reject-fields-required-when-1508)
    - [4.3.1 OrdRejReason (103) values](#431-ordrejreason-103-values)
  - [4.4 ExecType (150) ↔ OrdStatus (39) Mapping](#44-exectype-150--ordstatus-39-mapping)
- [5) Order Cancel Reject (35=9)](#5-order-cancel-reject-359)
  - [5.1 Required fields](#51-required-fields)
  - [5.2 CxlRejReason (102) values](#52-cxlrejreason-102-values)
  - [5.3 Example](#53-example)
- [6) Business Message Reject (35=j)](#6-business-message-reject-35j)
  - [6.1 Required fields](#61-required-fields)
  - [6.2 BusinessRejectReason (380) values](#62-businessrejectreason-380-values)
  - [6.3 Example](#63-example)
- [7) Order Lifecycles](#7-order-lifecycles)
- [8) Open Items & Questions for \<Broker\>](#8-open-items--questions-for-broker)
- [9) Public References](#9-public-references)
- [Appendix A: Message Type Summary](#appendix-a-message-type-summary)
- [Appendix B: Tag Quick Reference](#appendix-b-tag-quick-reference)

<div style="page-break-before: always;"></div>

## 0) Purpose & Background

This document provides the **recommended FIX 4.4 specification** for \<Broker\>'s FIX trading server, covering message types, required fields, expected behaviors, and order lifecycle flows.

Algotrade provides this specification to support \<Broker\>'s FIX 4.4 implementation. FIX protocol connectivity enables **programmatic order submission with lower latency** compared to GUI-based workflows. This specification reflects common practices for FIX integrations in the Vietnamese market.

Algotrade will serve as a **FIX client to connect and jointly test** the implementation with \<Broker\>.

The specification covers **equities and derivatives** uniformly. Both product types use the same message types and session model; the only difference is a small number of additional tags for contract identification on derivatives orders (see section 3.1.4).

**Covered in this document:**
- Session: **Logon (A)**, **Logout (5)**, and standard FIX session-level messages
- Trading: **New Order Single (D)**, **Order Cancel Request (F)**
- Reporting: **Execution Report (8)** — order acknowledgements, cancels, fills, rejects, and expiry
- Reporting: **Order Cancel Reject (9)** — when a cancel request cannot be honored
- Reporting: **Business Message Reject (j)** — for unsupported or invalid application messages

**Not covered (can be added later if needed):**
- Order Cancel/Replace Request (G), mass cancel, allocation, position management, market data
- Full FIX 4.4 specification details — please refer to the official FIX documentation for all standard behaviors not explicitly covered here

<div style="page-break-before: always;"></div>

## 1) Roles, Connectivity & Account Model

### 1.1 Roles

| Aspect | Value |
|---|---|
| **\<Broker\>** | FIX **Acceptor (Server)** |
| **Algotrade** | FIX **Initiator (Client)** |

### 1.2 Account Model — Single Connection, Multiple Sub-accounts

We recommend that the FIX server support **a single connection carrying orders for multiple sub-accounts**, across both equities and derivatives. This is a standard and scalable approach — it simplifies connectivity for both sides and avoids the overhead of managing separate sessions per account.

The **Account** tag (`1`) on every order identifies the specific sub-account. The FIX server should:

1. **Route orders** to the correct sub-account based on tag `1`.
2. **Echo back** the Account tag (`1`) on every Execution Report (`35=8`) and Order Cancel Reject (`35=9`) so that the client can attribute reports to the correct account.
3. Support **concurrent orders** from different sub-accounts on the same session without interference.

> **Note:** This is a one-to-many model (one FIX session → many accounts). It is the most common architecture for institutional FIX connectivity and is recommended over a one-to-one (one session per account) model.

### 1.3 Session Schedule & Sequence Numbers

| Parameter | Recommended Value |
|---|---|
| **Session start** | Suggest: before market open (e.g., 08:30 ICT / 01:30 UTC) |
| **Session end** | Suggest: after market close (e.g., 15:00 ICT / 08:00 UTC) |
| **Sequence number reset** | Daily at session start, via `141=Y` (ResetSeqNumFlag) on Logon |
| **Heartbeat interval** | 30 seconds (configurable on Logon via tag `108`) |

### 1.4 Reconnection & Failover

- If the TCP connection is lost, the client will reconnect and send a new Logon.
- If `141=Y` is **not** set, both sides should honor existing sequence numbers and use **ResendRequest (35=2)** / **SequenceReset (35=4)** to recover any gaps.
- If `141=Y` is set, sequence numbers reset to 1 on both sides.
- **Working orders** should remain active on the exchange during a disconnect. The server should buffer any Execution Reports generated while the client is disconnected and deliver them upon reconnection (via the standard gap-fill mechanism).
- **Reconnection rate:** the client will not attempt more than 1 reconnection per 5 seconds to avoid overwhelming the server.

<div style="page-break-before: always;"></div>

## 2) Session Layer (Admin)

### 2.1 Standard Header (required on all messages)

| Tag | Field | Req | Notes |
|---:|---|:--:|---|
| 8 | BeginString | Y | `FIX.4.4` |
| 9 | BodyLength | Y | Standard FIX |
| 35 | MsgType | Y | Message type |
| 34 | MsgSeqNum | Y | Monotonic per session |
| 49 | SenderCompID | Y | Sender's CompID (Algotrade when sending) |
| 56 | TargetCompID | Y | Receiver's CompID (\<Broker\> when sending) |
| 52 | SendingTime | Y | UTC timestamp |
| 10 | CheckSum | Y | Standard FIX |

### 2.2 Logon (35=A)

**Algotrade (Initiator)** sends Logon to start a session. **\<Broker\> (Acceptor)** responds with Logon.

**Required fields**
- `35=A`
- `98=0` (EncryptMethod = None)
- `108=<secs>` (HeartBtInt)

**Optional / by agreement**
- `141=Y` (ResetSeqNumFlag) — only when both sides agree to reset sequence numbers
- Credentials — if \<Broker\> requires password auth:
  - `553` (Username)
  - `554` (Password)

**Example — Logon with credentials and seq reset (illustrative)**
```
8=FIX.4.4|35=A|49=ALGOTRADE|56=BROKER|34=1|52=20260101-00:00:00.000|
98=0|108=30|141=Y|553=algotrade_user|554=p@ssw0rd|10=000|
```

**Example — Logon response from \<Broker\>**
```
8=FIX.4.4|35=A|49=BROKER|56=ALGOTRADE|34=1|52=20260101-00:00:00.050|
98=0|108=30|141=Y|10=000|
```

### 2.3 Logout (35=5)

Either party may initiate logout.

- Send `35=5`, optional `58` (Text) for reason
- The peer should respond with a Logout if it has not already sent one

**Example — Logout initiated by Algotrade**
```
8=FIX.4.4|35=5|49=ALGOTRADE|56=BROKER|34=100|52=20260101-08:00:00.000|
58=End of trading day|10=000|
```

**Example — Logout response from \<Broker\>**
```
8=FIX.4.4|35=5|49=BROKER|56=ALGOTRADE|34=95|52=20260101-08:00:00.050|
58=Goodbye|10=000|
```

### 2.4 Mandatory FIX session support

Both sides must support standard FIX 4.4 session messages:
- Heartbeat (35=0), TestRequest (35=1)
- ResendRequest (35=2), SequenceReset (35=4)
- Reject (35=3) — for protocol-level errors (malformed messages, invalid tags, etc.)

> **Note on examples:** Throughout this document, `|` is used in place of SOH (`\x01`) for readability. `BodyLength (9)` is omitted and `CheckSum (10)` is shown as `000`. In production, all must conform to the FIX specification.

<div style="page-break-before: always;"></div>

## 3) Trading Messages (Application)

> Unless otherwise stated, all timestamps are **UTC**.

### 3.1 New Order Single (35=D)

Used to place a new order for equities or derivatives.

#### 3.1.1 Required fields

| Tag | Field | Req | Notes |
|---:|---|:--:|---|
| 35 | MsgType | Y | `D` |
| 11 | ClOrdID | Y | Client order id. **Must be unique** per trading day (recommend ≤ 30 chars). |
| 1 | Account | Y | Sub-account identifier (see section 1.2). Multiple accounts may submit orders on the same FIX session. |
| 207 | SecurityExchange | Y | `HSX` / `HNX` / `UPCOM` |
| 55 | Symbol | Y | Ticker only (e.g., `MWG`) unless later agreed otherwise |
| 54 | Side | Y | `1=Buy`, `2=Sell` |
| 38 | OrderQty | Y | Quantity |
| 40 | OrdType | Y | `1=Market`, `2=Limit`, `K=MTL` *(custom — see note below)* |
| 44 | Price | C | **Required** when `40=2` (Limit); omitted otherwise. See price convention below. |
| 59 | TimeInForce | Y | See below |
| 60 | TransactTime | Y | Order creation time |

> **Price convention:**
> - **Equities:** Price (tag `44`) is the per-share price in **VND** (e.g., MWG at 72,000 VND → `44=72000`).
> - **Derivatives:** Price (tag `44`) is the **index point value** (e.g., VN30F2603 at 2,026.3 points → `44=2026.3`).

#### 3.1.2 Custom OrdType note

`OrdType K = MTL (Market-to-Limit)` is a custom extension specific to Vietnamese exchanges. It is not part of the standard FIX 4.4 OrdType enumeration. Both sides agree to support this value.

#### 3.1.3 TimeInForce (59) values

The following values cover all standard Vietnamese order types (LO, ATO, ATC, MAK, MOK). All values are standard FIX 4.4. Availability may vary between equities and derivatives sessions on the exchange.

| Value | Name | VN Order Type | Notes |
|:---:|---|---|---|
| `0` | Day | LO, MTL | Default for Limit / MTL |
| `2` | OPG (At the Opening) | ATO | |
| `3` | IOC (Immediate or Cancel) | MAK | Default for Market |
| `4` | FOK (Fill or Kill) | MOK | |
| `7` | AtTheClose | ATC | |

#### 3.1.4 Additional fields for derivatives

For derivatives orders (futures, options), the following fields should be included in addition to those in 3.1.1:

| Tag | Field | Req | Notes |
|---:|---|:--:|---|
| 167 | SecurityType | Y | `FUT` for futures; `OPT` for options (if applicable) |
| 200 | MaturityMonthYear | C | Format `YYYYMM` — required if `55` alone is ambiguous |
| 48 | SecurityID | C | Alternative to `55` if \<Broker\> uses a numeric ID scheme |
| 22 | SecurityIDSource | C | Required when `48` is used (e.g., `8` = Exchange Symbol) |

> **Question for \<Broker\>:** Please confirm the preferred contract identification scheme for derivatives — Symbol-based (`55`, e.g., `VN30F2603`) or SecurityID-based (`48` + `22`).

#### 3.1.5 Example — Equity (Limit Order, Day)

```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=21|52=20260101-01:00:00.000|
1=ACC001|11=CL000001|207=HSX|55=MWG|54=1|38=100|40=2|44=50000|59=0|60=20260101-01:00:00.000|10=000|
```

#### 3.1.6 Example — Derivatives (Futures, Limit Order)

```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=100|52=20260315-02:00:00.000|
1=ACC002|11=CL000050|167=FUT|207=HNX|55=VN30F2606|54=1|38=10|40=2|44=1350|59=0|60=20260315-02:00:00.000|10=000|
```

---

### 3.2 Order Cancel Request (35=F)

Requests cancellation of the remaining quantity of a working order.

#### Required fields

| Tag | Field | Req | Notes |
|---:|---|:--:|---|
| 35 | MsgType | Y | `F` |
| 11 | ClOrdID | Y | **New** cancel request id (unique) |
| 1 | Account | Y | Same as original order |
| 37 | OrderID | Y | Order id assigned by \<Broker\> (from Execution Report) |
| 41 | OrigClOrdID | Y | Original ClOrdID of the order being cancelled |
| 207 | SecurityExchange | Y | Exchange code |
| 55 | Symbol | Y | Ticker/contract symbol |
| 54 | Side | Y | Same as original order |
| 60 | TransactTime | Y | Cancel request time |

#### Example

```
8=FIX.4.4|35=F|49=ALGOTRADE|56=BROKER|34=40|52=20260101-01:05:00.000|
11=CXL000001|1=ACC001|37=123456|41=CL000001|207=HSX|55=MWG|54=1|60=20260101-01:05:00.000|10=000|
```

<div style="page-break-before: always;"></div>

## 4) Execution Report (35=8) — Fields, States & Mapping

The FIX server sends Execution Reports to confirm order state changes, trades, cancels, and rejects.

### 4.1 Fields expected on all Execution Reports

| Tag | Field | Req | Notes |
|---:|---|:--:|---|
| 35 | MsgType | Y | `8` |
| 150 | ExecType | Y | Execution type (see mapping in 4.4) |
| 39 | OrdStatus | Y | Order status (see mapping in 4.4) |
| 17 | ExecID | Y | Unique execution id (unique across the session) |
| 37 | OrderID | Y | \<Broker\>-assigned order id |
| 11 | ClOrdID | Y | Client ClOrdID (for cancel flow: the cancel request's ClOrdID) |
| 41 | OrigClOrdID | C | Required for cancel-related reports |
| 1 | Account | Y | Sub-account identifier (echoed from the original order) |
| 55 | Symbol | Y | Ticker / contract symbol |
| 207 | SecurityExchange | Y | Exchange code |
| 54 | Side | Y | `1` / `2` |
| 38 | OrderQty | Y | Original order quantity |
| 40 | OrdType | Y | As received |
| 59 | TimeInForce | Y | As received |
| 44 | Price | C | Required when `OrdType=2` (Limit) |
| 151 | LeavesQty | Y | Remaining open quantity |
| 14 | CumQty | Y | Cumulative filled quantity |
| 6 | AvgPx | Y | Average fill price (`0` if no fills yet) |
| 60 | TransactTime | Y | Execution or event timestamp |

### 4.2 Trade-specific fields (required when a fill occurs)

When `150=F` (Trade) is sent:
- `31` (LastPx) **required**
- `32` (LastQty) **required**

> **Price convention on fills:**
> - `LastPx` (`31`) and `AvgPx` (`6`) represent the **actual execution price before fees/commissions**.
>   - **Equities:** price in **VND** per share (e.g., MWG filled → `31=72000`).
>   - **Derivatives:** price in **index points** (e.g., VN30F2603 filled → `31=2026.3`).
> - Fee/commission information may optionally be provided via the Commission group (`136` NoMiscFees / `137` MiscFeeAmt / `139` MiscFeeType) if supported by \<Broker\> (see Open Item #7).

### 4.3 Reject fields (required when `150=8`)

When an order is rejected:
- `150=8` (Rejected), `39=8` (Rejected)
- `103` (OrdRejReason) **required** — see values below
- `58` (Text) **required** — human-readable reason

#### 4.3.1 OrdRejReason (103) values

| Value | Meaning |
|:---:|---|
| `0` | Broker / exchange option |
| `1` | Unknown symbol |
| `2` | Exchange closed |
| `3` | Order exceeds limit |
| `5` | Unknown order |
| `6` | Duplicate order (ClOrdID) |
| `13` | Incorrect quantity |
| `99` | Other |

> \<Broker\> may define additional values beyond the standard set above. We would appreciate if any custom rejection reasons are documented and shared so that Algotrade can handle them correctly.

### 4.4 ExecType (150) ↔ OrdStatus (39) Mapping

The following table defines the expected combinations. We recommend adhering to this standard mapping for consistency.

| Scenario | ExecType (150) | OrdStatus (39) | LeavesQty | CumQty |
|---|:---:|:---:|---|---|
| Pending New (broker acknowledged) | `A` | `A` | = OrderQty | 0 |
| New (exchange confirmed) | `0` | `0` | = OrderQty | 0 |
| Partial Fill | `F` | `1` | > 0 | > 0 |
| Full Fill | `F` | `2` | 0 | = OrderQty |
| Pending Cancel | `6` | `6` | unchanged | unchanged |
| Cancelled | `4` | `4` | 0 | unchanged |
| Rejected | `8` | `8` | 0 | 0 |
| Expired (IOC/FOK/end-of-day) | `C` | `C` | 0 | unchanged |

**Important notes:**

- **OrdStatus precedence:** If an order is partially filled *and* a cancel is pending, the OrdStatus must be `6` (Pending Cancel), not `1` (Partially Filled). Pending states always take precedence per the FIX specification.
- **Pending Cancel is optional to send:** If the cancellation is processed immediately, the server may skip the Pending Cancel step and send `150=4, 39=4` directly.
- **Expired state:** IOC or FOK orders that do not fully fill should receive `150=C, 39=C` as the terminal report. Day orders that are still open at end-of-day should also receive an Expired or Cancelled report.

<div style="page-break-before: always;"></div>

## 5) Order Cancel Reject (35=9)

When \<Broker\> cannot honor a cancel request (e.g., order already filled, unknown order, too late to cancel), the server should respond with an **Order Cancel Reject** instead of an Execution Report.

### 5.1 Required fields

| Tag | Field | Req | Notes |
|---:|---|:--:|---|
| 35 | MsgType | Y | `9` |
| 11 | ClOrdID | Y | The ClOrdID from the cancel request |
| 37 | OrderID | Y | \<Broker\>-assigned order id |
| 41 | OrigClOrdID | Y | Original order's ClOrdID |
| 1 | Account | Y | Sub-account identifier |
| 39 | OrdStatus | Y | Current status of the order (e.g., `0`, `1`, `2`) |
| 434 | CxlRejResponseTo | Y | `1` = Order Cancel Request |
| 102 | CxlRejReason | Y | See values below |
| 58 | Text | Y | Human-readable explanation |
| 60 | TransactTime | Y | Timestamp |

### 5.2 CxlRejReason (102) values

| Value | Meaning |
|:---:|---|
| `0` | Too late to cancel |
| `1` | Unknown order |
| `2` | Broker / exchange option |
| `3` | Order already in Pending Cancel or Pending Replace status |
| `6` | Duplicate ClOrdID on cancel request |
| `99` | Other |

### 5.3 Example

```
8=FIX.4.4|35=9|49=BROKER|56=ALGOTRADE|34=45|52=20260101-01:06:00.000|
11=CXL000001|37=123456|41=CL000001|1=ACC001|39=2|434=1|102=0|58=Too late to cancel, order already filled|60=20260101-01:06:00.000|10=000|
```

<div style="page-break-before: always;"></div>

## 6) Business Message Reject (35=j)

The FIX server should send a Business Message Reject when it receives a structurally valid FIX message that cannot be processed for business-level reasons (e.g., unsupported `MsgType`, application not available). This is distinct from Session-level Reject (`35=3`), which handles protocol-level errors.

### 6.1 Required fields

| Tag | Field | Req | Notes |
|---:|---|:--:|---|
| 35 | MsgType | Y | `j` |
| 45 | RefSeqNum | Y | MsgSeqNum of the rejected message |
| 372 | RefMsgType | Y | MsgType of the rejected message |
| 380 | BusinessRejectReason | Y | See values below |
| 58 | Text | Y | Human-readable reason |

### 6.2 BusinessRejectReason (380) values

| Value | Meaning |
|:---:|---|
| `0` | Other |
| `1` | Unknown ID |
| `2` | Unknown Security |
| `3` | Unsupported Message Type |
| `4` | Application not available |
| `5` | Conditionally required field missing |
| `6` | Not authorized |

### 6.3 Example

```
8=FIX.4.4|35=j|49=BROKER|56=ALGOTRADE|34=50|52=20260101-01:10:00.000|
45=48|372=D|380=6|58=Account not authorized to trade on HNX|10=000|
```

<div style="page-break-before: always;"></div>

## 7) Order Lifecycles

Full message examples for each scenario. All examples use: Buy MWG on HSX, Account `ACC001`. `BodyLength (9)` omitted, `CheckSum (10)` shown as `000`.

### 7.1 New Order → Accepted

**Step 1.** Algotrade → \<Broker\>: New Order Single
```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=2|52=20260101-01:00:00.000|
1=ACC001|11=CL000001|207=HSX|55=MWG|54=1|38=200|40=2|44=50000|59=0|60=20260101-01:00:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Pending New (`150=A, 39=A`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=2|52=20260101-01:00:00.100|
150=A|39=A|17=EXEC001|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=200|14=0|6=0|60=20260101-01:00:00.100|10=000|
```

**Step 3.** \<Broker\> → Algotrade: New (`150=0, 39=0`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=3|52=20260101-01:00:00.200|
150=0|39=0|17=EXEC002|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=200|14=0|6=0|60=20260101-01:00:00.200|10=000|
```

### 7.2 New Order → Rejected

**Step 1.** Algotrade → \<Broker\>: New Order Single
```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=2|52=20260101-01:00:00.000|
1=ACC001|11=CL000002|207=HSX|55=XYZ|54=1|38=100|40=2|44=10000|59=0|60=20260101-01:00:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Rejected (`150=8, 39=8`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=2|52=20260101-01:00:00.100|
150=8|39=8|17=EXEC010|37=ORD0002|11=CL000002|1=ACC001|55=XYZ|207=HSX|54=1|38=100|40=2|44=10000|59=0|
151=0|14=0|6=0|103=1|58=Unknown symbol|60=20260101-01:00:00.100|10=000|
```

### 7.3 New Order → Partial Fill → Full Fill

**Step 1.** Algotrade → \<Broker\>: New Order Single (Buy 200 MWG @ 50,000)
```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=2|52=20260101-01:00:00.000|
1=ACC001|11=CL000001|207=HSX|55=MWG|54=1|38=200|40=2|44=50000|59=0|60=20260101-01:00:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Pending New (`150=A, 39=A`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=2|52=20260101-01:00:00.100|
150=A|39=A|17=EXEC001|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=200|14=0|6=0|60=20260101-01:00:00.100|10=000|
```

**Step 3.** \<Broker\> → Algotrade: New (`150=0, 39=0`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=3|52=20260101-01:00:00.200|
150=0|39=0|17=EXEC002|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=200|14=0|6=0|60=20260101-01:00:00.200|10=000|
```

**Step 4.** \<Broker\> → Algotrade: Partial Fill (`150=F, 39=1`) — filled 100 @ 50,000
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=4|52=20260101-01:01:00.000|
150=F|39=1|17=EXEC003|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
32=100|31=50000|151=100|14=100|6=50000|60=20260101-01:01:00.000|10=000|
```

**Step 5.** \<Broker\> → Algotrade: Full Fill (`150=F, 39=2`) — filled 100 @ 50,500
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=5|52=20260101-01:02:00.000|
150=F|39=2|17=EXEC004|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
32=100|31=50500|151=0|14=200|6=50250|60=20260101-01:02:00.000|10=000|
```

### 7.4 Cancel — Success

Assumes order `CL000001` is in New state (`39=0`) with `OrderID=ORD0001`.

**Step 1.** Algotrade → \<Broker\>: Order Cancel Request
```
8=FIX.4.4|35=F|49=ALGOTRADE|56=BROKER|34=5|52=20260101-01:05:00.000|
11=CXL000001|1=ACC001|37=ORD0001|41=CL000001|207=HSX|55=MWG|54=1|60=20260101-01:05:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Pending Cancel (`150=6, 39=6`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=6|52=20260101-01:05:00.100|
150=6|39=6|17=EXEC005|37=ORD0001|11=CXL000001|41=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=200|14=0|6=0|60=20260101-01:05:00.100|10=000|
```

**Step 3.** \<Broker\> → Algotrade: Cancelled (`150=4, 39=4`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=7|52=20260101-01:05:00.200|
150=4|39=4|17=EXEC006|37=ORD0001|11=CXL000001|41=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=0|14=0|6=0|60=20260101-01:05:00.200|10=000|
```

> Step 2 (Pending Cancel) may be skipped if the cancel is processed immediately.

### 7.5 Cancel — Rejected (order already filled)

**Step 1.** Algotrade → \<Broker\>: Order Cancel Request
```
8=FIX.4.4|35=F|49=ALGOTRADE|56=BROKER|34=8|52=20260101-01:06:00.000|
11=CXL000002|1=ACC001|37=ORD0001|41=CL000001|207=HSX|55=MWG|54=1|60=20260101-01:06:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Order Cancel Reject (`35=9`)
```
8=FIX.4.4|35=9|49=BROKER|56=ALGOTRADE|34=9|52=20260101-01:06:00.100|
11=CXL000002|37=ORD0001|41=CL000001|1=ACC001|39=2|434=1|102=0|58=Too late to cancel, order already filled|
60=20260101-01:06:00.100|10=000|
```

### 7.6 Partial Fill → Cancel Remaining

**Step 1.** Algotrade → \<Broker\>: New Order Single (Buy 200 MWG @ 50,000)
```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=2|52=20260101-01:00:00.000|
1=ACC001|11=CL000001|207=HSX|55=MWG|54=1|38=200|40=2|44=50000|59=0|60=20260101-01:00:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Pending New (`150=A, 39=A`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=2|52=20260101-01:00:00.100|
150=A|39=A|17=EXEC001|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=200|14=0|6=0|60=20260101-01:00:00.100|10=000|
```

**Step 3.** \<Broker\> → Algotrade: New (`150=0, 39=0`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=3|52=20260101-01:00:00.200|
150=0|39=0|17=EXEC002|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=200|14=0|6=0|60=20260101-01:00:00.200|10=000|
```

**Step 4.** \<Broker\> → Algotrade: Partial Fill (`150=F, 39=1`) — filled 100 @ 50,000
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=4|52=20260101-01:01:00.000|
150=F|39=1|17=EXEC003|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
32=100|31=50000|151=100|14=100|6=50000|60=20260101-01:01:00.000|10=000|
```

**Step 5.** Algotrade → \<Broker\>: Order Cancel Request
```
8=FIX.4.4|35=F|49=ALGOTRADE|56=BROKER|34=5|52=20260101-01:02:00.000|
11=CXL000001|1=ACC001|37=ORD0001|41=CL000001|207=HSX|55=MWG|54=1|60=20260101-01:02:00.000|10=000|
```

**Step 6.** \<Broker\> → Algotrade: Cancelled (`150=4, 39=4`) — 100 filled, 100 cancelled
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=5|52=20260101-01:02:00.100|
150=4|39=4|17=EXEC007|37=ORD0001|11=CXL000001|41=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=0|14=100|6=50000|60=20260101-01:02:00.100|10=000|
```

### 7.7 IOC Order → Expired (no fill)

**Step 1.** Algotrade → \<Broker\>: New Order Single (IOC Market, 100 shares)
```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=2|52=20260101-01:00:00.000|
1=ACC001|11=CL000003|207=HSX|55=MWG|54=1|38=100|40=1|59=3|60=20260101-01:00:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Pending New (`150=A, 39=A`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=2|52=20260101-01:00:00.100|
150=A|39=A|17=EXEC020|37=ORD0003|11=CL000003|1=ACC001|55=MWG|207=HSX|54=1|38=100|40=1|59=3|
151=100|14=0|6=0|60=20260101-01:00:00.100|10=000|
```

**Step 3.** \<Broker\> → Algotrade: Expired (`150=C, 39=C`)
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=3|52=20260101-01:00:00.200|
150=C|39=C|17=EXEC021|37=ORD0003|11=CL000003|1=ACC001|55=MWG|207=HSX|54=1|38=100|40=1|59=3|
151=0|14=0|6=0|60=20260101-01:00:00.200|10=000|
```

### 7.8 IOC Order → Partial Fill → Expired

**Step 1.** Algotrade → \<Broker\>: New Order Single (IOC Market, 200 shares)
```
8=FIX.4.4|35=D|49=ALGOTRADE|56=BROKER|34=2|52=20260101-01:00:00.000|
1=ACC001|11=CL000004|207=HSX|55=MWG|54=1|38=200|40=1|59=3|60=20260101-01:00:00.000|10=000|
```

**Step 2.** \<Broker\> → Algotrade: Partial Fill (`150=F, 39=1`) — filled 80 @ 50,000
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=2|52=20260101-01:00:00.100|
150=F|39=1|17=EXEC030|37=ORD0004|11=CL000004|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=1|59=3|
32=80|31=50000|151=120|14=80|6=50000|60=20260101-01:00:00.100|10=000|
```

**Step 3.** \<Broker\> → Algotrade: Expired (`150=C, 39=C`) — remaining 120 expired
```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=3|52=20260101-01:00:00.200|
150=C|39=C|17=EXEC031|37=ORD0004|11=CL000004|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=1|59=3|
151=0|14=80|6=50000|60=20260101-01:00:00.200|10=000|
```

### 7.9 Unsolicited Cancel (exchange-initiated)

Exchange cancels order without client request (e.g., trading halt, regulatory action).

```
8=FIX.4.4|35=8|49=BROKER|56=ALGOTRADE|34=10|52=20260101-02:00:00.000|
150=4|39=4|17=EXEC040|37=ORD0001|11=CL000001|1=ACC001|55=MWG|207=HSX|54=1|38=200|40=2|44=50000|59=0|
151=0|14=0|6=0|58=Trading halt - order cancelled by exchange|60=20260101-02:00:00.000|10=000|
```

Algotrade identifies this as unsolicited because no preceding cancel request (`35=F`) was sent.

<div style="page-break-before: always;"></div>

## 8) Open Items & Questions for \<Broker\>

We would be happy to discuss any of the following items. Confirming these early will help ensure a smooth implementation and testing process.

| # | Item | Status |
|---|---|---|
| 1 | UAT and Production endpoints (host, port, CompIDs) | Pending |
| 2 | Session schedule (session start/end times) | Pending |
| 3 | SSL/TLS requirement on transport layer | Pending |
| 4 | Authentication method — Username/Password (`553`/`554`) on Logon? | Pending |
| 5 | Derivatives contract identification scheme (`55` vs `48`+`22`) | Pending |
| 6 | Custom OrdRejReason or CxlRejReason values the server plans to use | Pending |
| 7 | Will Execution Reports include fee information? (tags `136/137/139`) | Pending |
| 8 | Maximum message rate / throttling limits | Pending |
| 9 | Behavior of working orders during disconnect — remain active? | Pending |
| 10 | End-of-day handling — Expired (`39=C`) or Cancelled (`39=4`) for unfilled Day orders? | Pending |

---

## 9) Public References

- FIX 4.4 order state changes: https://www.fixtrading.org/online-specification/order-state-changes/
- FIX 4.4 field dictionary: https://www.onixs.biz/fix-dictionary/4.4/fields_by_tag.html

<div style="page-break-before: always;"></div>

## Appendix A: Message Type Summary

| MsgType | Name | Direction | Description |
|:---:|---|---|---|
| `A` | Logon | Both | Session initiation |
| `5` | Logout | Both | Session termination |
| `0` | Heartbeat | Both | Connection keep-alive |
| `1` | TestRequest | Both | Solicit heartbeat |
| `2` | ResendRequest | Both | Request retransmission of messages |
| `4` | SequenceReset | Both | Reset sequence numbers or gap-fill |
| `3` | Reject | Both | Session-level message rejection |
| `D` | NewOrderSingle | Algotrade → \<Broker\> | Submit a new order |
| `F` | OrderCancelRequest | Algotrade → \<Broker\> | Request order cancellation |
| `8` | ExecutionReport | \<Broker\> → Algotrade | Order status, fills, cancels, rejects |
| `9` | OrderCancelReject | \<Broker\> → Algotrade | Cancel request rejected |
| `j` | BusinessMessageReject | \<Broker\> → Algotrade | Application-level rejection |

## Appendix B: Tag Quick Reference

All application-level tags used in this specification:

| Tag | Field | Used In |
|---:|---|---|
| 1 | Account | D, F, 8, 9 |
| 6 | AvgPx | 8 |
| 11 | ClOrdID | D, F, 8, 9 |
| 14 | CumQty | 8 |
| 17 | ExecID | 8 |
| 22 | SecurityIDSource | D (derivatives) |
| 31 | LastPx | 8 (fills) |
| 32 | LastQty | 8 (fills) |
| 37 | OrderID | F, 8, 9 |
| 38 | OrderQty | D, 8 |
| 39 | OrdStatus | 8, 9 |
| 40 | OrdType | D, 8 |
| 41 | OrigClOrdID | F, 8, 9 |
| 44 | Price | D, 8 |
| 45 | RefSeqNum | j |
| 48 | SecurityID | D (derivatives) |
| 54 | Side | D, F, 8 |
| 55 | Symbol | D, F, 8 |
| 58 | Text | 8, 9, j |
| 59 | TimeInForce | D, 8 |
| 60 | TransactTime | D, F, 8, 9 |
| 102 | CxlRejReason | 9 |
| 103 | OrdRejReason | 8 |
| 150 | ExecType | 8 |
| 151 | LeavesQty | 8 |
| 167 | SecurityType | D (derivatives) |
| 200 | MaturityMonthYear | D (derivatives) |
| 207 | SecurityExchange | D, F, 8 |
| 372 | RefMsgType | j |
| 380 | BusinessRejectReason | j |
| 434 | CxlRejResponseTo | 9 |
