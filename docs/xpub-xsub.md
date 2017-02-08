XPub / XSub
=====

[Pub/Sub](pub-sub.md)適用於多個訂閱者和單一發佈者，但是如果您需要多個發佈者，那麼XPub / XSub模式會比較有趣。

XPub / XSub還可以協助所謂的 _dynamic discovery problem_. From the [ZeroMQ guide](http://zguide.zeromq.org/page:all#The-Dynamic-Discovery-Problem):

> 當你在設計較大型的分佈式架構時可能會遇到的一個問題是discovery，也就是每一個節點如何知道其它節點？特別是在節點來來去去的狀況下，我們把可狀況稱做"dynamic discovery problem"。
>
> 有幾種解決方法。最簡單的是整網路架構以hard-coding (or configuring)的方式手動指定以全然避免掉此狀況，也就是說當你增加一個新節點後，重新設置網路。
>
> ![](https://github.com/imatix/zguide/raw/master/images/fig12.png)
>
> 在實際上，這會導致越來越脆弱和笨重的架構。假設你有一個發佈者和一百個訂閱者。你通過在每個訂閱者中設定發佈伺服器端點，將每個訂閱者連接到發佈者。這很容易。訂閱者是動態的；發佈者是靜態的。現在如果說突然間你要增加更多發佈者，這不再容易完成。如果你繼續將每個訂閱者連接到每個發佈者，則避免dynamic discovery的成本會越來越高。
>
> ![](https://github.com/imatix/zguide/raw/master/images/fig13.png)
>
> 這有不少解答，最簡單的是增加一個中介層；也就是說，在網路中增加一個固定的點，以讓其它節點連線。在典型的訊息架構中，這會由message broker負責。ZeroMQ並沒有這樣的一個message broker，但它讓建立中介層的工作變得很簡單。
>
> 你也許會疑惑，如果所有的網路最終會大到需要一個中介層，為什麼我們不為所有的應用都提供一個中介層？對於初學者，這是一個公平的妥協。總是使用星狀拓璞，不要考慮效能，事情總是能夠工作。然而，message brokers是貪婪的東西；在它們做為中央中介者的角色，會變得太複雜，太多狀態，最終會造成問題。
>
> 最好是把中介層當做一個簡單的無狀態的訊息交換機。一個好的類比是HTTP代理；它存在那裡，但不作為任何特定的角色。在我們的範例中，增加一個pub-sub代理可解決dynamic discovery問題，我們將代理設置在網路的"中間"，這個代理會打開一個XSUB的socket，及一個XPUB的socket，並綁定至一個大家都知道的IP及port上，然後，所有其它的節點連線至此代理，而不是互相連線。增加更多的訂閱者或是發佈者不再是問題。
>
> 我們需要XPUB和XSUB socket，因為ZeroMQ會把訂閱者的訂閱轉發至發佈者。XPUB和XSUB與PUB和SUB完全一樣，除了它們將訂閱當成特別的訊息。代理器需轉發這些訂閱者的訂閱至發佈者，靠著從XSUB socket讀取並寫至XPUB socket上，這是XPUB和XSUB主要的使用方式。


## An Example

所以現在我們已經了解了為什麼要使用XPub / XSub，現在讓我們看一個依上述描述的範例。分為三個部分：

+ Publisher
+ Intermediary
+ Subscriber

### Publisher

可以看到`PublisherSocket`連線到`XSubscriberSocket`的位址。

    :::csharp
    using (var pubSocket = new PublisherSocket(">tcp://127.0.0.1:5678"))
    {
        Console.WriteLine("Publisher socket connecting...");
        pubSocket.Options.SendHighWatermark = 1000;

        var rand = new Random(50);
        
        while (true)
        {
            var randomizedTopic = rand.NextDouble();
            if (randomizedTopic > 0.5)
            {
                var msg = "TopicA msg-" + randomizedTopic;
                Console.WriteLine("Sending message : {0}", msg);
                pubSocket.SendMore("TopicA").Send(msg);
            }
            else
            {
                var msg = "TopicB msg-" + randomizedTopic;
                Console.WriteLine("Sending message : {0}", msg);
                pubSocket.SendMore("TopicB").Send(msg);
            }
        }
    }


### Intermediary

`Intermediary`負責在`XPublisherSocket`和`XSubscriberSocket`之間雙向地中繼訊息。`NetMQ`提供了一個使用簡單的代理類別。

    :::csharp
    using (var xpubSocket = new XPublisherSocket("@tcp://127.0.0.1:1234"))
    using (var xsubSocket = new XSubscriberSocket("@tcp://127.0.0.1:5678"))
    {
        Console.WriteLine("Intermediary started, and waiting for messages");

        // proxy messages between frontend / backend
        var proxy = new Proxy(xsubSocket, xpubSocket);

        // blocks indefinitely
        proxy.Start();
    }


### Subscriber

可以看到`SubscriberSocket`連線到`XPublisherSocket`的位址。

    :::csharp
    string topic = /* ... */; // one of "TopicA" or "TopicB"

    using (var subSocket = new SubscriberSocket(">tcp://127.0.0.1:1234"))
    {
        subSocket.Options.ReceiveHighWatermark = 1000;
        subSocket.Subscribe(topic);
        Console.WriteLine("Subscriber socket connecting...");

        while (true)
        {
            string messageTopicReceived = subSocket.ReceiveString();
            string messageReceived = subSocket.ReceiveString();
            Console.WriteLine(messageReceived);
        }
    }


執行時，可以看到如下列輸出：

![](Images/XPubXSubDemo.png)

不像[Pub/Sub](pub-sub.md)模式，我們可以有不定數量的發佈者及訂閱者。