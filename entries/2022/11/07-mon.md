## Last Week

### Marshalling

I've spent last week, and the weekend (yes I have no life right now), to improve the existing foundations to read/write to the storage. These foundations are currently referred as the "engine". I'm pretty happy about what I have done so far, even if I think I will come back to this in the future.

Concretly, I removed the notion of `Scalar` or `Object` in the engine and embraced a more "rusty" way of thinking about it: traits. Any object can implement two pairs of traits: `Encode`/`Decode` and `Write`/`Read`.

`Encode`/`Decode` represent anything that can be marshalled to a binary representation. In storage terms they can either represent a key or a value. `Encode`/`Decode` have 2 paths each:

- Big endian for keys, so we keep lexical order which is paramount for B/LSM trees,
- Little endian for values, as it's the most ubiquitous encoding out there.

The `Write`/`Read` pair is tied to the storage. It represents anything that can marshalled to one or multiple key-values. It is typically implemented by more complex objects that span across multiple key-values. It's worth mentioning that any `Encode` is automatically a `Write` to one key-value, as any `Decode` is a `Read` from one key-value. It helps the engine to make no distinction between simple values (scalars) and complex values (objects).

These two pairs of traits give enough flexibility to pretty much store anything we like: scalars or objects, using the layout we want.

### Nested objects

That being said, I hit one issue: how to handle multiple nested objects? Without entering in two much details, it all boils down to isolate an object's key at the current level of nesting. The solution I found is to let the implementor of `Write`/`Read` use what I call a "sentinel key". I borrowed the idea from CockroachDB, but I'm pretty sure it's used by various other DBs out there.

A sentinel key is basically a marker for the current object that is garanted to end with the actual object key (and not one of it's nested fields). The sentinel key is written at the currenlty active key in the `Writer` with an optional value, which is `0x00` by default. The optional value can be used by the implementor to store any kind of metadata at the object level.
Later on, the `Reader` can then grab this key and and return it, so the parent object know what is the key of the current child. This is particularly helpful if the parent doesn't know the key of the current child in advance (ie. `Map`).

The `Map` object indirectly relies on children using sentinel keys to derive the actual store key fragment that identifies the map key. Same thing goes for the `Array` that can check the index of the current entry if it needs to.

## This Week

This week I'm going to focus on slightly higher-level components of GoatDB and start putting my hands in the actual valuable stuff!

One of the main feature I want out of GoatDB is versioning and authorization, at the key level! Versioning will allow higher-level features such as dynamic schemas, schemas/records versioning, collaboration, fast rollbacks, etc... Authorization will allow the user to directly leverage the database authorization system instead of developing an abstraction on top of it.

I would like to tackle versioning this week, or at least come up with a PoC.

Several things needs to be done:

- Store the versioning metadata (eg. branches, versions, ...) direclty in the store. This will likely involve creating objects that are loaded/saved by the engine automatically. I already have designed how versioning will work in theory, so it's just about creating these objects.
- Have access to the current version of the database anywhere. My idea is that a version is picked when the user "connects" to the database. That version could just be stored as part of a `Session` object that gets passed everywhere needed.
- Have access to the versioning metadata itself. This would be necessary so the multiple parts involved in versioning can actually know which versions exists, if a version matches the current version, etc... I'm leaning towards a global set of functions backed by a singleton object held in a `LazyLock`.
- Find a way to optionally version keys that is transparent for objects. By that I mean that an object implementing `Write`/`Read` shouldn't care about versioning at all. Objects should retreive the appropriate keys for the current version of the database transparently. Versioning must happen at a lower level so we can leverage this feature anywhere. It will very likely be at the `Writer`/`Reader` level, but I need to think more about that.

## Learnings

I've learned to organize my code in a more "rusty" way. By that I mean actually separate the data from its behavior. Let me explain myself. I previously had a `Key` struct which was basically wrapping a `Vec<u8>` which specific behavior for a key. It ended up being a pain in the ass to convert things to a `Key` everywhere. It added more rigidity than necessary to build a key from anything that can be converted to `Vec<u8>` or `&[u8]`.

What I ended up doing is to accept a `AsRef<[u8]>` in place of a key. I created two aliases: `Bytes` for `[u8]` and `BytesBuf` for `Vec<u8>`. Now I can accept as a key, anything that converts to bytes, that's it. This is way more flexible. Ok, but now,  what do I do if I really need key-specific operations? Well, because a key is just a bytes buffer, I simply have an auto-trait `Key` that is implemented for `Bytes` and `BytesBuf` so I can operate of these types directly. I simply pull this trait when I need to some key manipulations.

## Oh la vache!

I finally understood the whole point of `Cow` 🙏. I needed a way for `Encode`/`Decode` to return either `Bytes` or `BytesBuf`, depending on the implementor type. Why that? Because if you take a `String` for example, it would be stupid to clone it to a `BytesBuf` as its internal representation is already a `BytesBuf`. However for a `u32`, you actually need to create a new value out of it as it's internal representation is more specific. So I needed a way to return either a borrowed value or an owned value. That's exactly what `Cow` is made for.

`Cow` can hold a borrowed `Bytes` value for a `String` returned by `String::as_bytes`, but it can also hold an owned `BytesBuf` value for a `u32` returned by `u32::to_le_bytes`. This saves uncessary costly clones when the underlying type is just an abstraction over an already existing `ByteBuf`.
