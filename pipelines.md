# System.IO.Pipelines: A new library for high performance IO in .NET

## What is it?

System.IO.Pipelines is a new library that is designed for doing high performance IO in .NET. It's new in .NET Core 2.1 and is a netstandard library that works on all .NET implementations. 

It was born from the work the .NET Core team did to make Kestrel one of the fastest web servers in the industry. What started as an implementation detail inside of Kestrel progressed into a re-usable API that shipped in 2.1 as a first class BCL API (System.IO.Pipelines) available for all .NET developers. 

Today Pipelines powers Kestrel and SignalR and we hope to see it at the center of many networking libraries and components from the .NET community. 

## What problem does it solve? 

Correctly parsing data from a stream or socket is dominated by boilerplate code and has many corner cases; leading to complex code. 
Achieving high performance and being correct; while also dealing this complexity, is unnecessarily hard. Pipelines aims to solve this complexity. [Skip to the pipelines version](#tcp-server-with-systemiopipelines)

## What extra complexity does Streams involve? 

Let's start with a simple problem. We want to write a TCP server that receives line based messages (delimited by \n) from a client. 

### TCP Server with NetworkStream

The typical code you would write in .NET before pipelines looks something like this:

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffer = new byte[1024];
    await stream.ReadAsync(buffer, 0, buffer.Length);
    ProcessLine(buffer);
}
```

This code might work when testing locally but it's broken because the entire message (end of line) may not have been received in a single call to `ReadAsync`. Even worse, we're failing to look at the result of `stream.ReadAsync()` which returns how much data was actually filled into the buffer. This is a common mistake when using `Stream` today. To make this work, we need to buffer the incoming data until we have found a new line.

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffer = new byte[1024];
    var read = 0;
    while (read < buffer.Length)
    {
        var current = await stream.ReadAsync(buffer, read, buffer.Length - read);
        if (current == 0)
        {
            break;
        }
        read += current;
        var lineLength = Array.IndexOf(buffer, (byte)'\n', 0, read);

        if (lineLength >= 0)
        {
            ProcessLine(buffer, 0, lineLength);
            read = 0;
        }
    }
}
```

