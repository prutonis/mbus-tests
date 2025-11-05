# mbus-tests
Decoding M-Bus

```
// Setup serial port on PLC
SerialPort.configure(
    baudrate = 2400,
    databits = 8,
    parity = EVEN,
    stopbits = 1
)

// Define function to compute checksum
function computeChecksum(bytes[], startIndex, endIndex):
    sum = 0
    for i from startIndex to endIndex do
        sum += bytes[i]
    return sum mod 256

// Prepare a “request user data (class 2)” long frame to meter address A
function buildRequestFrame(address):
    // According to spec: 68 LL LL 68 C A CI 16
    CI = 0x51  // example: request user data (variable) – from spec / typical usage
    C  = 0x5B  // control field for REQ_UD2
    // Without additional DATA bytes
    frame = []
    frame.append(0x68)
    frame.append(0x03)   // length L = number of bytes after second 68 up to checksum
    frame.append(0x03)   // L repeated
    frame.append(0x68)
    frame.append(C)
    frame.append(address)
    frame.append(CI)
    checksum = computeChecksum(frame, 4, 6)  // from C through CI
    frame.append(checksum)
    frame.append(0x16)
    return frame

// Main logic to poll meter
address = 1  // set your meter’s primary address
requestFrame = buildRequestFrame(address)

// Send request
SerialPort.write(requestFrame)

// Wait and read response (timeout maybe 1000 ms)
response = SerialPort.read(maxBytes = 256, timeout = 1000)
if response.length == 0 then
    log("Failed: no response from meter at addr " + address)
    return
end if

// Basic validation of response according to M-Bus spec
if response[0] != 0x68 then
    log("Invalid start byte")
    return
end if
lengthL = response[1]
if response[2] != lengthL then
    log("Invalid length repetition")
    return
end if
if response[3] != 0x68 then
    log("Missing second start byte")
    return
end if

// Determine where checksum is: at position (4 + lengthL – 1)
checksumIndex = 4 + lengthL - 1
if response[checksumIndex+1] != 0x16 then
    log("Missing stop byte")
    return
end if

// Compute checksum and compare
cs = computeChecksum(response, 4, checksumIndex-1)
if cs != response[checksumIndex] then
    log("Checksum mismatch")
    return
end if

// If valid, parse application data (DATA field)
parseMbusData(response, 7, checksumIndex-1)  // start parsing after CI

function parseMbusData(bytes[], startIndex, endIndex):
    i = startIndex
    while i <= endIndex do
        dataFieldId = bytes[i]
        length      = bytes[i+1]
        // read length bytes of value
        valueBytes = bytes[i+2 .. i+1+length]
        // decode value according to type (BCD, integer, float) – meter manufacturer dependent
        value = decodeValue(valueBytes)
        log("Field " + dataFieldId + " = " + value)
        i = i + 2 + length
    end while
end function
```
