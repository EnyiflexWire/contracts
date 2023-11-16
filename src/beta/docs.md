# Lists

A list is comprised of list records, which can be associated with a list of strings called "tags".

There are four operations supported on lists:

| Code | Operation     |
| ---- | ------------- |
| 1    | Add record    |
| 2    | Remove record |
| 3    | Tag record    |
| 4    | Untag record  |

Other operations may be added in the future.

## List Record

A `ListRecord` is a fundamental data structure representing a record in a list. Each `ListRecord` consists of the following components:

- `version`: A `uint8` representing the version of the `ListRecord`. This is used to ensure compatibility and facilitate future upgrades.
- `recordType`: A `uint8` indicating the type of record. This serves as an identifier for the kind of data the record holds.
- `data`: A `bytes` array containing the actual data of the record. The structure of this data depends on the `recordType`.

```solidity
struct ListRecord {
    uint8 version;
    uint8 recordType;
    bytes data;
}
```

### Record Types

There is only one record type defined at this time:

| Type | Description | Data            |
| ---- | ----------- | --------------- |
| 1    | Raw address | 20-byte address |

To illustrate the design, however, consider a few hypothetical list record types:

- a subscription to another EFP List, where the `data` field would contain the 32-byte token ID of the corresponding EFP NFT.
- an ERC-721 NFT token, where the `data` field would contain the 20-byte address of the ERC-721 contract, and the 32-byte token ID.
- an ERC-1155 token, where the `data` field would contain the 20-byte address of the ERC-1155 contract, the 32-byte token ID (exclude token amount).
- an ENS name, where the `data` field would contain the 32-byte hash of the ENS name OR possibly the normalized string of the ENS name.

## Tag

A `Tag` is a string associated with a `ListRecord` in a list. A `ListRecord` can have multiple tags associated with it. A `Tag` is represented as a string.

### Normalization

Tags are normalized by converting them to lowercase and removing leading and trailing whitespace.

Tags should be normalized before they are encoded into a `ListOp`.

## ListOp

A `ListOp` is a structure used to encapsulate an operation to be performed on a list. It includes the following fields:

- `version`: A `uint8` representing the version of the `ListOp`. This is used to ensure compatibility and facilitate future upgrades.
- `code`: A `uint8` indicating the operation code. This defines the action to be taken using the `ListOp`.
- `data`: A `bytes` array which holds the operation-specific data. For instance, if the operation involves adding a `ListRecord`, this field would contain the encoded `ListRecord`.

```solidity
struct ListOp {
    uint8 version;
    uint8 code;
    bytes data;
}
```

### Operation Codes

There are four operations defined at this time:

| Code | Operation     | data                                 |
| ---- | ------------- | ------------------------------------ |
| 1    | Add record    | Encoded `ListRecord`                 |
| 2    | Remove record | Encoded `ListRecord`                 |
| 3    | Tag record    | Encoded `ListRecord` followed by tag |
| 4    | Untag record  | Encoded `ListRecord` followed by tag |

## Encoding

The encoding of a `ListOp` is designed to be flexible, accommodating various types of operations and their corresponding data structures. The encoded form looks as follows:

| Byte(s) | Description                     |
| ------- | ------------------------------- |
| 0       | `ListOp` version (1 byte)       |
| 1       | Operation code (1 byte)         |
| 2 - N   | Encoded operation-specific data |

The `2 - N` byte range is variable and depends on the operation being performed.

### Example - Add Record

The following is an example of an encoded `ListOp` for adding a `ListRecord` of type 1 (raw address) to a list:

| Byte(s) | Description                   | Value                                      |
| ------- | ----------------------------- | ------------------------------------------ |
| 0       | `ListOp` version (1 byte)     | 0x01                                       |
| 1       | Operation code (1 byte)       | 0x01                                       |
| 2       | `ListRecord` version (1 byte) | 0x01                                       |
| 3       | `ListRecord` type (1 byte)    | 0x01                                       |
| 4 - 23  | `ListRecord` data (20 bytes)  | 0x00000000000000000000000000000000DeaDBeef |

### Example - Remove Record

The following is an example of an encoded `ListOp` for removing a `ListRecord` of type 1 (raw address) from a list:

| Byte(s) | Description                   | Value                                      |
| ------- | ----------------------------- | ------------------------------------------ |
| 0       | `ListOp` version (1 byte)     | 0x01                                       |
| 1       | Operation code (1 byte)       | 0x02                                       |
| 2       | `ListRecord` version (1 byte) | 0x01                                       |
| 3       | `ListRecord` type (1 byte)    | 0x01                                       |
| 4 - 23  | `ListRecord` data (20 bytes)  | 0x00000000000000000000000000000000DeaDBeef |

### Example - Tag Record

The following is an example of an encoded `ListOp` for tagging a `ListRecord` of type 1 (raw address) in a list:

| Byte(s) | Description                   | Value                                      |
| ------- | ----------------------------- | ------------------------------------------ |
| 0       | `ListOp` version (1 byte)     | 0x01                                       |
| 1       | Operation code (1 byte)       | 0x03                                       |
| 2       | `ListRecord` version (1 byte) | 0x01                                       |
| 3       | `ListRecord` type (1 byte)    | 0x01                                       |
| 4 - 23  | `ListRecord` data (20 bytes)  | 0x00000000000000000000000000000000DeaDBeef |
| 24 - N  | Tag (variable) (UTF-8)        | 0x746167 ("tag")                           |

The tag should be encoded as UTF-8.

