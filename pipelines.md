# System.IO.Pipelines

## What is it?

System.IO.Pipelines is a new library that is designed for doing high performance IO in .NET. It's new in .NET Core 2.1 and is a netstandard library that works on all .NET platforms. 

Pipelines was born from the work the .NET Core team did to make Kestrel one of the fastest web servers in the industry. What started as an implementation detail inside of Kestrel progressed into a re-usable API that shipped in 2.1 as a first class BCL API (System.IO.Pipelines) available for all .NET developers. Today Pipelines powers Kestrel and SignalR and we hope to see it at the center of many networking libraries and components from the .NET community. 

## What problem does it solve? 

Let's start with a simple problem. We want to write a TCP server that receives line based messages (delimited by \n) from a client. The typical
code you would write in .NET before pipelines looks something like this:

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffer = new byte[4096];
    await stream.ReadAsync(buffer, 0, buffer.Length);
    ProcessLine(buffer);
}
```

This code might work when testing locally but it's broken in general because the entire message (end of line) may not been received in a single call to `ReadAsync`. Even worse, we're failing to look at the result of `stream.ReadAsync()` which returns how much data was actually filled into the buffer. This is a common mistake when using `Stream` today. To make this work, we need to buffer the incoming data until we have found a new line.

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffer = new byte[4096];
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

        if (lineLength > 0)
        {
            ProcessLine(buffer, 0, lineLength);
            read = 0;
        }
    }
}
```

Once again, this might work in local testing but it's possible that the line is bigger than 4KiB (4096 bytes). So we need to resize the input buffer until we have found a new line.

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffer = new byte[4096];
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

