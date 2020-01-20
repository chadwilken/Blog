---
path: sidekiq-to-kafka-and-back
date: 2020-01-19T22:24:59.498Z
title: Sidekiq to Kafka…and Back
description: 'Our journey from Sidekiq, to Kafka and back to Sidekiq'
---
As developers, we all love to try new tools and services when building features. Even more so when the tool seems like it was _built_ for what you’re trying to accomplish. New tools are exciting and sorta _sexy_, plus you look ~~smart and cutting-edge~~ when you write stupid articles like this one. This is a tale of considering trusty ole’ Sidekiq, choosing and building a feature with Kafka, and then ending up back with Sidekiq. It also touches on how you can write code so that it requires minimal changes when there's a **big** requirement change.

Recently I was working on a project that involved replicating a subset of records to Mongo after various events happened. We use Mongo to store an _active set_ of photos and videos that belong to _active_ projects. When a project is archived or a photo changes state by being created, deleted, or restored, we have to reflect that change in Mongo. Said another way, an _event_ happens and _something_ cares about that event and _performs_ an action. When you break down these requirements, it sounds like something Kafka was built for. The project homepage describes itself as:

> “Kafka is used for building real-time data pipelines and streaming apps. It is horizontally scalable, fault-tolerant, wicked fast, and runs in production in thousands of companies.”.

Well that sounds exactly like the tool we _need_…

![I need it](https://media.giphy.com/media/cALkoAIov3Y9a/source.gif)

## Sidekiq 💔

First, let me say how much I love Sidekiq. **I love Sidekiq**, there I said it. We use Sidekiq to process millions of jobs per day at CompanyCam. It is one of the best open-source projects in the Ruby ecosystem, in my opinion.

So, why not Sidekiq? It is horizontally scalable, wicked fast, and runs in production in thousands of companies, ours included. To be honest, I wish I had a better answer than "I dunno". Seriously though, I thought we could benefit from using Kafka at some point _in the future_. I thought some of the features of Kafka could be useful, like an ordered log of events. But in the end I was guilty of premature optimization.

![oof](https://i.kym-cdn.com/entries/icons/original/000/032/425/Screen_Shot_2020-01-14_at_10.34.57_AM.jpg)

## Kafka’s Shortcomings

I implemented the feature, tested it, and set it free. Soon after I began noticing a few issues. By default Kafka doesn't allow the automatic creation of topics. You have to create the topic before you can start writing events to it. This can have some weird side-effects. Say you add a new event topic and push it live, but forget to create the new topic in production. When you attempt to produce events to that topic you will receive an error that the topic doesn't exist. [DeliveryBoy](https://github.com/zendesk/delivery_boy), the gem we use to produce events, can handle this to an extent, but doesn’t allow for easy resolution. Other things such as networking issues can and will pop up as well. If you don't plan ahead for these events your message buffer will overflow and you will begin to lose events. There are other limitations as well such as adding partitions or consumers causes rebalancing and can slow things down temporarily.

The first time we experienced a networking issue I began to think of ways to mitigate this. I could catch the exception, write the messages to Redis, and then have a job reprocess failed deliveries every few minutes. Even if I fixed this issue, [Racecar](https://github.com/zendesk/racecar) is multi-process but single-threaded so concurrency _costs_ more than Sidekiq. This made the events extremely slow during our peak times. It was at this moment I started to realize that maybe Kafka wasn't actually what we needed.

## Revert

I still wanted the ability to produce events from anywhere, asynchronously, and without caring about what the payload _looked_ like. We already use the [Wisper](https://github.com/krisleech/wisper) gem along with the [sidekiq broadcaster](https://github.com/krisleech/wisper-sidekiq) for async processing. We were able to reuse those gems pretty easily without much refactoring.

During the initial implementation I abstracted away the logic of producing and consuming events. It isn't the responsibility of the classes to care _where_ to read or write data. The producers share a superclass, `BaseProducer`, that would deliver the event when `produce` was called.

```ruby
class BaseProducer
  def self.call(*args)
    new.call(*args)
  end

  def produce(data, topic:)
    begin
      DeliveryBoy.deliver_async!(data.to_json, topic: topic)
    rescue Kafka::BufferOverflow => error
      Rails.logger.error "Message for '#{topic}' dropped due to buffer overflow"
    end
  end
end

module Assets
  class DiscardedProducer < BaseProducer

    def call(asset)
      data = {
        id: asset.id,
        type: asset.class.name
      }
      produce(data, topic: Events::Topics::ASSET_DISCARDED)
    end

  end
end

# In your controller/service
# Assets::DiscardedProducer.call(photo)
```

Likewise, the consumers shared the `BaseConsumer` superclass that would parse the message and call the method matching the topic's name.

```ruby
class BaseConsumer < Racecar::Consumer
  def process(message)
    data = JSON.parse(message.value)
    send(message.topic.to_sym, data)
  rescue JSON::ParserError => error
    Rails.logger.error(error)
  end
end

class AssetConsumer < BaseConsumer
  subscribes_to Events::Topics::ASSET_DISCARDED

  def asset_discarded(data)
    # logic to delete asset from Mongo
  end

end
```

This made the change super easy. I was able to change `BaseProducer` class to:

```ruby
class BaseProducer
  include Wisper::Publisher

  def self.call(*args)
    new.call(*args)
  end

  def produce(data, event:)
    broadcast(event.to_sym, data)
  end
end
```

Since Wisper already parses the data and calls the method with the same name as the event, the `BaseConsumer` class could be deleted. The only thing required to hook up the events to the corresponding consumer was to create an initializer:

```ruby
# wisper.rbWisper.subscribe(
  AssetListener,
  broadcaster: :sidekiq,
  on: [
    :asset_discarded,
    # ...other events
  ]
)
```

I ran the specs to make sure they were green and then deployed to production. Events began to process much faster, and I was able to kill two consumer servers as a side-benefit.

If you're still reading, nice. I know that some of these things, such as the performance can be mitigated. We used shared hardware with ConfluentCloud rather than running dedicated hardware. We could have added more servers and consumers, but those things cost money. We already use Sidekiq and have spare capacity, it just makes sense to keep using it. It is safe, reliable, fast, and ultimately the right choice for what we are trying to accomplish.

P.S. I'm just getting back into writing, sorry about the rambling.
