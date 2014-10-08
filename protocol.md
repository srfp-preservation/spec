# Simple Read-only File Protocol

SRFP sits atop any bidirectional stream protocol, such as RS-232. 

## Headers

Request and response headers are identical. They have the following format:

| MessageType (1 byte) | MessageID (3 bytes) |||
|-
| MessageLength (4 bytes) ||||
| Message (variable length) ||||

## LIST_VOLUMES (MessageType 0x01)

### Request

Empty request body.
    
### Response

All drive letters available to read from, as ASCII bytes.

Example:

    ACD

## LIST_DIR (MessageType 0x02)

### Request

Absolute path to the directory to list.

Example:

    C:\\FOLDER

### Response

The names of all files in that folder, separated by NUL bytes.

Example:

    FILE1.TXT\0FILE2.COM\0FILE3.TXT
    
## FILE_INFO (MessageType 0x03)

### Request

The path to a file to get info about.

Example:

    C:\\FOLDER\\FILE1.TXT
    
### Response

Size, created time, accessed time, modified time, all as 32-bit integers. Times are in UNIX time notation.

## READ_FILE (MessageType 0x04)

### Request

The byte offset and length both as 32-bit integers, followed by the path to the file to read.

### Response

Byte stream as read from file.

### Notes

When using READ_FILE, calling applications should consider that all of the requested data is returned in a single response, so requesting large amounts of data in one request could block other threads from performing other operations.