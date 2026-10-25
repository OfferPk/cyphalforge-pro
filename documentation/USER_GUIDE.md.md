# CyphalForge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "Avionics Transfer Generator" panel, configure your Node ID (e.g. `42`), Priority (`4 - NOMINAL`), and Subject ID (e.g. `7509` for Heartbeat).  
3. Select your Transport MTU (`Classic CAN 8 Bytes` or `CAN FD 64 Bytes`).  
4. Enter telemetry payload bytes or JSON structure.  
5. Click **Segment & Transmit** to calculate the 29-bit CAN ID, generate multi-frame chunks with alternating tail bytes, append CRC-16, and inspect the wire hex dump.  
6. Check the Cyphal/UDP Multicast Target panel for mapped IPv4 multicast endpoints.  
7. Click **Run Normative Audit** to evaluate all 10 OpenCyphal protocol compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 cyphal-cli.py \--audit \--node 42 \--subject 7509