# Oak
> Oak (Off-heap Allocated Keys) is a scalable, concurrent, in-memory Key Value (KV) map.

Oak is a concurrent Key-Value Map that may keep all keys and values off-heap. This enables working with bigger heap sizes than JVM's managed heap.
Oak implements an API similar to the industry standard Java8 ConcurrentNavigableMap API. It provides strong (atomic) semantics for read, write, and read-modify-write, as well as (non-atomic) range query (scan) operations, both forward and backward.
Oak is optimized for big keys and values, in particular, for incremental maintenance of objects (update in-place).
It is faster and scales better with additional CPU cores than the popular Java ConcurrentNavigableMap [ConcurrentSkipListMap](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentSkipListMap.html).

## Why Oak?
1. Oak provides great performance: it employs fine-grain synchronization, and thus scales well with numbers of threads; it also achieves cache-friendliness by avoiding memory fragmentation. [Performance evaluation](https://git.ouroath.com/anastas/oak/wiki/Performance).
2. Oak takes keys and the data off-heap, and thus allows working with a huge heap (RAM) -- even more than 50G -- without JVM GC overheads.
	- To support off-heap, Oak has embedded, efficient, epoch-based memory management that mostly eliminates JVM GC overheads.
4. Oak provides a rich API for **atomic** accesses to data. For example, Oak supports atomic compute() -- in place computations on existing keys -- whereas the current Java ConcurrentSkipListMap implementation does not guarantee the atomicity of compute(). Oak’s update operations (such as put and compute) take user-provided lambda functions for easy integration in diverse use cases.
5. Descending Scans: Oak expedites descending scans without additional complexity. In our experiments, Oak’s descending scans are 4.8x faster than ConcurrentSkipListMap’s, and perform similarly to their ascending counterparts. [Performance evaluation](https://git.ouroath.com/anastas/oak/wiki/Performance).

## Table of Contents

- [Background](#background)
- [Install](#install)
- [Builder](#builder)
- [API](#api)
- [Usage](#usage)
- [Transformations](#transformations)
- [Contribute](#contribute)
- [License](#license)

## Background

### Design Points
- Oak consists of an on-heap index to off-heap keys and values. Oak's index is structured as a list of contiguous chunks of memory; this speeds up searches through the index due to access locality and cache-friendliness. Read more about [Oak design](https://git.ouroath.com/anastas/oak/wiki/Design).
- Oak's keys and the values are copied and stored in self-managed off-heap byte arrays.

### Design Requirements
In order to efficiently manage its content, Oak requires that the user define two auxiliary tools: a Serializer and a Comparator; both are passed during construction.
1. *Serializer:* Both keys and values need to provide a (1) serializer, (2) deserializer, and (3) serialized size calculator. All three are parts of [Serializer](#serializer).
	- For boosting performance, Oak allocates space for a given key/value and then uses the given serializer to write the key/value directly to the allocated space. Oak uses the appropriate size calculator to deduce the amount of space to be allocated. Note that both keys and the values are variable-sized.
2. *Comparator:* In order to compare the internally-kept serialized keys with the deserialized key provided by the API, Oak requires a (key) comparator. The comparator compares two keys, each of which may be provided either as a deserialized object or as a serialized one, determining whether they are equal, and if not, which is bigger.

## Install
Oak is a library. After downloading Oak, compile it using `mvn install package` to compile and install. Then update your project's pom.xml file dependencies, as follows:
```
  <dependency>
      <groupId>oak</groupId>
      <artifactId>oak</artifactId>
      <version>1.0-SNAPSHOT</version>
  </dependency>
```
Finally, import the relevant classes and use Oak according to the description below. 

## Builder

In order to build Oak the user should first create the builder, and then use it to construct Oak:.
```java
OakMapBuilder<K,V> builder = ... \\ create a builder; details provided below
OakMap<K,V> oak = builder.build();
```

Oak requires multiple parameters to be defined for the builder, as shall be explained below.
When constructing off-heap Oak, the memory capacity (per Oak instance) needs to be specified. Oak allocates the off-heap memory with the requested capacity at construction time, and later manages this memory.

### Serializer
As explained above, OakMap<K,V> is given key 'K' and value 'V', which are requested to come with a serializer, deserializer and size calculator. Oak user is requested to implement the following interface that can be found in the Oak project.

```java
public interface Serializer<T> {

  // serializes the object
  void serialize(T source, ByteBuffer targetBuffer);

  // deserializes the given byte buffer
  T deserialize(ByteBuffer byteBuffer);

  // returns the number of bytes needed for serializing the given object
  int calculateSize(T object);
}
```

This is how to create those classes in your code:

```java
public class OakKeySerializerImplementation implements Serializer<K>
{...}

public class OakValueSerializerImplementation implements Serializer<V>
{...}
```

### Minimal Key
Oak requires a minimal key that can represent negative infinity according in the user-defined comparision among the keys. The requested minimal key is of type 'K', and is considered by the given comparator to be smaller than every other key (serialized or not). The minimal key is passed as a parameter during builder creation.

### Comparator
After a Key-Value pair is inserted into Oak, it is kept in a serialized (buffered) state. However, Oak's API gets the input key as an object, the serialization of which is deferred until it proves to be required.
Thus, while searching through the map, Oak might compare between keys in their Object and Serialized modes. Oak provides the following interface for such a comparator:
```java
public interface OakComparator<K> {

  int compareKeys(K key1, K key2);

  int compareSerializedKeys(ByteBuffer serializedKey1, ByteBuffer serializedKey2);

  int compareSerializedKeyAndKey(ByteBuffer serializedKey, K key);
}
```

This is how to create the comparator class in your code:

```java
public class OakKeyComparatorImplementation implements OakComparator<K>
{...}
```

### Builder
We provide an example how to create OakMapBuilder and OakMap. For a more comprehensive code example please refer to the [Usage](#usage) section.

```java
OakMapBuilder<K,V> builder = new OakMapBuilder()
            .setKeySerializer(new OakKeySerializerImplementation(...))
            .setValueSerializer(new OakValueSerializerImplementation(...))
            .setMinKey(...)
            .setKeysComparator(new OakKeyComparatorImplementation(...))
            .setMemoryCapacity(...);

OakMap<K,V> oak = builder.build();
```

## API

### OakMap Methods
You are welcome to take a look on the Oak's [full API](https://git.ouroath.com/anastas/oak/wiki/Full-API). 
Oak's API is similar to the ConcurrentNavigableMap API; some non-standard API methods and special cases are discussed below.

### OakBuffers
Oak provides two types of memory buffers: *OakRBuffer* (read-only) and *OakWBuffer* (read and write). These buffers support the standard Java API for read-only Java ByteBuffers and writable Java ByteBuffers, respectively.

Oak buffers allow the user direct access to the underlying serialized key-value pairs, without needing to worry about  concurrent accesses and memory management. This access reduces unnecessary copies and deserialization of the underlying mappings.
Note, however, that since Oak's get method avoids copying the value and instead returns access to the same underlying memory buffer that compute operations update in-place, the reader may encounter different values -- and even value deletions -- when accessing the buffer returned from get multiple times. This is of course normal behavior for a concurrent map that avoids copying.

An OakRBuffer can represent either a key or a value. The OakRBuffer's user can use the standard interface of a *read-only* ByteBuffer, for example, `int getInt(int index)`, `char getChar(int index)`, `limit()`, etc. Notice that ConcurrentModificationException can be thrown as a result of any OakRBuffer method in case the mapping is concurrently deleted.

### Notes on data retrieval
1. For better performance of data retrieval, Oak supplies an OakBufferView of the OakMap. The OakBufferView provides the following four methods for data retrieval, whose result is presented as an OakRBuffer:
	- `OakRBuffer get(K key)`
	- `CloseableIterator<OakRBuffer> valuesIterator()`
	- `CloseableIterator<Map.Entry<OakRBuffer, OakRBuffer>> entriesIterator()`
	- `CloseableIterator<OakRBuffer> keysIterator()`
2. Without the OakBufferView, OakMap's data can be directly retrieved via the following four methods:
	- `V get(K key)`
	- `CloseableIterator<V> valuesIterator()`
	- `CloseableIterator<Map.Entry<K, V>> entriesIterator()`
	- `CloseableIterator<K> keysIterator()`
	
	However, these direct methods return keys and/or values as Objects by applying deseriliazation (copy). This is costly,  and we strongly advice to use OakBufferView or OakTransformView to operate directly on the internal data representation.
3. For further understanding of data retrieval via OakTransformView, please refer to the [Transformations](#transformations) section.

### Notes on data ingestion
1. Data can be ingested and updated via the following five methods:
 	- `void put(K key, V value)`
 	- `boolean putIfAbsent(K key, V value)`
 	- `void remove(K key)`
 	- `boolean computeIfPresent(K key, Consumer<OakWBuffer> computer)`
 	- `void putIfAbsentComputeIfPresent(K key, V value, Consumer<OakWBuffer> computer)`
2. In contrast to the ConcurrentNavigableMap API, `void put(K key, V value)` does not return the value previously associated with the key, if key existed. Likewise, `void remove(K key)` does not return a boolean indicating whether key was actually deleted, if key existed.
3. `boolean computeIfPresent(K key, Consumer<OakWBuffer> computer)` gets the user-defined computer function. The computer is invoked in case the key exists.
The computer is provided with OakWBuffer, representing the serialized value associated with the key. The computer's effect is atomic, meaning that either all updates are seen by concurrent readers, or none are.
The compute functionality offers the Oak user an efficient zero-copy update-in-place, which allows Oak users to focus on business logic without dealing with the hard problems that data layout and concurrency control present.
5. Oak additionally supports an atomic `void putIfAbsentComputeIfPresent(K key, V value, Consumer<OakWBuffer> computer)` interface, (which is not part of ConcurrentNavigableMap).
This API looks for a key. If the key does not exist, it adds a new Serialized key --> Serialized value mapping. Otherwise, the value associated with the key is updated with computer(old value). This interface works concurrently with other updates and requires only one search traversal.

## Usage

An Integer to Integer build example can be seen in [Code Examples](https://git.ouroath.com/anastas/oak/wiki/Code-Examples). Here we illustrate individual operations.

### Code Examples

##### Simple Put and Get
```java
oak.put(Integer(10),Integer(100));
Integer i = oak.get(Integer(10));
```

##### PutIfAbsent
```java
boolean res = oak.putIfAbsent(Integer(11),Integer(110));
```

##### Remove
```java
oak.remove(Integer(11));
```

##### Get OakRBuffer
```java
OakBufferView oakView = oak.createBufferView();
OakRBuffer buffer = oakView.get(Integer(10));
if(buffer != null) {
    try {
        int get = buffer.getInt(0);
    } catch (ConcurrentModificationException e){
    }
}
```

#### Compute
```java
Consumer<OakWBuffer> func = buf -> {
    if (buf.getInt(0) == 1) {
        buf.putInt(1);
        buf.putInt(1);
    }
};
oak.computeIfPresent(Integer(10), func);
```

##### Simple Iterator
```java
try (CloseableIterator<Integer> iterator = oak.keysIterator()) {
    while (iter.hasNext()) {
        Integer i = iter.next();
    }
}
```

##### Simple Descending Iterator
```java
try (CloseableIterator<Integer, Integer> iter = oak.descendingMap().entriesIterator()) {
    while (iter.hasNext()) {
        Map.Entry<Integer, Integer> e = iter.next();
    }
}
```

##### Simple Range Iterator
```java
Integer from = Integer(4);
Integer to = Integer(6);

OakMap sub = oak.subMap(from, false, to, true);
try (CloseableIterator<Integer>  iter = sub.valuesIterator()) {
    while (iter.hasNext()) {
        Integer i = iter.next();
    }
}
```

## Transformations

In addition to the OakBufferView explained above, Oak provides the OakTransformView, which allows manipulating ByteBuffers instead of OakRBuffers. This view might be useful for directly retrieving modified (transformed) data from the OakMap. Transform view is created via `OakTransformView createTransformView(Function<Map.Entry<ByteBuffer, ByteBuffer>, T> transformer)`.
It requires a transform function `Function<Map.Entry<ByteBuffer, ByteBuffer>, T> transformer` that transforms key-value pairs given as **read-only** ByteBuffers into arbitrary `T` objects. The first ByteBuffer parameter (of the Entry) is the key and the second is the value. The API of OakTransformView is the same as that of OakBufferView, except that the return value type is `T`; namely:
	- `T get(K key)`,
	- `CloseableIterator<T> valuesIterator()`,
	- `CloseableIterator<Map.Entry<T, T>> entriesIterator()`,
	- `CloseableIterator<T> keysIterator()`

### Code example

```java
Function<Map.Entry<ByteBuffer, ByteBuffer>, Integer> func = (e) -> {
    if (e.getKey().getInt(0) == 1) {
        return e.getKey().getInt(0)*e.getValue().getInt(0);
    } else return 0;
};

OakTransformView oakView = oak.createTransformView(func);

try (CloseableIterator<Integer> iter = oakView.entriesIterator()) {
    while (iter.hasNext()) {
        Integer i = iter.next();
    }
}
```

## Contribute

Please refer to [the contributing.md file](Contributing.md) for information about how to get involved. We welcome issues, questions, and pull requests.  


## License

This project is licensed under the terms of the [Apache 2.0](LICENSE-Apache-2.0) open source license.
