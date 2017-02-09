## RouterSocket

從 [ZeroMQ guide](http://zguide.zeromq.org/page:all):

> ROUTER socket，不像其它的sockets，會追蹤它的每個連線，且告知caller。告知的方式是透過在收到的訊息的前面加上一連線示別的資訊。示別碼，有時也被稱為位址，只是一個表示“這是代表此連線的唯一示別碼”，而不包含任何其它資訊。然後，當你透過ROUTER socket傳送訊息時，你會傳送一個示別碼的frame。
> 當接收訊息時，一個ZMQ_ROUTER socket應在傳送至應用程式前，在訊息前置一個包含原始節點的辨視碼，收到的訊息會公平地將所有節點的訊息放至佇列中。當傳送訊息時，一個ZMQ_ROUTER socket應該將訊息的第一個部份移除，並使用目的端的辨視碼取代。
>
> Identities是一個很難的概念，但如果你想成為一個ZeroMQ的專家，它是至關重要的。ROUTER socket會為它的每一個連線隨機產生一個辨視碼。如果有三個REQ socket連線至一個ROUTER socket上，它會產生三個辨視碼，對映至每一個REQ socket上。

所以我們來看一個較小的範例，我們有一個`DealerSocket`，帶有一個3 byte的示別碼"ABC"，在內部，這表示`RouterSocket`型別的socket內保有一個hash table，它可以搜尋"ABC"，並找到這一個`DealerSocket`的TCP連線。

當我們收到來自`DealerSocket`的訊息時，我們取得三個frames：

![](https://github.com/imatix/zguide/raw/master/images/fig28.png)


### Identities and Addresses

From [ZeroMQ guide, Identities and Addresses](http://zguide.zeromq.org/page:all#Identities-and-Addresses):

> ZeroMQ中的辨視碼概念特指的是ROUTER sockets，以及它們如何辨別與其它socket的連線。更廣泛的說，辨視碼被當作為回信的地址。大多狀況下，辨視碼是arbitrary且在本地對映至ROUTER socket上：它是一個雜湊表中的查詢鍵。所以一個節點可以有一個實體的位址(如"tcp://192.168.55.117:5670"的網路端點)或邏輯上的位址(一個UUID或是email或其它的唯一鍵值)。
>
> 一個使用ROUTER socket和特定節點溝通的應用程式，如果有建立雜湊表，就可以將一個邏輯位址轉成辨視碼。因為ROUTER socket只announce一個連線(至特定節點)的identity，當此連線傳送訊息時，你只能夠回覆，而不能自發地與之交談。
>
> 這是事實，即時你將規則翻轉，且讓ROUTER連線至節點，而不是等待節點連線至ROUTER。然而你可以強制一個ROUTER socket使用邏輯位址來替代其identity，zmq_setsockopt說明頁呼叫這個以設定socket的identity，它的工作原理如下：
>
> * 節點應用程式在binding或connecting前設定它的節點socket(DEALER or REQ)的ZMQ_IDENTITY選項。
> * 再來這節點會連線至already-bound的ROUTER socket上，但ROUTER也可以連線至此節點。
> * 在連線時，節點socket會告訴router socket，“請為此連線使用這個辨視碼”。
> * 如果節點socket沒有這樣子說，router會隨機產生一個辨視碼給此連線。
> * ROUTER socket現在會提供一個邏輯位址給此程式，做為所有來自此節點的訊息的前置辨視碼用的frame。


## DealerSocket

NetMQ的`DealerSocket`不做任何特別的事情，它提供的是以完全非同步方式工作的能力。

Which if you recall was not something that other socket types could do, where the `ReceieveXXX` / `SendXXX` methods are blocking, and would also throw exceptions should you try to call
things in the wrong order, or more than expected.

DealerSocket的主要賣點是它的非同步能力。通常，`DealerSocket`會與`RouterSocket`結合使用，這就是為什麼我們決定將這兩種socket型別的介紹放在一起。

如果你想瞭解更多包含`DealerSocket`的socket combinations，指南總是你的朋友，在指南中的這一頁<a href="http://zguide.zeromq.org/page:all#toc58" target="_blank">Request-Reply Combinations</a>你也許也感興趣。

## An example

又到了範例的時間，瞭解此範例的最佳要點總結如下：

* 有一個伺服器，它綁定了一個`RouterSocket`，因此會儲存傳入的請求連線的示別資訊，所以可以正確的將訊息回應至client socket。
* 有很多個client，每個client都屬於個別執行緒，這些client的型別是`DealerSocket`，這一個client socket會提供固定的示別碼，以讓伺服端(`DealerSocket`)可以正確的回應訊息。

程式碼如下：

```csharp
    public static void Main(string[] args)
    {
        // NOTES
        // 1. Use ThreadLocal<DealerSocket> where each thread has
        //    its own client DealerSocket to talk to server
        // 2. Each thread can send using it own socket
        // 3. Each thread socket is added to poller

        const int delay = 3000; // millis

        var clientSocketPerThread = new ThreadLocal<DealerSocket>();

        using (var server = new RouterSocket("@tcp://127.0.0.1:5556"))
        using (var poller = new NetMQPoller())
        {
            // Start some threads, each with its own DealerSocket
            // to talk to the server socket. Creates lots of sockets,
            // but no nasty race conditions no shared state, each
            // thread has its own socket, happy days.
            for (int i = 0; i < 3; i++)
            {
                Task.Factory.StartNew(state =>
                {
                    DealerSocket client = null;

                    if (!clientSocketPerThread.IsValueCreated)
                    {
                        client = new DealerSocket();
                        client.Options.Identity =
                            Encoding.Unicode.GetBytes(state.ToString());
                        client.Connect("tcp://127.0.0.1:5556");
                        client.ReceiveReady += Client_ReceiveReady;
                        clientSocketPerThread.Value = client;
                        poller.Add(client);
                    }
                    else
                    {
                        client = clientSocketPerThread.Value;
                    }

                    while (true)
                    {
                        var messageToServer = new NetMQMessage();
                        messageToServer.AppendEmptyFrame();
                        messageToServer.Append(state.ToString());
                        Console.WriteLine("======================================");
                        Console.WriteLine(" OUTGOING MESSAGE TO SERVER ");
                        Console.WriteLine("======================================");
                        PrintFrames("Client Sending", messageToServer);
                        client.SendMultipartMessage(messageToServer);
                        Thread.Sleep(delay);
                    }

                }, string.Format("client {0}", i), TaskCreationOptions.LongRunning);
            }

            // start the poller
            poller.RunAsync();

            // server loop
            while (true)
            {
                var clientMessage = server.ReceiveMessage();
                Console.WriteLine("======================================");
                Console.WriteLine(" INCOMING CLIENT MESSAGE FROM CLIENT ");
                Console.WriteLine("======================================");
                PrintFrames("Server receiving", clientMessage);
                if (clientMessage.FrameCount == 3)
                {
                    var clientAddress = clientMessage[0];
                    var clientOriginalMessage = clientMessage[2].ConvertToString();
                    string response = string.Format("{0} back from server {1}",
                        clientOriginalMessage, DateTime.Now.ToLongTimeString());
                    var messageToClient = new NetMQMessage();
                    messageToClient.Append(clientAddress);
                    messageToClient.AppendEmptyFrame();
                    messageToClient.Append(response);
                    server.SendMultipartMessage(messageToClient);
                }
            }
        }
    }

    void PrintFrames(string operationType, NetMQMessage message)
    {
        for (int i = 0; i < message.FrameCount; i++)
        {
            Console.WriteLine("{0} Socket : Frame[{1}] = {2}", operationType, i,
                message[i].ConvertToString());
        }
    }

    void Client_ReceiveReady(object sender, NetMQSocketEventArgs e)
    {
        bool hasmore = false;
        e.Socket.Receive(out hasmore);
        if (hasmore)
        {
            string result = e.Socket.ReceiveFrameString(out hasmore);
            Console.WriteLine("REPLY {0}", result);
        }
    }
```

執行後，輸出應如下所示：


```dos
    ======================================
     OUTGOING MESSAGE TO SERVER
    ======================================
    ======================================
     OUTGOING MESSAGE TO SERVER
    ======================================
    Client Sending Socket : Frame[0] =
    Client Sending Socket : Frame[1] = client 1
    Client Sending Socket : Frame[0] =
    Client Sending Socket : Frame[1] = client 0
    ======================================
     INCOMING CLIENT MESSAGE FROM CLIENT
    ======================================
    Server receiving Socket : Frame[0] = c l i e n t   1
    Server receiving Socket : Frame[1] =
    Server receiving Socket : Frame[2] = client 1
    ======================================
     INCOMING CLIENT MESSAGE FROM CLIENT
    ======================================
    Server receiving Socket : Frame[0] = c l i e n t   0
    Server receiving Socket : Frame[1] =
    Server receiving Socket : Frame[2] = client 0
    REPLY client 1 back from server 08:05:56
    REPLY client 0 back from server 08:05:56
```

記住這是非同步的程式碼，所以事件的發生順序可能不如你所預期的。
