Transport Protocols
===

NetMQ支援三種主要的協定：

+ TCP (`tcp://`)
+ InProc (`inproc://`)
+ PGM (`pgm://`) &mdash; requires MSMQ and running as administrator

下面會一一介紹。


## TCP

TCP是最常用到的協定，因此，大部份的程式碼會使用TCP展示。

### Example

又一個簡單的範例：

    :::csharp
    using (var server = new ResponseSocket())
    using (var client = new RequestSocket())
    {
        server.Bind("tcp://*:5555");
        client.Connect("tcp://localhost:5555");

        Console.WriteLine("Sending Hello");
        client.SendFrame("Hello");

        var message = server.ReceiveFrameString();
        Console.WriteLine("Received {0}", message);

        Console.WriteLine("Sending World");
        server.SendFrame("World");

        message = client.ReceiveFrameString();
        Console.WriteLine("Received {0}", message);
    }

輸出：

    :::text
    Sending Hello
    Received Hello
    Sending World
    Received World

### Address format

注意位址格式字串會傳送給`Bind()`及`Connect()`函式。
在TCP連線中，它會被組成：

    :::text
    tcp://*:5555

這由三個部份構成：

 1. 協議（tcp）
 2. 主機（IP地址，主機名或匹配`"*"`的wildcard）
 3. 埠號（5555）


## InProc

InProc (in-process)讓你可以在同一個process中用sockets連線溝通，這很有用，有幾個理由：

+ 取消共享狀態/鎖。當你傳送資料至socket時不需要擔心共享狀態。Socket的每一端都有自己的副本。
+ 能夠在系統的不同的部分之間進行通信。

NetMQ提供了幾個使用InProc的組件，例如[Actor模型](actor.md)和Devices，在相關文件中會再討論。

### Example

現在讓我們通過在兩個執行緒之間傳送一個字串（為了簡單起見）展示一個簡單的InProc。

    :::csharp
    using (var end1 = new PairSocket())
    using (var end2 = new PairSocket())
    {
        end1.Bind("inproc://inproc-demo");
        end2.Connect("inproc://inproc-demo");

        var end1Task = Task.Run(() =>
        {
            Console.WriteLine("ThreadId = {0}", Thread.CurrentThread.ManagedThreadId);
            Console.WriteLine("Sending hello down the inproc pipeline");
            end1.SendFrame("Hello");
        });
        var end2Task = Task.Run(() =>
        {
            Console.WriteLine("ThreadId = {0}", Thread.CurrentThread.ManagedThreadId);
            var message = end2.ReceiveFrameString();
            Console.WriteLine(message);
        });
        Task.WaitAll(new[] { end1Task, end2Task });
    }

輸出：

    :::text
    ThreadId = 12
    ThreadId = 6
    Sending hello down the inproc pipeline
    Hello

### Address format

注意位址格式字串會傳送給`Bind()`及`Connect()`函式。
在InProc連線中，它會被組成：

    :::text
    inproc://inproc-demo

這由兩個部份構成：

1. 協定(inproc)
2. 辨識名稱（inproc-demo可以是任何字串，在process範圍內是唯一的名稱）


## PGM

Pragmatic General Multicast (PGM)是一種可靠的多播傳輸協定，用於需要有序、無序、不重覆等可從多個來源至多個接收者的多播數據。

PGM保證群組中的接收者可接收來自不管是傳送或修復，或可偵測無法復原的資料封包的遺失。PGM被設計為一個擁有基本的可靠度需求的解決方案。它的中心設計目標是操作的簡易性且保證其彈性及網路效率。

要使用NetMQ的PGM，我們不用做太多，只須遵循下列三點：

1. Sockets型別現在是`PublisherSocket` and `SubscriberSocket`，在[pub-sub pattern](pub-sub.md)會有更詳細的介紹。
2. 確定你以"Administrator"等級執行軟體。
3. 確定已打開"Multicastng Support"，可依下列方式：

![](Images/PgmSettingsInWindows.png)

### Example

這裡是一個使用PGM的小範例，以及`PublisherSocket` and `SubscriberSocket`和幾個選項值。

```csharp
    const int MegaBit = 1024;
    const int MegaByte = 1024;
    using (var pub = new PublisherSocket())
    using (var sub1 = new SubscriberSocket())
    using (var sub2 = new SubscriberSocket())
    {
        pub.Options.MulticastHops = 2;
        pub.Options.MulticastRate = 40 * MegaBit; // 40 megabit
        pub.Options.MulticastRecoveryInterval = TimeSpan.FromMinutes(10);
        pub.Options.SendBuffer = MegaByte * 10; // 10 megabyte
        pub.Connect("pgm://224.0.0.1:5555");

        sub1.Options.ReceiveBuffer = MegaByte * 10;
        sub1.Bind("pgm://224.0.0.1:5555");
        sub1.Subscribe("");

        sub2.Bind("pgm://224.0.0.1:5555");
        sub2.Options.ReceiveBuffer = MegaByte * 10;
        sub2.Subscribe("");

        Console.WriteLine("Server sending 'Hi'");
        pub.Send("Hi");

        bool more;
        Console.WriteLine("sub1 received = '{0}'", sub1.ReceiveString(out more));
        Console.WriteLine("sub2 received = '{0}'", sub2.ReceiveString(out more));
    }
```

執行後輸出如下：
```dos
    Server sending 'Hi'
    sub1 received = 'Hi'
    sub2 received = 'Hi'
```

### Address format

注意傳入`Bind()` and `Connect()`的字串位址格式，對InProc連線來說，會類似：

* pgm://224.0.0.1:5555

它以三個部份組成：

1. 協定(`pgm`)
2. 主機(如`244.0.0.1`之類的IP位址，主機名稱，或萬用字元`*`的匹配)
3. Port number(`5555`)

另外的好PGM資訊是[PGM unit tests](https://github.com/zeromq/netmq/blob/master/src/NetMQ.Tests/PgmTests.cs).
