# Simple Read-only File Protocol

SRFP is a protocol that allows read-only access to filesystems on a remote host.

Its intended use case is data preservation; a modern client computer can connect to an older server, and access files on any volume that server is capable of reading, without the client needing to be compatible with the physical media or understand the filesystem.

SRFP is a binary client-server protocol, that can sit atop any bidirectional binary stream protocol. Because of the intended use case, it is designed primarily for an uncomplicated server-side implementation; most error control logic takes place on the client.

## Message Format

Messages contain the following data, in the following order.

MessageType (uint8)
: Describes the type of message.

MessageID (uint16)
: In requests, this value starts at zero and increments for each subsequent request. In responses, this value matches the MessageID of the request that triggered the response.

MessageLength (uint16)
: Length of the following message value, in bytes, and not including the checksum.

Message (Variable length)
: The message value. Semantics of this value depend on the MessageType.

Checksum (uint32)
: CRC32 checksum of the entire message, including headers, using the polynomial `0xedb88320` (identical to Ethernet, zip, png and so on).

## Message Types

### DirectoryList

#### 0x01 Request

Absolute path to the directory to list, with the NUL byte (`\0`) as the directory separator, but no trailing NUL byte.

If the request value is empty, list the root directory.

#### 0x81 Response

The names of all files in that folder, separated by NUL bytes.

#### Examples

The following are example request and response bodies from a system running DOS.

| Request body | Response body |
|--------------|---------------|
| `''` | `'A:\0C:\0D:'` |
| `'C:'` | `'FOLDER1\0FILE1.TXT\0FILE2.COM'` |
| `'C:\0FOLDER1'` | `'FOLDER2\0FILE1.TXT\0FILE2.COM'` |

#### Notes

Servers should expose all volumes accessible to them as a single SRFP filesystem. (So, in operating systems without a concept of a 'root directory', the root of the SRFP filesystem should be a list of volumes, as in the DOS example above.)

File and directory names may use any character except a null byte. Clients should anticipate this; clients that expose a virtual filesystem should escape and unescape file names appropriately.

    
### NodeInfo

#### 0x02 Request

The path to a file to get info about, with the NUL byte (`\0`) as the directory separator, but no trailing NUL byte.
    
#### 0x82 Response

The following values, in the following order:

Flags (uint8)
: 0x00 if the node is a folder. 0x01 if it is a file.

Size (uint32)
: Size of the file, in bytes.

CreatedTime (uint32)
: Created time, in seconds since January 1970.

AccessedTime (uint32)
: Accessed time, in seconds since January 1970.

ModifiedTime (uint32)
: Time last modified, in seconds since January 1970.

### FileContents

#### 0x03 Request

The following data, in the following order:

ByteOffset (uint32)
: Offset to start reading from, where 0 is the start of the file.

Length (uint32)
: Number of bytes to read.

Path (variable length)
: Path to the file to read from, with the NUL byte (`\0`) as the directory separator, but no trailing NUL byte.

#### 0x83 Response

Byte stream as read from file.

#### Notes

When implementing a client, consider that all of the requested data is returned in a single response. To avoid lengthy retransmissions in the case of errors, and locking up the communication channel in the case of a multithreaded client, it is a good idea for the client to make multiple, small calls to FileContents.

### Version

#### 0x7F Request

Empty request body.

#### 0xFF Response

The major, minor and bugfix version numbers of the protocol version the server implements, as three successive uint8 values. So, a server implementing version 3.1.4 will return `0x030104`.

### Errors

#### 0x80 Error

Denotes an error message.

Contains one uint8 field, which can have one of the following values. (The names are not part of the protocol, but are provided as a consistent naming scheme for client code.

| Value | Name | Meaning |
|-------|------|---------|
| `0x01` | `DOES_NOT_EXIST` | The file or folder specified in the request does not exist. |
| `0xFF` | `OTHER` | Other error. The server should output descriptive text to the console. |