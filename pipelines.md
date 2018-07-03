## System.IO.Pipelines

### What is it?

System.IO.Pipelines is a library that is designed for doing high performance IO in .NET. It's new in .NET Core 2.1 and is a netstandard library that works on all .NET platforms. 

Let's start with a simple problem. We want to write a TCP server that receives line based messages from a client. The typical
code you would write in .NET today looks something like this:

```C#
var socket = new Socket(...);
var stream = new NetworkStream(socket);
byte[] buffer = new byte[4096];
await stream.ReadAsync(buffer, 0, buffer.Length);
ProcessLine(buffer);
```

This code might work when testing locally but it's broken in general because the entire message (end of line) may not been received in a single call to `ReadAsync`. Even worse, we're failing to look at the result of `stream.ReadAsync()` which returns how much data was actually filled into the buffer. This is a common mistake when using `Stream` today. To make this work, we need to buffer the incoming data until we have found a new line.

```C#
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
```

Once again, this might work locally but it's possible that the line is bigger than 4 KiB (4096 bytes). So we need to resize the input buffer until we have found a new line.

```C#
var socket = new Socket(...);
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
```

This code works but now we're re-sizing the buffer which causes extra allocations and copies. To avoid this, we can store a list of buffers instead of resizing each time we cross the 4KiB buffer size.

```C#
var socket = new Socket(...);
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
```

This code just got much more complicated. We're keeping track the filled up buffers as we're looking for the delimeter. To do this, we're using a `List<ArraySegment<byte>>` here to represent the buffered data while looking for the new line delimeter. 

Now we have a bunch of heap allocated buffers in a list. We can optimize this by using the new `ArrayPool<T>` introduced in .NET Core 2.0:

```C#
var socket = new Socket(...);
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
```

With `System.IO.Pipelines` there is a writer and a reader. The `Socket` is the writer and the code processing the lines is the reader.

```C#
var pipe = new Pipe();
var socket = new Socket(...);
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

# Notes and other random things 


When doing IO in .NET the primary exchange type used today is a `System.IO.Stream`. The typical pattern for reading forces the caller to allocate a `byte[]` to pass into Read\ReadAsync. 

- Various `Stream` implementations have an internal buffer for performance reasons. 

- This can cause more copying that is necessary as the `Stream` has to copy from its internal buffers to the user specified buffer. 

#### Marc Gravel's post on pipelines when it was called channels

The short description of Channels would be something like: "high performance zero-copy buffer-pool-managed asynchronous message pipes". Which is quite a mouthful, so we need to break that down a bit. I'm going to talk primarily in the context of network IO software (aka network client libraries and server applications), but the same concepts apply equally to anything where data goes in or out of a stream of bytes. It is relatively easy to write a basic client library or server application for a simple protocol; heck, spin up a Socket via a listener, maybe wrap it in a NetworkStream, call Read until we get a message we can process, then spit back a response. But trying to do that efficiently is remarkably hard – especially when you are talking about high volumes of connections. I don’t want to get clogged down in details, but it quickly gets massively complicated, with concerns like:

- threading – don’t want a thread per socket
- connection management (listeners starting the read/write cycle when connections are accepted)
- buffers to use to wait for incoming data
- buffers to use to store any backlog of data you can’t process yet (perhaps because you don’t have an entire message)
- dealing with partial messages (i.e. you read all of one message and half of another – do you shuffle the remainder down in the existing buffer? or…?)
- dealing with “frame” reading/writing (another way of saying: how do we decode a stream in to multiple successive messages for sequential processing)
- exposing the messages we parse as structured data to calling code – do we copy the data out into new objects like string? how “allocatey” can we afford to be? (don’t believe people who say that objects are cheap; they become expensive if you allocate enough of them)
- do we re-use our buffers? and if so, how? lots of small buffers? fewer big buffers and hand out slices? how do we handle recycling?

Every layer is forced to allocate `byte[]` to be able to read even before you know data is ready. Buffering on 
multiple levels

[Image here?]

- No common pattern for pooling. Today the consumer of a `Stream` is exposed to using the `ArrayPool<byte>` directly. Pipelines expose APIs that allow pooling to be used safely without consumer intervention.


#### Notes from slides
- ReadAsync returns the internal buffer
- Pipe is responsible for buffer management
- Ability to read without consuming
- No allocations per read/write
- Supports multi-segmented reads/writes


![image](https://user-images.githubusercontent.com/95136/41954870-7ae78b26-7992-11e8-8a84-9cd92367a012.png)

![image](https://user-images.githubusercontent.com/95136/41954897-94a630ee-7992-11e8-9b1b-c541ba05494b.png)

![image](https://user-images.githubusercontent.com/95136/41954905-9f7ed692-7992-11e8-9315-a4067224f8ef.png)