### Example - Untag Record

The following is an example of an encoded `ListOp` for untagging a `ListRecord` of type 1 (raw address) in a list:

| Byte(s) | Description                   | Value                                      |
| ------- | ----------------------------- | ------------------------------------------ |
| 0       | `ListOp` version (1 byte)     | 0x01                                       |
| 1       | Operation code (1 byte)       | 0x04                                       |
| 2       | `ListRecord` version (1 byte) | 0x01                                       |
| 3       | `ListRecord` type (1 byte)    | 0x01                                       |
| 4 - 23  | `ListRecord` data (20 bytes)  | 0x00000000000000000000000000000000DeaDBeef |
| 24 - N  | Tag (variable) (UTF-8)        | 0x746167 ("tag")                           |

The tag should be encoded as UTF-8.

## Social Graph

The social graph is the full set of lists and their associated records and tags.

To construct the social graph, it is sufficient to iterate through all lists and apply the operations in order:

```
for each list in lists:
    for each op in list:
        apply op to social graph
```

### Via Logs

The contract defines a `ListOperation` event as:

```solidity
event ListOperation(uint indexed nonce, bytes op);
```

So, the social graph can be constructed by iterating through the `ListOperation` events emitted by the EFP NFT contract.

### Via Contract Calls

The contract defines four read functions:

- `getListOpCount`: Returns the number of operations in a list.
- `getListOp`: Returns a single operation in a list.
- `getListOpsInRange`: Returns a range of operations in a list.
- `getAllListOps`: Returns all operations in a list.

If you are using a node without gas limits, you can use `getAllListOps` to retrieve all operations in a list. Otherwise, you will need to use `getListOpCount` and `getListOpsInRange` to retrieve the operations in batches.

### Social Graph implementation

A rough implementation of a Social Graph is defined below

```TypeScript
type TokenId = number;

type ListRecord {
    version: number;
    recordType: number;
    data: bytes;
}

type Tag = string;

class LinkedListNode {
    value: ListRecord;
    next: LinkedListNode | null;
    prev: LinkedListNode | null;

    constructor(value: ListRecord) {
        this.value = value;
        this.next = null;
        this.prev = null;
    }
}

class LinkedList {
    head: LinkedListNode | null;
    tail: LinkedListNode | null;

    constructor() {
        this.head = null;
        this.tail = null;
    }

    // O(1) time
    add(record: ListRecord) {
        const newNode = new LinkedListNode(record);
        if (!this.head) {
            this.head = newNode;
            this.tail = newNode;
        } else {
            if (this.tail) {
                this.tail.next = newNode;
                newNode.prev = this.tail;
                this.tail = newNode;
            }
        }
        return newNode; // Return the node for external reference
    }

    // O(1) time
    remove(node: LinkedListNode) {
        if (node.prev) {
            node.prev.next = node.next;
        } else {
            this.head = node.next;
        }
        if (node.next) {
            node.next.prev = node.prev;
        } else {
            this.tail = node.prev;
        }
    }
}

// Social Graph supports:
// O(1) add/remove records via doubly-linked list to maintain order
// O(1) add/remove tags via set
// O(n) get records since we need to iterate through the linked list
// O(1) get tags since we use a set
//
// other tradeoffs are possible but this is a simple implementation shown as an example
class SocialGraph {
    private lists: Map<TokenId, LinkedList>;
    private nodeMap: Map<ListRecord, LinkedListNode>; // To quickly find the node for a given record
    private tags: Map<TokenId, Map<LinkedListNode, Set<Tag>>>;

    constructor() {
        this.lists = new Map();
        this.tags = new Map();
        this.nodeMap = new Map();
    }

    addRecord(listId: TokenId, record: ListRecord): void {
        if (!this.lists.has(listId)) {
            this.lists.set(listId, new LinkedList());
        }
        const node = this.lists.get(listId).add(record);
        this.nodeMap.set(record, node);
    }

    removeRecord(listId: TokenId, record: ListRecord): void {
        if (this.lists.has(listId) && this.nodeMap.has(record)) {
            const node = this.nodeMap.get(record);
            this.lists.get(listId).remove(node);
            this.nodeMap.delete(record);
        }
    }

    tagRecord(listId: TokenId, record: ListRecord, tag: Tag): void {
        if (!this.tags.has(listId)) {
            this.tags.set(listId, new Map());
        }
        const node = this.nodeMap.get(record);
        if (!this.tags.get(listId).has(node)) {
            this.tags.get(listId).set(node, new Set<Tag>());
        }
        this.tags.get(listId).get(node).add(tag);
    }

    untagRecord(listId: TokenId, record: ListRecord, tag: Tag): void {
        const node = this.nodeMap.get(record);
        if (this.tags.has(listId) && this.tags.get(listId).has(node)) {
            this.tags.get(listId).get(node).delete(tag);
        }
    }

    // read-only functions

    // O(n) time
    getRecords(listId: TokenId): ListRecord[] {
        const records: ListRecord[] = [];
        if (this.lists.has(listId)) {
            let node = this.lists.get(listId).head;
            while (node) {
                records.push(node.value);
                node = node.next;
            }
        }
        return records;
    }

    // O(1) time
    getTags(listId: TokenId, record: ListRecord): Set<Tag> {
        const node = this.nodeMap.get(record);
        if (this.tags.has(listId) && this.tags.get(listId).has(node)) {
            return this.tags.get(listId).get(node);
        }
        return new Set<Tag>();
    }
}
```