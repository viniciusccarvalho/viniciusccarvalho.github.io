---
layout: post
title: "RXFire - Gemfire meets RX"
date: 2015-04-07 18:14:45 -0400
comments: true
categories: 
---

Gemfire meets RXJava
--------------------

[Gemfire](http://pivotal.io/big-data/pivotal-gemfire) is a very powerful IMDG solution. One of the many aspects that differentiate it from others in the space, is the great support for Continuous Queries.

CQ allows developers to subcribe to regions on the data grid and receive live notifications when changes happen. There are several examples of applications that could be built using this feature, from
CEP to creating data streams.

Since [RX Java](https://github.com/ReactiveX/RxJava/wiki) and [reactive-streams](http://www.reactive-streams.org/) are very hot in the moment, I decided that I should play around the idea of creating
streams of data from a gemfire region into an Observable and then be able to play with it a little.

### Introduction

The project can be found on [github](https://github.com/viniciusccarvalho/rxfire), I'll focus on some of the key classes that were implemented for this prototype.

``` java
public class SubscriberEventAdapter implements EventAdapter{

    final Subscriber subscriber;

    public SubscriberEventAdapter(Subscriber subscriber){
        this.subscriber = subscriber;
    }


    @Override
    public void handleEvent(CqEvent event) {
        subscriber.onNext(event);
    }


}


```

This first class is just a simple adapter. It translates the gemfire reflection based query listener (handleEvent method) and invokes the RX subscriber that would be subscribing for events from this region.

``` java

@Component
public class StockStreamService {

    @Autowired
    private ContinuousQueryListenerContainer cqlc;

    public Observable<CqEvent> toObservable(String query){

        return Observable.create((subscriber) -> {
            ContinuousQueryDefinition def = new ContinuousQueryDefinition(query, new ContinuousQueryListenerAdapter(new SubscriberEventAdapter(subscriber)));
            cqlc.addListener(def);

        });
    }

}

```
This is where the rubber meets the road. When an observable its created it registers a query listener within the container and that listener invokes this Observable's subscriber on events.

### Using

It really could not be simpler, there are things to be done on subscription handling and multiple subscribers, but in a nutshell that's pretty much it. Now a simple example test:

``` java

 @Test
    public void testStreamAggregation() throws Exception {
        Observable<CqEvent> observable = streamService.toObservable("SELECT * FROM /stocks t where t.value > 1");
        observable.map((event) -> {
            return (Stock) ((PdxInstance) event.getNewValue()).getObject();
        }).filter((stock) -> {
            return stock.getId().equals("PVT");
        }).take(5).subscribe((stock) -> {
                    System.out.println(stock);
                }, (error) -> {
                    error.printStackTrace();
                }
        );


        latch.await();
    }

```

This stream is very basic, but it shows some basic functionalities of RX and why you should be excited on morphing your listeners into observables.

The first line we create an Observable that listens for changes on a particular query. 

This observable first maps object of type CqEvent<PDXInstance> -> Stock

Then we filter to only consume stocks of type 'PVT'

And finally we only take the first 5 stocks of this stream.

The subcriber part will then act on those 5 stocks, here just printing them to the console.

The real power of RX comes when you start combining multiple streams, imagine for a moment you had a second or a third stream from different gemfire regions, or even, as from my previous [post](http://vcc.devguild.us/blog/2014/11/13/rxjava-simple-concurrency-usage/) you could combine a stream from MongoDB or from Twitter. Once you move your Iterables into Observables a new horizon opens for you.

Cheers

 
