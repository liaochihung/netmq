介紹
=====

所以你正在找訊息函式庫，也許你對WCF跟MSMQ感到沮喪(我們也是…)，且聽說ZeroMQ很快，所以你找到這裡，NetMQ，一個Zero(或稱ØMQ)的.Net移植。

是的，NetMQ是一個訊息函式庫，而且很快，但需要一點學習時間，期望你能夠快速地掌握它。

## 從那裡開始

`ZeroMQ`和`NetMQ`不是那些你下載後，看一下範例就會的函式庫，它背後有一些原則，要先瞭解後才能順利應用，所以最佳的開始的地方是[ZeroMQ guide](http://zguide.zeromq.org/page:all)，讀一或兩次後，再回來這篇文章。

## ZeroMQ中的Zero

`ZeroMQ`的哲理是從**Zero**開始。**Zero**是指`Zero broker`(`ZeroMQ`沒有中介者)、零延遲、零成本(免費的)及零管理。

更進一步說，**"zero"**代表滲透整個專案的極簡主義的文化。我們通過消除複雜性而不是增加新函式來提供功能。


## 取得函式庫

你可以從[NuGet](https://nuget.org/packages/NetMQ/)取得函式庫。

## 傳送及接收

由於`NetMQ`就是關於`sockets`的，所以傳送及接收是很自然的預期。更由於這屬於NetMQ的一般區域，所以另有一個關於[接收與傳送](https://netmq.readthedocs.io/en/latest/receiving-sending/)的介紹頁面。


## 第一個範例

讓我們開始第一個範例吧，(當然)是**"Hello world"**了：

### Server

    :::csharp
    using (var server = new ResponseSocket())
    {
        server.Bind("tcp://*:5555");

        while (true)
        {
            var message = server.ReceiveFrameString();

            Console.WriteLine("Received {0}", message);

            // processing the request
            Thread.Sleep(100);

            Console.WriteLine("Sending World");
            server.SendFrame("World");
        }
    }

伺服端建立了一個`response`的`socket`型別(在[request-response](request-response.md))章節有更多介紹)，將它綁定到port 5555然後等待訊息。

你可以看到我們不用任何設定，只需要傳送字串。`NetMQ`不只可以傳送字串，雖然它沒有實作序列化的功能(你需要自己實作)，不過你可以在後續學到一些很酷的技巧([Multipart messages](#multipart-messages)。

### Client

    :::csharp
    using (var client = new RequestSocket())
    {
        client.Connect("tcp://localhost:5555");

        for (int i = 0; i < 10; i++)
        {
            Console.WriteLine("Sending Hello");
            client.SendFrame("Hello");

            var message = client.ReceiveFrameString();
            Console.WriteLine("Received {0}", message);
        }
    }

`Client端`建立了一個`request`的`socket`型別，連線並開始傳送訊息。

傳送及接收函式(預設)是阻塞式的。對接收來說很簡單：如果沒有收到訊息它會阻塞；而傳送較複雜一點，且跟它的socket型別有關。對request sockets來說，如果到達high watermark，且沒有另一端的連線，函式會阻塞。

然而你可以呼叫`TrySend`和`TryReceive`以避免等待，如果需要等待，它會回傳false。

    :::csharp
    string message;
    if (client.TryReceiveFrameString(out message))
        Console.WriteLine("Received {0}", message);
    else
        Console.WriteLine("No message received");


## Bind vs Connect

上述範例中你可能會注意到server端使用Bind而client端使用Connect，為什麼？有什麼不同嗎？

`ZeroMQ`為每個潛在的連線建立佇列。如果你的socket連線到三個socket端點，背後實際有三個佇列存在。

使用`Bind`，可以讓其它端點和你建立連接，因為你不知道未來會有多少端點且無法先建立佇列，相反，佇列會在每個端點bound後建立。

使用`Connect`，`ZeroMQ`知道至少會有一個端點，因此它會馬上建立佇列，除了ROUTE型別外的所有型別都是如此，而ROUTE型別只會在我們連線的每個端點有了回應後才建立佇列。

因此，當傳送訊息至沒有綁定的端點的socket或至沒有連線的ROUTE時，將沒有可以儲存訊息的佇列存在。

### When should I use bind and when connect?

作為一般規則，在架構中最穩定的端點上使用**bind**，在動態的、易變的端點上使用**connect**。對request/reply型來說，伺服端使用**bind**，而client端使用**connect**，如同傳統的TCP一樣。

If you can't figure out which parts are more stable (i.e. peer-to-peer), consider a stable device in the middle, which all sides can connect to.
如果你無法確認那一部份會比較穩定(例如點對點連線)，可以考慮在中間放一個穩定的可讓所有端點連線的裝置。

你可 進一步閱讀[ZeroMQ FAQ](http://zeromq.org/area:faq)中的_"Why do I see different behavior when I bind a socket versus connect a socket?"_部份。

## Multipart messages 多段訊息

ZeroMQ/NetMQ在frame的概念上工作，大多數的訊息都可以想成含有一或多個frame。NetMQ提供一些方便的函式讓你傳送字串訊息，然而你也應該瞭解多段frame的概念及如何應用。

在[Message](message.md)章節有更多說明。

##Pattern
`ZeroMQ`(和`NetMQ`)都是關於模式和building blocks的。[ZeroMQ指南](http://zguide.zeromq.org/page:all)講述了所有你需要知道的知識，以幫助你應用這些模式。在你開始用NetMQ前請先確定你已讀過下列章節。

+ <a href="http://zguide.zeromq.org/page:all#Chapter-Sockets-and-Patterns" target="_blank">Chapter 2 - Sockets and Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Advanced-Request-Reply-Patterns" target="_blank">Chapter 3 - Advanced Request-Reply Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Reliable-Request-Reply-Patterns" target="_blank">Chapter 4 - Reliable Request-Reply Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Advanced-Pub-Sub-Patterns" target="_blank">Chapter 5 - Advanced Pub-Sub Patterns</a>

NetMQ也提供了以NetMQ API撰寫的針對少數幾個模式的範例。你應該也能夠在看過[ZeroMQ指南](http://zguide.zeromq.org/page:all)後很簡單的改用NetMQ實作。

這裡有一些已用NetMQ實作的範例程式：

+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Brokerless%20Reliability%20(Freelance%20Pattern)/Model%20One" target="_blank">Brokerless Reliability Pattern - Freelance Model one</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Load%20Balancing%20Pattern" target="_blank">Load Balancer Patterns</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Pirate%20Pattern/Lazy%20Pirate" target="_blank">Lazy Pirate Pattern</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Pirate%20Pattern/Simple%20Pirate" target="_blank">Simple Pirate Pattern</a>
(譯者：原文連結錯誤，可至[NetMQ Samples](https://github.com/NetMQ/Samples/tree/master/src)查看)

其餘的範例，[ZeroMQ指南](http://zguide.zeromq.org/page:all)應是你的第一選擇。

ZeroMQ的模式是以特定型別實作的sockets配對。換句話說，要瞭解ZeroMQ的模式，你要先知道有那些socket型別及它們如何配合。Mostly, this just takes study; there is little that is obvious at this level.

ZeroMQ內建的核心模式是：

* [請求-回應](https://netmq.readthedocs.io/en/latest/request-response/)，將一組客戶端連線至一組服務端。這是一種遠端程序呼叫和task分佈模式。
* [發佈-訂閱](https://netmq.readthedocs.io/en/latest/pub-sub/)，連結一組發佈者至一組訂閱者。這是一種資料分佈式模式。
* 管線，連結在一個有多步驟及迴圈的fan-out/fan-in模式中的節點，這是一種 parallel task distribution and collection。
* Exclusive pair，獨占式地連接兩個socket。這是一種在process中連接兩個執行緒的模式，不要和一般的socket配對混肴。

下列是有效的connect-bind的socket合併配對(雙邊都可以bind)：

+ `PublisherSocket` and `SubscriberSocket`
+ `RequestSocket` and `ResponseSocket`
+ `RequestSocket`  and `RouterSocket`
+ `DealerSocket` and `ResponseSocket`
+ `DealerSocket` and `RouterSocket`
+ `DealerSocket` and `DealerSocket`
+ `RouterSocket` and `RouterSocket`
+ `PushSocket` and `PullSocket`
+ `PairSocket` and `PairSocket`

任何其它的配對方式會產生undocumented及不可靠的結果，ZeroMQ未來的版本可能會在你嘗試時告知錯誤。當然，你也可以使用程式橋接不同的配對，如從某種socket讀取並寫至另一種。

##Options

NetMQ提供了數個會影響動作的選項。

根據你使用的socket型別或你嘗試建立的拓撲，你可能發現需要設定一些ZeroMQ的選項。在NetMQ中，可透過NetMQSocket.Options屬性完成。

下面是你可以在NetMQSocket.Options上設定的可用屬性的列表。很難說要設定那些值，那取決於你想要實現什麼。這邊能做的是列出選項，以讓你知道。如下所示：

+ `Affinity`
+ `BackLog`
+ `CopyMessages`
+ `DelayAttachOnConnect`
+ `Endian`
+ `GetLastEndpoint`
+ `IPv4Only`
+ `Identity`
+ `Linger`
+ `MaxMsgSize`
+ `MulticastHops`
+ `MulticastRate`
+ `MulticastRecoveryInterval`
+ `ReceiveHighWaterMark`
+ `ReceiveMore`
+ `ReceiveBuffer`
+ `ReconnectInterval`
+ `ReconnectIntervalMax`
+ `SendHighWaterMark`
+ `SendTimeout`
+ `SendBuffer`
+ `TcpAcceptFilter`
+ `TcpKeepAlive`
+ `TcpKeepaliveIdle`
+ `TcpKeepaliveInterval`
+ `XPubVerbose`

這裡不會講到所有選項，在用到時才會提。現在只要注意，如果你已經在[ZeroMQ指南](http://zguide.zeromq.org/page:all)中讀過某些選項，那麼這應是你需要設置/讀取的地方。
