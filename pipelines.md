## System.IO.Pipelines

### What is it?

Let's start with a simple problem. We want to write a TCP server that receives line based messages from a client. The typical
code you would write in .NET today looks something like this:

```C#
var socket = new Socket(...);
var stream = new NetworkStream(socket);
byte[] buffer = new byte[4096];
var read = await stream.ReadAsync(buffer, 0, buffer.Length);
ProcessLine(buffer, 0, read);
```

This code might work when testing locally but it's broken in general because the entire message (end of line) may not been received in a single call to `ReadAsync`. 

We need to buffer the incoming data until we have found a new line.

```C#
var socket = new Socket(...);
var stream = new NetworkStream(socket);
byte[] buffer = new byte[4096];
var read = 0;
var lineLength = -1;
while (lineLength == -1 && read < buffer.Length)
{
    var current = await stream.ReadAsync(buffer, read, buffer.Length - read);
    read += current;
    lineLength = Array.IndexOf(buffer, (byte)'\n', 0, read);
}
ProcessLine(buffer, 0, lineLength);
```

Once again, this might work locally but it's possible that the line is bigger than 4 KiB (4096 bytes). So we need to resize the input buffer until we have found a new line.

```C#
var socket = new Socket(...);
var stream = new NetworkStream(socket);
byte[] buffer = new byte[4096];
var read = 0;
while (true)
{
    var cur = await stream.ReadAsync(buffer, read, buffer.Length - read);
    read += cur;

    if (cur == 0) break;
}
ProcessLine(buffer);
```

The resizing causes extra allocations and copies. To avoid this, we can store a list of buffers instead.

```C#
var socket = new Socket(...);
var stream = new NetworkStream(socket);
var buffers = new List<byte[]>();
byte[] buffer = new byte[4096];
var read = 0;
while (true)
{

    while (read < buffer.Length)
    {
        read = await stream.ReadAsync(buffer, raead, buffer.Length);
    }
    buffers.Add(buffer);
    buffer = new byte[4096];
}
ProcessLine(buffers);
```

Now we have a bunch of heap allocated buffers in a list. We can optimize this by using the new `ArrayPool<T>` introduced in .NET Core 2.0:

```C#
var socket = new Socket(...);
var stream = new NetworkStream(socket);
var buffers = new List<byte[]>();
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
var read = 0;

while (true)
{
    while (read < buffer.Length)
    {
        read = await stream.ReadAsync(buffer, read, buffer.Length);
    }
    buffers.Add(buffer);
    buffer = ArrayPool<byte>.Shared.Rent(4096);
}

ProcessLine(buffers);

foreach (var buffer buffers) 
{
    ArrayPool<byte>.Shared.Return(buffer);
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

        reader.Advance(buffer.Start, buffer.End);
    }

    reader.Complete();
}


await reading;
await writing;
```

When doing IO in .NET the primary exchange type used today is a `System.IO.Stream`. The typical pattern for reading forces the caller to allocate a `byte[]` to pass into Read\ReadAsync. 

- Various `Stream` implementations have an internal buffer for performance reasons. 

- This can cause more copying that is necessary as the `Stream` has to copy from its internal buffers to the user specified buffer. 


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

### History

This came from implementing the Kestrel HTTP web server which powers ASP.NET Core.

[Image here?]

- No common pattern for pooling. Today the consumer of a `Stream` is exposed to using the `ArrayPool<byte>` directly. Pipelines expose APIs that allow pooling to be used safely without consumer intervention.

Sample here

```C#
var reader = new FilePipeReader();
```

### Benefits

- ReadAsync never copies
- Pipe is responsible for buffer management
- Ability to read without consuming
- No allocations per read/write
- Supports multi-segmented reads/writes

