# Caching

Delta Client uses caching to speed up launches by orders of magnitude. The only thing I cache at the moment are block models because these can take up to 30 seconds to load from the JSON model descriptors with my current method (optimisation attempts are very welcome). To decide which serialization technique to use I created a test project and implemented multiple methods of caching the vanilla block model palette. Below is my interpretation of the [results](#the-numbers);

The first implementation was made using [Flatbuffers](https://github.com/google/flatbuffers) and the other was made using [Protobuf](https://github.com/protocolbuffers/protobuf). In release builds, Flatbuffers was faster at deserialization but slower than serialization. In this use case, deserialization performance is really the only thing we care about. To an end user only release build performance matters but when I'm developing the client, debug build performance greatly affects development speed. Interestingly, in debug builds, the fastest at serialization was Flatbuffers (by quite a bit), and Protobuf was over 3 times faster than Flatbuffers at deserialization.

Now that I understood how Flatbuffers worked, I created an implementation that took a similar approach to my neater Protobuf-base implementation. This new Flatbuffers-based implementation was close enough to Protobuf deserialization speed in debug builds, and the serialization speed was pretty much the same as the original Flatbuffers-based implementation. However, in release builds deserialization speeds were now almost twice as slow as Protobuf and serialization speeds were also the slowest out of all three. Somehow, a change that almost doubled debug build performance significantly slowed down release performance?

I have also previously tried out FastBinaryEncoding but it was quite a bit slower at deserialization (more like 2 seconds in release builds). To its credit, it did manage to encode it all into only 13mb which is pretty impressive. But that matters less than performance to me.

## The verdict

For now I will be using Protobufs for caching because they are nicer to use and their performance is more consistent between debug and release builds. They also have the fastest debug build deserialization speed which is what I really care about right now, and the release build serialization speed wasn't too far off the fastest solution in the grand scheme of things. It also has the fastest release build serialization by far. It's only main downfall is that its debug build serialization speed is dismal. Oh, and It also had the smallest serialized size as an added bonus.

## The numbers

### Flatbuffers (with messy code)

Serialized size: 21601644 bytes

| Operation   | Debug   | Release |
| ----------- | ------- | ------- |
| Serialize   | 17676ms | 2185ms  |
| Deserialize | 23190ms | 368ms   |

### Flatbuffers, cleaner code

Serialized size: 21601644 bytes

| Operation   | Debug   | Release |
| ----------- | ------- | ------- |
| Serialize   | 18661ms | 2601ms  |
| Deserialize | 11760ms | 972ms   |

### Protobuf

Serialized size: 16630993 bytes

| Operation   | Debug   | Release |
| ----------- | ------- | ------- |
| Serialize   | 29436ms | 1219ms  |
| Deserialize | 7490ms  | 551ms   |

## Sticky encoding

[Sticky encoding](https://github.com/stickytools/sticky-encoding) looks quite enticing too so I tried it out. All it requires is adding Codable conformance to the types that need to be serialized. However, I tried it out and it's awefully slow (likely because of limitations of Swift's Codable implementation). In debug builds, both serialize and deserialize took over 50 seconds, and in release builds, serialize took around 14 seconds and deserialize took around 12 seconds. It would be nice to use such a nice solution, but it just can't meet our performance requirements.