This code works but now we're re-sizing the buffer which causes extra allocations and copies. To avoid this, we can store a list of buffers instead of resizing each time we cross the 4KiB buffer size.

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffers = new List<ArraySegment<byte>>();
    var buffer = new byte[4096];
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining == 0)
        {
            buffers.Add(new ArraySegment<byte>(buffer));
            buffer = new byte[4096];
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

This code just got much more complicated. We're keeping track the filled up buffers as we're looking for the delimeter. To do this, we're using a `List<ArraySegment<byte>>` here to represent the buffered data while looking for the new line delimeter. As a result, ProcessLine now accepts a `List<ArraySegment<byte>>` instead of a `byte[]`, `offset` and `count`. Our parsing logic needs to now handle either a single/multiple buffer segments.

There are a few more optimizations that we need to make before we call this server complete. Right now we have a bunch of heap allocated buffers in a list. We can optimize this by using the new `ArrayPool<T>` introduced in .NET Core 2.0 to avoid repeated buffer allocations as we're parse more lines from the client. We also probably don't want to pass small buffers to `ReadAsync` as that would result in more calls into the operating system.

```C#
async Task AcceptAsync(Socket socket)
{
    // We want at least this much room left in the buffer for a call to ReadAsync
    const int minimumBufferSize = 1024;
    
    var stream = new NetworkStream(socket);
    var buffers = new List<ArraySegment<byte>>();
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining < minimumBufferSize)
        {
            buffers.Add(new ArraySegment<byte>(buffer, 0, read));
            buffer = ArrayPool<byte>.Shared.Rent(4096);
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

Our server now handles partial messages, and it uses pooled memory to reduce overall memory consumption. Now we need to tackle the throughput. A common pattern used to increase the throughput is to decouple the reading and processing logic. This lets us consume buffers from the `Socket` as they become available without letting the parsing of those buffers stop us from reading more data. This introduces a couple problems though:
- We need 2 loops, one that reads from the socket and one that processes and parses the buffers.
- We need a way to signal the parsing logic when data becomes available.
- We need to decide what happens if the loop reading from the socket is "too fast". We need a way to throttle the reading loop if the parsing logic can't keep up.
- We need to make sure things are thread safe. We're not sharing a set of buffers between the reading loop and the parsing loop and those run independently on different threads.
- Memory management logic is now spread across 2 different pieces of code, the code that rents from the buffer pool is reading from the socket and the code that returns from the buffer pool is the parsing logic.
- We need to be extremely careful with how we return buffers after the parsing logic is done with them. If we're not careful, it's possible that we return a buffer that's still being written into by the reading logic.


```C#

class BufferSegment
{
   public byte[] Buffer { get; set; }
   public int Length { get; set; }
   public bool Full { get; set; }
   public int Remaining => Buffer.Length - Length;
}

async Task AcceptAsync(Socket socket)
{
    var semaphore = new SemaphoreSlim(1, 1);
    var queue = new ConcurrentQueue<BufferSegment>();
    var reading = ReadFromSocket(socket, queue, semaphore);
    var writing = ReadFromQueue(queue, semaphore);
    
    async Task ReadFromSocket(Socket s, ConcurrentQueue<BufferSegment> buffers, SemaphoreSlim semaphore)
    {
        const int minimumBufferSize = 1024;
        
        var stream = new NetworkStream(s);
        var segment = new BufferSegment 
        {
            Buffer = ArrayPool<byte>.Shared.Rent(4096);
        };
        
        buffers.Enqueue(segment);
        
        while (true)
        {
            if (segment.Remaining < minimumBufferSize)
            {
                segment.Full = true;
                segment = new BufferSegment 
                {
                    Buffer = ArrayPool<byte>.Shared.Rent(4096);
                };
                
                buffers.Enqueue(segment);
            }

            var current = await stream.ReadAsync(segment.Buffer, segment.Length, segment.Remaining);
            semaphore.Release();
            
            if (current == 0)
            {
                break;
            }

            segment.Length += current;
        }
    }
    
    async Task ReadFromQueue(List<BufferSegment> buffers, SemaphoreSlim semaphore)
    {      
        // This is still broken...
        while (true)
        {
            await semaphore.WaitAsync();
            
            for (var i = 0; i < buffers.Count; ++i)
            {
                var segment = buffers[i];
                var lineLength = Array.IndexOf(segment.Buffer, (byte)'\n', 0, segment.Length);

                if (lineLength > 0)
                {
                    ProcessLine(buffers, 0, i, lineLength);

                    foreach (var segment in buffers)
                    {
                        if (segment.Full)
                        {
                            ArrayPool<byte>.Shared.Return(segment.Buffer);
                        }
                    }
                }
            }
        }
    }
    
    await reading;
    await writing;
}
```

The complexity has gone through the roof (and there are still bugs!). High performance networking usually means writing very complex code in order to eke out more performance from the system.

## TCP server with System.IO.Pipelines

Let's take a look at what this example looks like with System.IO.Pipelines.

```C#
async Task AcceptAsync(Socket socket)
{
    var pipe = new Pipe();
    Task writing = ReadFromSocket(socket, pipe.Writer);
    Task reading = ReadFromPipe(pipe.Reader);

    async Task ReadFromSocket(Socket socket, PipeWriter writer)
    {
        const int minimumBufferSize = 1024;
        
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

    async Task ReadFromPipe(PipeReader reader)
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

            reader.AdvanceTo(buffer.Start, buffer.End);

            if (result.IsCompleted)
            {
                break;
            }
        }

        reader.Complete();
    }

    await reading;
    await writing;
}
```

The pipelines version of our line reader has the same 2 loops:
- One loop reads from the Socket and writes into the PipeWriter.
- The other loop reads from the PipeReader and parses incoming lines.

Unlike, the `Stream` version of the example, there are no explicit buffers allocated anywhere. This is one of pipelines' core features. All buffer management is delegated to the `PipeReader`/`PipeWriter` implementations. This makes it easier for consuming code to focus solely on the business logic instead of complex buffer management. In the first loop, we first call `PipeWriter.GetMemory(int)` to get some memory from the underlying writer then we call `PipeWriter.Advance(int)` to tell the `PipeWriter` how much data we actually wrote to the buffer. We then call `PipeWriter.FlushAsync()` to make the data available to the `PipeReader`.

In the second loop, we're consuming the buffers written by the `PipeWriter` which ultimately comes from the `Socket`. When the call to `PipeReader.ReadAsync()` returns, we get a `ReadResult` which contains 2 important pieces of information, the data that was read in the form of `ReadOnlySequence<byte>` and a bool `IsCompleted` that lets the reader know if more data would be written into the pipe. After finding the line end delimeter and parsing the line, we slice the buffer to skip what we've already processed and then we call `PipeReader.AdvanceTo` to tell the `PipeReader` how much data we have both consumed and observed. `PipeReader.AdvanceTo` does a couple of things, it signals to the `PipeReader` that these buffers are no longer required by the reader so they can be discarded (for e.g returned to the underlying buffer pool) and it allows the reader to identify the how much of the buffer was examined. This is important for performance when buffers can be partially consumed as  it allows the reader to say "don't wake me up again until there's more data available".

At the end of each of the loops, we complete both the reader and the writer.

Other benefits that arise from the `PipeReader` patterns:
- Because the buffers are controlled by the `PipeReader` it makes writing a `PipeReader` that wraps another `PipeReader` *mostly* buffer free. The buffers from the underlying `PipeReader` flow all the way out to the reader.
- Some underlying systems support a "bufferless wait", that is, a buffer never needs to be allocated until there's actually data available in the underlying system. For example on linux with epoll, it's possible to wait until data is ready before actually supplying a buffer to do the read. 
- Having a default `Pipe` available makes it easy to write unit tests against networking code. It also makes it easy to test those hard to test patterns where partial data is sent. ASP.NET Core uses this to test various aspects of the Kestrel's http parser.
- Systems that allow exposing the underlying OS buffers (like the Registered IO APIs on Windows) to user code are a natural fit for pipelines since buffers are always provided by the `PipeReader` implementation.

## ReadOnlySequence<T>

Both `Span<T>` and `Memory<T>` provide functionality for contiguous buffers such as arrays and strings. System.Memory contains a new sliceable type called `ReadOnlySequence<T>` within the System.Buffers namespace that offers support for discontiguous buffers represented by a linked list of `ReadOnlyMemory<T>` nodes. 

