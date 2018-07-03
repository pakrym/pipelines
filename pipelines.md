## System.IO.Pipelines

### What is it?

System.IO.Pipelines is a library that is designed for doing high performance IO in .NET. It's new in .NET Core 2.1 and is a netstandard library that works on all .NET platforms. 

Let's start with a simple problem. We want to write a TCP server that receives line based messages from a client. The typical
code you would write in .NET today looks something like this:

```C#
async Task AcceptAsync(Socket socket)
{
    var socket = new Socket(...);
    var stream = new NetworkStream(socket);
    byte[] buffer = new byte[4096];
    await stream.ReadAsync(buffer, 0, buffer.Length);
    ProcessLine(buffer);
}
```

This code might work when testing locally but it's broken in general because the entire message (end of line) may not been received in a single call to `ReadAsync`. Even worse, we're failing to look at the result of `stream.ReadAsync()` which returns how much data was actually filled into the buffer. This is a common mistake when using `Stream` today. To make this work, we need to buffer the incoming data until we have found a new line.

```C#
async Task AcceptAsync(Socket socket)
{
    var socket = new Socket(...);
    var stream = new NetworkStream(socket);
    byte[] buffer = new byte[4096];
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

Once again, this might work locally but it's possible that the line is bigger than 4 KiB (4096 bytes). So we need to resize the input buffer until we have found a new line.

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    byte[] buffer = new byte[4096];
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining == 0)
        {
            // Resize the buffer and copy all of the data
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

        if (lineLength > 0) 
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
    byte[] buffer = new byte[4096];
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining == 0)
        {
            // This buffer is full so add it to the list
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

        if (lineLength > 0) 
        {
            // Add the buffer to the list of buffers
            buffers.Add(new ArraySegment<byte>(buffer, 0, lineLength));

            ProcessLine(buffers);

            buffers.Clear();
        }
    }
}
```

This code just got much more complicated. We're keeping track the filled up buffers as we're looking for the delimeter. To do this, we're using a `List<ArraySegment<byte>>` here to represent the buffered data while looking for the new line delimeter. As a result, ProcessLine now accepts a `List<ArraySegment<byte>>` instead of a `byte[]`, `offset` and `count`. Our parsing logic needs to now handle either a single/multiple buffer segments.

There are a few more optimizations that we need to make before we call this server complete. Right now we have a bunch of heap allocated buffers in a list. We can optimize this by using the new `ArrayPool<T>` introduced in .NET Core 2.0 to avoid repeated buffer allocations as we're parse more lines from the client:

```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffers = new List<ArraySegment<byte>>();
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining == 0)
        {
            // This buffer is full so add it to the list
            buffers.Add(new ArraySegment<byte>(buffer));
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

        if (lineLength > 0) 
        {
            // Add the buffer to the list of buffers
            buffers.Add(new ArraySegment<byte>(buffer, 0, lineLength));

            ProcessLine(buffers);

            // Return to the array pool so we don't leak memory
            foreach (var buffer buffers) 
            {
                ArrayPool<byte>.Shared.Return(buffer.Array);
            }

            buffers.Clear();
        }
    }
}
```

So far our server now handles, partial messages and it uses pooled memory to reduce overall memory consumption. We can improve on this further by decoupling the reading and parsing logic. This lets us consume buffers from the `Socket` as they become available without letting the parsing of those buffers stop us from reading more data. 


```C#
async Task AcceptAsync(Socket socket)
{
    var stream = new NetworkStream(socket);
    var buffers = new List<ArraySegment<byte>>();
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
    var read = 0;
    while (true)
    {
        var remaining = buffer.Length - read;

        if (remaining == 0)
        {
            // This buffer is full so add it to the list
            buffers.Add(new ArraySegment<byte>(buffer));
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

        if (lineLength > 0) 
        {
            // Add the buffer to the list of buffers
            buffers.Add(new ArraySegment<byte>(buffer, 0, lineLength));

            ProcessLine(buffers);

            // Return to the array pool so we don't leak memory
            foreach (var buffer buffers) 
            {
                ArrayPool<byte>.Shared.Return(buffer.Array);
            }

            buffers.Clear();
        }
    }
}

async Task Parsing()
{
}
```

Let's take a look at what this example looks like with pipelines.

```C#
async Task AcceptAsync(Socket socket)
{
    var pipe = new Pipe();
    Task writing = ReadFromSocket(socket, pipe.Writer);
    Task reading = ReadFromPipe(pipe.Reader);

    async Task ReadFromSocket(Socket socket, PipeWriter writer)
    {
        while (true)
        {
            Memory<byte> memory = writer.GetMemory();
            int read = await socket.ReceiveAsync(memory, SocketFlags.None);
            if (read == 0)
            {
                break;
            }
            writer.Advance(read);
            await writer.FlushAync();
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

The pipelines version of our line reader has 2 loops:
- One loop reads from the Socket and writes into the PipeWriter
- The other loop reads from the PipeReader and parses incoming lines

Unlike, the `Stream` version of the example, there are no explicit buffers allocated anywhere. This is one of pipelines' core features. All buffer management is delegated to the `PipeReader`/`PipeWriter` implementations. This makes it easier for consuming code to focus solely on the business logic instead of complex buffer management. In the first loop, we first call `PipeWriter.GetMemory()` to get some memory from the underlying writer then we call `PipeWriter.Advance(int)` to tell the `PipeWriter` how much data we actually wrote to the buffer. We then call `PipeWriter.FlushAsync()` to make the data available to the `PipeReader`.

In the second loop, we're consuming the buffers written by the `PipeWriter` which ultimately comes from the `Socket`. When the call to `PipeReader.ReadAsync()` returns, we get a `ReadResult` which contains 2 important pieces of information, the data that was read in the form of `ReadOnlySequence<byte>` and a bool `IsCompleted` that lets the reader know if more data would be written into the pipe. After finding the line end delimeter and parsing the line, we slice the buffer to skip what we've already processed and then we call `PipeReader.AdvanceTo` to tell the `PipeReader` how much data we have both consumed and observed.

There are a few more concepts to grok here but under the covers there are lots of things being managed for you:
- The `Pipe` is efficiently managing renting/returning pooled buffers.
  - When you call `PipeWriter.GetMemory()`, the `Pipe` will efficiently manage a linked list of segments.


### ReadOnlySequence<T>

Both `Span<T>` and `Memory<T>` provide functionality for contiguous buffers such as arrays and strings. System.Memory contains a new sliceable type called `ReadOnlySequence<T>` within the System.Buffers namespace that offers support for discontiguous buffers represented by a linked list of `ReadOnlyMemory<T>` nodes. 

You can read more about it on the https://www.codemag.com/Article/1807051/Introducing-.NET-Core-2.1-Flagship-Types-Span-T-and-Memory-T
