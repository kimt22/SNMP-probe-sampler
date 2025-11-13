# SNMP-probe-sampler

A command-line SNMP probing tool that samples an SNMP agent at a fixed frequency and calculates **rate of change** for any number of COUNTER-type OIDs (both 32-bit and 64-bit).
The tool maintains strict sampling frequency, handles timeouts, detects counter wrap-around, and identifies SNMP agent restarts using **sysUpTime**.

## **Features**

### ✔ Handles both COUNTER32 and COUNTER64 types

Includes wrap-around compensation for both modulo (2^{32}) and (2^{64}) counters.

### ✔ Maintains required sampling frequency

Requests are sent at exact intervals regardless of delays or SNMP timeouts.

### ✔ Detects SNMP agent restarts

If `sysUpTime` decreases, the script resets internal state.

### ✔ Robust timeout handling

If a device fails to respond, the script waits and continues maintaining correct sampling intervals.

### ✔ Continuous or fixed-sample operation

* **N = -1** → run forever
* **N ≥ 2** → collect exactly N successful samples

### ✔ Output follows required format

```
<Sample Time> | <Rate OID1> | <Rate OID2> | ... | <Rate OIDn>
```

### ✔ Uses SNMP API (easysnmp)

No system commands, fully compliant with assignment requirement to use an SNMP API.

---

## **Usage**

### **Command Format**

```
prober <IP:port:community> <sample frequency> <samples> <OID1> <OID2> ... <OIDn>
```

### **Argument Details**

| Argument            | Description                                    |
| ------------------- | ---------------------------------------------- |
| `IP:port:community` | SNMP agent connection info                     |
| `sample frequency`  | Sampling frequency in Hz (0.1–10 Hz required)  |
| `samples`           | Number of samples (≥2), or `-1` for continuous |
| `OIDn`              | Absolute OIDs to probe (must be COUNTER types) |

---

## **Example**

Probe two OIDs at **1 Hz** for **10 samples**:

```bash
./prober.py 18.219.51.6:1611:public 1 10 1.3.6.1.4.1.4171.40.2 1.3.6.1.4.1.4171.40.3
```

Probe indefinitely at **0.5 Hz**:

```bash
./prober.py 18.219.51.6:1612:public 0.5 -1 1.3.6.1.2.1.2.2.1.10.2
```

---

## **Output Format (Required)**

```
<Unix Timestamp> | <Rate OID1> | <Rate OID2> | ... | <Rate OIDn>
```

Example:

```
1504083911 | 2124 | 819 | 0 | 281761
1504083912 | 2471 | 819 | 110 | 450782
1504083913 | 1904 | 819 | 2000 | 325448
```

Where each column is the **rate of change between the last two successful samples**.

---

## **Technical Details & Behavior**

### **Sampling Frequency Enforcement**

* Script timestamps each request cycle.
* Computes drift and compensates next sleep duration so that outgoing SNMP queries are emitted exactly every **1/Fs seconds**.
* Required for packet-trace validation.

### **Timeout Handling**

* If a device does not respond:

  * A warning prints.
  * The script waits 5 seconds.
  * The sampling timer continues based only on local time, producing the required behavior where inter-sample interval becomes `n / Fs`.

### **Agent Restart Detection**

* Obtains sysUpTime (`1.3.6.1.2.1.1.3.0`) every probe.
* If sysUpTime drops, internal state resets and counters are treated as new.

### **Counter Wrap-Around**

* If delta < 0:

  * For COUNTER32 → add (2^{32})
  * For COUNTER64 → add (2^{64})

### **SNMP API**

This tool uses **easysnmp**, which wraps `net-snmp` and provides high-level access to SNMP operations.

Install:

```bash
pip install easysnmp
```

You must also have system-level `net-snmp` libraries installed.