Once again, this might work in local testing but it's possible that the line is bigger than 1KiB (1024 bytes). So we need to resize the input buffer until we have found a new line.

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffer = new byte[1024];
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining == 0)
        {
            var newBuffer = new byte[buffer.Length * 2];
            Buffer.BlockCopy(buffer, 0, newBuffer, 0, buffer.Length);
            buffer = newBuffer;
            read = 0;
            remaining = buffer.Length;
        }

        var current = await stream.ReadAsync(buffer, read, remaining);
        if (current == 0)
        {
            break;
        }

        read += current;
        var lineLength = Array.IndexOf(buffer, (byte)'\n', 0, read);

        if (lineLength >= 0) 
        {
            ProcessLine(buffer, 0, lineLength);
            read = 0;
        }
    }
}
```

This code works but now we're re-sizing the buffer which causes extra allocations and copies. To avoid this, we can store a list of buffers instead of resizing each time we cross the 1KiB buffer size. We're also re-using the 1KiB buffer until it's completely empty. This means we can end up passing smaller and smaller buffers to `ReadAsync` which will result in more calls into the operating system. To mitigate this, we'll allocate a new buffer when we there's 512 bytes remaining in the buffer.

```C#
async Task AcceptAsync(Socket socket)
{
    const int minimumBufferSize = 512;
    
    var stream = new NetworkStream(socket);
    var buffers = new List<ArraySegment<byte>>();
    var buffer = new byte[1024];
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining < minimumBufferSize)
        {
            buffers.Add(new ArraySegment<byte>(buffer, 0, read));
            buffer = new byte[1024];
            read = 0;
            remaining = buffer.Length;
        }

        var current = await stream.ReadAsync(buffer, read, remaining);
        if (current == 0)
        {
            break;
        }

        read += current;
        var lineLength = Array.IndexOf(buffer, (byte)'\n', 0, read);

        if (lineLength >= 0) 
        {
            buffers.Add(new ArraySegment<byte>(buffer, 0, lineLength));

            ProcessLine(buffers);

            buffers.Clear();
        }
    }
}
```

This code just got much more complicated. We're keeping track the filled up buffers as we're looking for the delimeter. To do this, we're using a `List<ArraySegment<byte>>` here to represent the buffered data while looking for the new line delimeter. As a result, `ProcessLine` now accepts a `List<ArraySegment<byte>>` instead of a `byte[]`, `offset` and `count`. Our parsing logic needs to now handle either a single or multiple buffer segments.

There's another optimization that we need to make before we call this server complete. Right now we have a bunch of heap allocated buffers in a list. We can improve thes allocations by using the new `ArrayPool<T>` (introduced in .NET Core 1.0) to avoid repeated buffer allocations as we're parse more lines from the client. 

```C#
async Task AcceptAsync(Socket socket)
{
    const int minimumBufferSize = 512;
    
    var stream = new NetworkStream(socket);
    var buffers = new List<ArraySegment<byte>>();
    byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining < minimumBufferSize)
        {
            buffers.Add(new ArraySegment<byte>(buffer, 0, read));
            buffer = ArrayPool<byte>.Shared.Rent(1024);
            read = 0;
            remaining = buffer.Length;
        }

        var current = await stream.ReadAsync(buffer, read, remaining);
        if (current == 0)
        {
            break;
        }

        read += current;
        var lineLength = Array.IndexOf(buffer, (byte)'\n', 0, read);

        if (lineLength >= 0) 
        {
            buffers.Add(new ArraySegment<byte>(buffer, 0, lineLength));

            ProcessLine(buffers);

            foreach (var buffer buffers) 
            {
                ArrayPool<byte>.Shared.Return(buffer.Array);
            }

            buffers.Clear();
        }
    }
}
```

Our server now handles partial messages, and it uses pooled memory to reduce overall memory consumption but there are still a couple of problems. The `byte[]` we're using from the `ArrayPool<byte>` are just regular managed arrays. This means whenever we do a `ReadAsync` or `WriteAsync`, those buffers get pinned for the lifetime of the asynchornous operation (in order to interop with the native IO APIs on the operating system). This has performance implications on the garbage collector since pinned memory cannot be moved which can lead to heap fragmentation. Depending on how long the async operations are pending, the pool implementation may need to change. The other issue we need to tackle the throughput. A common pattern used to increase the throughput is to decouple the reading and processing logic. This lets us consume buffers from the `Socket` as they become available without letting the parsing of those buffers stop us from reading more data. This introduces a couple problems though:
- We need 2 loops that run independently of each other. One that reads from the `Socket` and one that parses the buffers.
- We need a way to signal the parsing logic when data becomes available.
- We need to decide what happens if the loop reading from the `Socket` is "too fast". We need a way to throttle the reading loop if the parsing logic can't keep up. This is commonly referred to as "flow control" or "back pressure".
- We need to make sure things are thread safe. We're now sharing a set of buffers between the reading loop and the parsing loop and those run independently on different threads.
- The memory management logic is now spread across 2 different pieces of code, the code that rents from the buffer pool is reading from the socket and the code that returns from the buffer pool is the parsing logic.
- We need to be extremely careful with how we return buffers after the parsing logic is done with them. If we're not careful, it's possible that we return a buffer that's still being written to by the `Socket` reading logic.

The complexity has gone through the roof (and we haven't even covered all of the cases). High performance networking usually means writing very complex code in order to eke out more performance from the system. The goal of System.IO.Pipelines is to make writing this type of code easier.

### TCP server with System.IO.Pipelines

Let's take a look at what this example looks like with System.IO.Pipelines.

```C#
async Task AcceptAsync(Socket socket)
{
    var pipe = new Pipe();
    Task writing = ReadFromSocketAsync(pipe.Writer);
    Task reading = ReadFromPipeAsync(pipe.Reader);

    async Task ReadFromSocketAsync(PipeWriter writer)
    {
        const int minimumBufferSize = 512;
        
        while (true)
        {
            Memory<byte> memory = writer.GetMemory(minimumBufferSize);
            int read = await socket.ReceiveAsync(memory, SocketFlags.None);
            if (read == 0)
            {
                break;
            }
            writer.Advance(read);
            FlushResult result = await writer.FlushAync();
            
            if (result.IsCompleted) 
            {
                break;
            }
        }
        writer.Complete();
    }

    async Task ReadFromPipeAsync(PipeReader reader)
    {
        while (true)
        {
            ReadResult result = await reader.ReadAsync();

            ReadOnlySequence<byte> buffer = result.Buffer;
            SequencePosition? position = buffer.PositionOf((byte)'\n');

            if (position != null)
            {
                ProcessLine(buffer.Slice(0, position.Value));
                buffer = buffer.Slice(buffer.GetPosition(1, position.Value));
            }

            reader.AdvanceTo(buffer.Start);

            if (result.IsCompleted)
            {
                break;
            }
        }

        reader.Complete();
    }

    await Task.WhenAll(reading, writing);
}
```

The pipelines version of our line reader has 2 loops:
- One loop reads from the `Socket` and writes into the `PipeWriter`.
- The other loop reads from the `PipeReader` and parses incoming lines.

Unlike the original examples, there are no explicit buffers allocated anywhere. This is one of pipelines' core features. All buffer management is delegated to the `PipeReader`/`PipeWriter` implementations. This makes it easier for consuming code to focus solely on the business logic instead of complex buffer management. In the first loop, we first call `PipeWriter.GetMemory(int)` to get some memory from the underlying writer then we call `PipeWriter.Advance(int)` to tell the `PipeWriter` how much data we actually wrote to the buffer. We then call `PipeWriter.FlushAsync()` to make the data available to the `PipeReader`.

In the second loop, we're consuming the buffers written by the `PipeWriter` which ultimately comes from the `Socket`. When the call to `PipeReader.ReadAsync()` returns, we get a `ReadResult` which contains 2 important pieces of information, the data that was read in the form of `ReadOnlySequence<byte>` and a bool `IsCompleted` that lets the reader know if the writer is done writing (EOF). After finding the line end delimeter and parsing the line, we slice the buffer to skip what we've already processed and then we call `PipeReader.AdvanceTo` to tell the `PipeReader` how much data we have both consumed and observed. 

At the end of each of the loops, we complete both the reader and the writer. This lets the underlying `Pipe` release all of the memory it allocated.

## System.IO.Pipelines

### Partial Reads

Besides handling the memory management, the other core pipelines feature is the ability to peek at data in the `Pipe` without actually reading it. `PipeReader` has 2 core APIs `ReadAsync` and `AdvanceTo`. `ReadAsync` gets the data in the `Pipe`, `AdvanceTo` does a couple of things, it tells the `PipeReader` that these buffers are no longer required by the reader so they can be discarded (for example returned to the underlying buffer pool). It also allows the reader to tell the `PipeReader` "don't wake me up again until there's more data available". This is important for the performance of the reader as it means the reader won't be signalled until there's more data than was previously marked "observed".

### Back pressure and flow control

As mentioned previously, one of the challenges of de-coupling the parsing thread from the reading thread is the fact that we may end up buffering too much data if the parsing thread can't keep up with the reading thread. To solve this problem, the pipe has 2 settings to control the flow of data, the `PauseWriterThreshold` and the `ResumeWriterThreshold`. The `PauseWriterThreshold` determines how much data should be buffered before calls to `PipeWriter.FlushAsync` returns an incomplete `ValueTask`. The `ResumeWriterThreshold` controls how much the reader has to consume before writing can resume (the ValueTask returned from `PipeWriter.FlushAsync` is marked as complete).

![image](https://user-images.githubusercontent.com/95136/42291183-0114a0f2-7f7f-11e8-983f-5332b7585a09.png)

`PipeWriter.FlushAsync` "blocks" when amount of data in `Pipe` crosses `PauseWriterThreshold`  and "unblocks" when it becomes lower then `ResumeWriterThreshold`. Two values are used to prevent thrashing around the limit.

### Scheduling IO

Usually when using async/await, continuations are called on either on thread pool threads or on the current `SynchronizationContext`. When doing IO it's very important to have fine grained control over where that IO is performed and pipelines exposes a `PipeScheduler` that determines where asynchronous callbacks run. This gives the caller fine grained control over exactly what threads are used for IO. An example of this in practice is in the Kestrel Libuv transport where IO callbacks run on dedicated event loop threads.

### ReadOnlySequence\<T\>

The `Pipe` implementation stores a linked list of buffers that get passed between the `PipeWriter` and `PipeReader`. `PipeReader.ReadAsync` exposes a `ReadOnlySequence<T>` which is a new BCL type that represents a view over one or more  segments of `ReadOnlyMemory<T>`, similar to `Span<T>` and `Memory<T>` which provide a view over arrays and strings.

![image](https://user-images.githubusercontent.com/95136/42292592-74a4028e-7f88-11e8-85f7-a6b2f925769d.png)

The `Pipe` internally maintains pointers to where the reader and writer are in the overall set of allocated data and updates them as data is written or read. The `SequencePosition` represents a single point in the linked list of buffers and can be used to efficiently Slice the `ReadOnlySequence<T>`. Since the `ReadOnlySequence<T>` can support one or more segments, it's typical for high performance processing logic to split fast and slow paths based on single or multiple segments. For example, here's a routine that converts an ASCII `ReadOnlySequence<byte>` into a `string`:

```C#
string GetAsciiString(ReadOnlySequence<byte> buffer)
{
    if (buffer.IsSingleSegment)
    {
        return Encoding.ASCII.GetString(buffer.First.Span);
    }

    return string.Create((int)buffer.Length, buffer, (span, sequence) =>
    {
        var output = span;

        foreach (var segment in sequence)
        {
            Encoding.ASCII.GetChars(segment.Span, output);

            output = output.Slice(segment.Length);
        }
    });
}
```

### Other benefits of the `PipeReader` pattern:
- Some underlying systems support a "bufferless wait", that is, a buffer never needs to be allocated until there's actually data available in the underlying system. For example on linux with epoll, it's possible to wait until data is ready before actually supplying a buffer to do the read. 
- The default `Pipe` makes it easy to write unit tests against networking code. It also makes it easy to test those hard to test patterns where partial data is sent. ASP.NET Core uses this to test various aspects of the Kestrel's http parser.
- Systems that allow exposing the underlying OS buffers (like the Registered IO APIs on Windows) to user code are a natural fit for pipelines since buffers are always provided by the `PipeReader` implementation.

### Other Related types

As part of making System.IO.Pipelines, we also added a number of new primitive BCL types:
- `MemoryPool<T>`, `IMemoryOwner<T>`, `MemoryManager<T>` - .NET Core 1.0 added `ArrayPool<T>` and in .NET Core 2.1 we now have a more general abstration for a pool that works for more than just `T[]`.
- `IBufferWriter<T>` - Represents a sink for writing synchronous buffered data. (`PipeWriter` implements this)
- `IValueTaskSource<T>`, `ValueTask` (non-generic) - `ValueTask<T>` has existed since .NET Core 1.1 but has gained some super powers in .NET Core 2.1 to allow allocation-free awaitable async operations. See https://github.com/dotnet/corefx/issues/27445 for more details.

## How do I use them?

The APIs exist in the [System.IO.Pipelines](https://www.nuget.org/packages/System.IO.Pipelines/) nuget package. Here's an example of a .NET Core 2.1 server application that uses pipelines to handle line based messages (our example above) https://github.com/davidfowl/TcpEcho. It should run with `dotnet run` (or by running it in Visual Studio). It listens to a socket on port 8087 and writes out received messages to the console. You can use a client like netcat or putty to make a connection to 8087 and send line based messages to see it working.

