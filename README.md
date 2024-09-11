# opnet-storage

Storage abstraction for OP_NET storage slots with support for unstructured.

## Install

```sh
yarn add https://github.com/youngslohmlife/opnet-storage
```

## Usage

### StorageSlot

StorageSlot is an abstraction over a key in a key-value pair. It is a wrapper over ArrayBuffer but has methods to add more segments to the key to form a new StorageSlot, as well as methods to set an ArrayBuffer value or any primitive value.

```js
abstract class StorageSlot {
  static wrap(pointer: ArrayBuffer): StorageSlot;
  static for(keyword: string): 
  select(key: ArrayBuffer): StorageSlot;
  keyword(key: string): StorageSlot;
  set(v: u256): void;
  get(): u256;
  lengthKey(): StorageSlot;
  length(): u32;
  getList(): Array<u256>;
  extend(): StorageSlot;
  selectIndex(index: u32): StorageSlot;
  nullify(): void;
  append(v: u256);
}
```

##### StorageSlot.wrap(pointer: ArrayBuffer)

Wraps an ArrayBuffer as an StorageSlot.

```js
import { primitiveToBuffer } from "metashrew-as/assembly/utils/utils";
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  const pointer = StorageSlot.wrap(primitiveToBuffer(<u32>0x01010101)); // creates an StorageSlot for 0x01010101
  pointer.set(u256.from(0x20202020));
}
```

##### StorageSlot.for(keyword: string)

Converts `keyword` to an ArrayBuffer then wraps the ArrayBuffer as an StorageSlot. Useful for creating StorageSlot.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  const pointer = StorageSlot.for("/some-table/");
  pointer.set(u256.from(0x12121212));
}
```

##### StorageSlot#select(key: ArrayBuffer): StorageSlot

Grows the StorageSlot by appending another byte slice to the end of the ArrayBuffer it wraps.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";
import { sha256 } from "fast-sha256-as/assembly/sha256";

export function _start(): void {
  const pointer = StorageSlot.for("/value/by-txid/").select(sha256(String.UTF8.encode("test")));
  pointer.set(u256.from(0x0101010101));
}
```

##### StorageSlot#keyword(word: string): StorageSlot

Encodes `word` as UTF8 bytes and uses the ArrayBuffer to grow the key represented by the StorageSlot, returning a new StorageSlot.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  StorageSlot.for("/txid/number/").select(new ArrayBuffer(1)).keyword("/tag").set(u256.from(111));
}
```

##### StorageSlot#get(): u256

Gets the value stored at the key represented by the StorageSlot. The result is an ArrayBuffer type. Values that are set with StorageSlot are cached in-memory, but when reading out of the cache, the value is copied, so it can be mutated without affecting the call to `_flush()` at the end of the program run.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";
import { Box } from "opnet-storage/assembly/utils/box";
import { console } from "opnet-storage/assembly/utils/logging";

export function _start(): void {
  const pointer = StorageSlot.for("/txid/last");
  console.log(Box.from(pointer.get()).toHexString());
}
```

##### StorageSlot#set(v: u256): void

Sets the value in the key-value database corresponding to the key represented by the StorageSlot. When inserting into the cache with `set` it will copy the bytes of its argument, so the value can potentially be mutated afterwards.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";
import { u256 } from "as-bignum/assembly";

export function _start(): void {
  StorageSlot.for("/txid/last").set(u256.from(111));
}
```

##### StorageSlot#lengthKey(): StorageSlot

List operation:

Appends `"/length"` to the end of the key and returns a new StorageSlot. Useful for doing operations on the value representing the length of a list, to be used with `append` or `appendValue`.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  const length = StorageSlot.for("/txid/list").lengthKey().get().toU32();
  console.log(length.toString());
}
```

##### StorageSlot#length(): u32

List operation:

Calls `StorageSlot#lengthKey()` then `StorageSlot#getValue<u32>()` on the key it returns. Ultimately will produce the length of the list stored at the desired key.

```js
// same program as above, in effect
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  const length = StorageSlot.for("/txid/list").length();
  console.log(length.toString(10));
}
```


##### StorageSlot#getList(): Array<u256>

List operation:

Gets the list of values at the desired key. First it fetches the length of the list then called `StorageSlot#selectIndex(v: u32)` for each key, then gets the u256 stored at that key.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";
import { u256 } from "as-bignum/assembly";

export function _start(): void {
  const values = StorageSlot.for("/txid/list").getList();
  values.forEach((v: u256, i: i32, ary: Array<u256>) => {
    console.log(v.toString());
  });
}
```
##### StorageSlot#extend(): StorageSlot

List operation:

Increases the length of the list by 1. Returns the StorageSlot at the newly created slot in the list.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";
import { u256 } from "as-bignum/assembly";

export function _start(): void {
  const pointer = StorageSlot.for("/txid/values");
  pointer.extend().set(u256.from(1));
}
```

##### StorageSlot#selectIndex(index: u32): StorageSlot

List operation:

Selects the value in a list at `index`. Returns the StorageSlot at that slot.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  const pointer = StorageSlot.for("/txid/values");
  const first = StorageSlot.for("/txid/values").selectIndex(0);
  first.get().toString();
}
```

##### StorageSlot#nullify(): void

Deletes the value at the key represented by the StorageSlot.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  const pointer = StorageSlot.for("/some-key");
  pointer.nullify() // deletes the value at "/some-key"
}
```

##### StorageSlot#append(v: ArrayBuffer): void

List operation:

Appends an ArrayBuffer to the list of values at the key represented by the StorageSlot.

```js
import { StorageSlot } from "opnet-storage/assembly/StorageSlot";

export function _start(): void {
  const pointer = StorageSlot.for("/some-list");
  pointer.append(String.UTF8.encode("some value"));
  pointer.append(String.UTF8.encode("other value"));
  console.log(String.UTF8.decode(pointer.pop())) // logs "other value";
}
```

### StorageLayout

Simple class to track unique storage slots. Only contains a next() method which returns the next available storage slot.

Should be used within a constructor to set up the layout of storage slots by pointer.

```js
class StorageLayout {
  next(): u16
}
```

Examples:

```js

class MyContract extends OP_NET {
  public _balanceOf: StorageSlot;
  public _allowance: StorageSlot;
  constructor() {
    const layout = new StorageLayout();
    this._balanceOf = StorageSlot.at(layout.next());
    this_allowance = StorageSlot.at(layout.next());
  }
  increaseBalance(address: u256, amount: u256): BytesWriter {
    const slot = this._balanceOf.select(changetype<Uint8Array>(address.toBytes()).buffer);
    slot.set(slot.get() + amount); // use SafeMath here instead
  }
}
```


## Author

FREEJEFFREY

## License

MIT



