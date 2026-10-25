# CyphalForge Pro Technical Architecture & Specification

## 1\. OpenCyphal (UAVCAN v1.0) Architecture

OpenCyphal is an open technology for real-time intravehicular distributed computing and communication based on modern networking principles (CAN bus, Ethernet, serial). It is designed specifically for aerospace, defense drones, robotics, and safety-critical vehicles.

### Key Architectural Pillars:

1. **Deterministic 29-Bit CAN Header**:  
   - `Priority (3 bits)`: 0 (Exceptional) to 7 (Optional).  
   - `Anonymous bit`: Enables node address auto-allocation via plug-and-play mechanisms.  
   - `Subject ID (13 bits)`: Maps telemetry data types to logical pub/sub data streams.  
   - `Source Node ID (7 bits)`: Unique address (0–127) across physical bus segments.  
2. **Multi-Frame Transfer Protocol**:  
   - Manages large payload segmentation exceeding CAN FD (64B) or Classic CAN (8B).  
   - Incorporates a 1-byte Tail Byte containing Start (\$S\$), End (\$E\$), Toggle (\$T\$), and Transfer ID (\$TID\$).  
   - Trailing 2-byte CRC-16/CCITT (\$0x1021\$) detects frame dropouts and out-of-order delivery.  
3. **Cyphal/UDP Transport Specification**:  
   - Direct translation of CAN subject streams into IPv4 multicast groups (`239.0.X.Y:16384`) without protocol proxies.  
4. **Data Structure Description Language (DSDL v1.0)**:  
   - High-integrity serialization format eliminating unaligned data structures and implicit type conversions.