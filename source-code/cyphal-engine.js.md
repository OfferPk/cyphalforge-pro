/\*\*  
 \* OpenCyphal (UAVCAN v1.0) Core Wire Framing & Multi-Frame Transfer Engine  
 \* Implements 29-bit CAN ID bitfield decomposition, tail-byte segmentation,  
 \* CRC-16/CCITT transfer integrity, and Cyphal/UDP multicast mappings.  
 \*/

// Cyphal Priorities (3 bits)  
const CYPHAL\_PRIORITIES \= {  
  EXCEPTIONAL: 0,  
  IMMEDIATE: 1,  
  FAST: 2,  
  HIGH: 3,  
  NOMINAL: 4,  
  LOW: 5,  
  SLOW: 6,  
  OPTIONAL: 7  
};

class CyphalEngine {  
  constructor(nodeId \= 42\) {  
    this.nodeId \= nodeId; // 7-bit (0-127)  
    this.transferId \= 0;   // 5-bit (0-31)  
  }

  getNextTransferId() {  
    const tid \= this.transferId;  
    this.transferId \= (this.transferId \+ 1\) % 32;  
    return tid;  
  }

  /\*\*  
   \* Constructs a 29-bit Cyphal/CAN Message Identifier  
   \* Bit Allocation:  
   \* \[28:26\] Priority (3 bits)  
   \* \[25\]    Anonymous transfer bit (1 if anonymous)  
   \* \[24\]    Reserved/Discriminator  
   \* \[23:21\] Reserved  
   \* \[20:8\]  Subject ID (13 bits, 0-8191)  
   \* \[7\]     Service/Message flag (0 \= Message)  
   \* \[6:0\]   Source Node ID (7 bits, 0-127)  
   \*/  
  static encodeMessageCanId(priority, subjectId, sourceNodeId) {  
    let canId \= 0;  
    canId \|= (priority & 0x07) \<\< 26;  
    canId \|= 0 \<\< 25; // Non-anonymous  
    canId \|= (subjectId & 0x1FFF) \<\< 8;  
    canId \|= 0 \<\< 7; // Message transfer  
    canId \|= (sourceNodeId & 0x7F);  
    return canId \>\>\> 0;  
  }

  /\*\*  
   \* Decomposes a 29-bit Cyphal/CAN Identifier into fields  
   \*/  
  static decodeCanId(canId) {  
    const priority \= (canId \>\> 26\) & 0x07;  
    const isService \= ((canId \>\> 7\) & 0x01) \=== 1;  
    const sourceNodeId \= canId & 0x7F;  
    const subjectId \= \!isService ? ((canId \>\> 8\) & 0x1FFF) : null;  
    const serviceId \= isService ? ((canId \>\> 14\) & 0x1FF) : null;

    return {  
      canId: canId \>\>\> 0,  
      hexCanId: '0x' \+ (canId \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      priority: priority,  
      isService: isService,  
      subjectId: subjectId,  
      serviceId: serviceId,  
      sourceNodeId: sourceNodeId  
    };  
  }

  /\*\*  
   \* Cyphal CRC-16/CCITT accumulator (Polynomial 0x1021)  
   \*/  
  static calculateCrc16(bytes) {  
    let crc \= 0xFFFF;  
    for (let b of bytes) {  
      crc ^= (b \<\< 8);  
      for (let i \= 0; i \< 8; i++) {  
        if ((crc & 0x8000) \!== 0\) {  
          crc \= ((crc \<\< 1\) ^ 0x1021) & 0xFFFF;  
        } else {  
          crc \= (crc \<\< 1\) & 0xFFFF;  
        }  
      }  
    }  
    return crc;  
  }

  /\*\*  
   \* Constructs Tail Byte: \[Start (1b), End (1b), Toggle (1b), TransferID (5b)\]  
   \*/  
  static createTailByte(start, end, toggle, transferId) {  
    let byte \= (transferId & 0x1F);  
    if (start) byte \|= (1 \<\< 7);  
    if (end) byte \|= (1 \<\< 6);  
    if (toggle) byte \|= (1 \<\< 5);  
    return byte;  
  }

  /\*\*  
   \* Segments a payload into Cyphal/CAN frames (Classic CAN max 8 bytes, MTU payload 7 bytes \+ tail)  
   \*/  
  segmentPayload(subjectId, payloadBytes, mtu \= 8, priority \= CYPHAL\_PRIORITIES.NOMINAL) {  
    const canId \= CyphalEngine.encodeMessageCanId(priority, subjectId, this.nodeId);  
    const transferId \= this.getNextTransferId();  
    const frames \= \[\];

    const maxPayloadPerFrame \= mtu \- 1; // 1 byte reserved for Tail Byte

    if (payloadBytes.length \<= maxPayloadPerFrame) {  
      // Single-frame transfer (S=1, E=1, Toggle=1 for Classic CAN single frame)  
      const tailByte \= CyphalEngine.createTailByte(true, true, true, transferId);  
      const frameData \= \[...payloadBytes, tailByte\];  
      frames.push({  
        canId: canId,  
        hexCanId: '0x' \+ canId.toString(16).toUpperCase().padStart(8, '0'),  
        frameIndex: 0,  
        isSingleFrame: true,  
        data: frameData,  
        hexDump: frameData.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
      });  
    } else {  
      // Multi-frame transfer requires prepending CRC-16  
      const crc \= CyphalEngine.calculateCrc16(payloadBytes);  
      const fullTransfer \= \[...payloadBytes, crc & 0xFF, (crc \>\> 8\) & 0xFF\];

      let offset \= 0;  
      let frameIndex \= 0;  
      let toggle \= true;

      while (offset \< fullTransfer.length) {  
        const isStart \= (offset \=== 0);  
        const remaining \= fullTransfer.length \- offset;  
        const chunkSize \= Math.min(remaining, maxPayloadPerFrame);  
        const isEnd \= (offset \+ chunkSize \>= fullTransfer.length);

        const chunk \= fullTransfer.slice(offset, offset \+ chunkSize);  
        const tailByte \= CyphalEngine.createTailByte(isStart, isEnd, toggle, transferId);  
        const frameData \= \[...chunk, tailByte\];

        frames.push({  
          canId: canId,  
          hexCanId: '0x' \+ canId.toString(16).toUpperCase().padStart(8, '0'),  
          frameIndex: frameIndex,  
          isSingleFrame: false,  
          isStart: isStart,  
          isEnd: isEnd,  
          toggle: toggle,  
          transferId: transferId,  
          data: frameData,  
          hexDump: frameData.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
        });

        offset \+= chunkSize;  
        frameIndex++;  
        toggle \= \!toggle;  
      }  
    }

    return {  
      canId: canId,  
      subjectId: subjectId,  
      nodeId: this.nodeId,  
      transferId: transferId,  
      totalFrames: frames.length,  
      frames: frames  
    };  
  }

  /\*\*  
   \* Deterministically maps Subject ID to Cyphal/UDP Multicast Address  
   \* OpenCyphal UDP standard: 239.0.X.Y where X/Y are derived from subject ID  
   \*/  
  static getUdpMulticastAddress(subjectId) {  
    const high \= (subjectId \>\> 8\) & 0xFF;  
    const low \= subjectId & 0xFF;  
    const ip \= \`239.0.\${high}.\${low}\`;  
    const port \= 16384; // Standard Cyphal/UDP Subject port  
    return { ip: ip, port: port };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= { CyphalEngine, CYPHAL\_PRIORITIES };  
}  
