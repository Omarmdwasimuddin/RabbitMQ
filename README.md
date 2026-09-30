# RabbitMQ (NestJS Microservices)

**Source:** https://docs.nestjs.com/microservices/rabbitmq

[RabbitMQ](https://www.rabbitmq.com/) হলো একটা open-source, lightweight message broker, যেটা একাধিক messaging protocol support করে। High-scale, high-availability এর প্রয়োজন মেটাতে এটা distributed আর federated configuration এ deploy করা যায়, আর এটা সবচেয়ে বেশি widely deployed message broker গুলোর একটা — ছোট startup থেকে শুরু করে বড় enterprise, সবাই এটা ব্যবহার করে।

---

## 1. Installation

RabbitMQ-based microservice বানানো শুরু করার আগে, প্রথমে প্রয়োজনীয় package গুলো install করো:

```bash
npm i --save amqplib amqp-connection-manager
```

---

## 2. Overview

RabbitMQ transporter ব্যবহার করার জন্য, `createMicroservice()` method এ নিচের options object টা পাস করো:

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false,
    },
  },
});
```

> **Hint:** `Transport` enum টা `@nestjs/microservices` package থেকে import করা হয়।

---

## 3. Options

`options` property টা বেছে নেওয়া transporter অনুযায়ী নির্দিষ্ট। **RabbitMQ** transporter নিচের property গুলো expose করে।

| Property | বিবরণ |
|---|---|
| `urls` | ক্রমানুসারে try করার জন্য connection URL এর একটা array |
| `queue` | তোমার server যেই queue তে listen করে তার নাম |
| `prefetchCount` | Channel এর জন্য prefetch count সেট করে |
| `isGlobalPrefetchCount` | `true` হলে, prefetch count প্রতি consumer এর বদলে প্রতি channel এ apply হয় |
| `noAck` | `false` হলে, manual acknowledgment mode enable হয়। Default `true` |
| `consumerTag` | Consumer এর জন্য message delivery আলাদা করতে server যেই নাম ব্যবহার করে; এটা channel এ already ব্যবহৃত হওয়া চলবে না। সাধারণত এটা বাদ দেওয়াই সহজ — সেক্ষেত্রে server নিজে random একটা নাম তৈরি করে reply তে সরবরাহ করে (দেখো amqplib documentation এর [`channel.consume()`](https://amqp-node.github.io/amqplib/channel_api.html#channel_consume)) |
| `queueOptions` | অতিরিক্ত queue option (দেখো amqplib documentation এর [`channel.assertQueue()`](https://amqp-node.github.io/amqplib/channel_api.html#channel_assertQueue)) |
| `socketOptions` | অতিরিক্ত socket option (দেখো amqplib documentation এর [`connect()`](https://amqp-node.github.io/amqplib/channel_api.html#connect)) |
| `headers` | প্রতিটা message এর সাথে পাঠানো header। শুধু producer (client) configuration এর জন্য apply হয় |
| `replyQueue` | Producer এর জন্য reply queue। Default `amq.rabbitmq.reply-to` |
| `persistent` | Truthy হলে, message গুলো broker restart এর পরও টিকে থাকে, যদি সেগুলো এমন একটা queue তে থাকে যেটা নিজেও restart এ টিকে থাকে |
| `noAssert` | `true` হলে, consume করার আগে queue assert করা হয় না। Default `false` |
| `wildcards` | Message গুলো queue তে route করতে topic exchange ব্যবহার করতে চাইলে শুধু তখনই `true` সেট করো। এটা enable করলে message আর event pattern এ wildcard (`*`, `#`) ব্যবহার করা যায় |
| `exchange` | Exchange এর নাম। `wildcards` `true` সেট করা থাকলে default হয় queue এর নাম |
| `exchangeType` | Exchange এর type। Default `topic`। Standard AMQP type গুলো `direct`, `fanout`, `topic`, আর `headers` accept করে, অথবা একটা custom exchange type এর নাম |
| `exchangeArguments` | Exchange assert করার সময় পাস করা অতিরিক্ত argument |
| `routingKey` | Topic exchange এর জন্য অতিরিক্ত routing key |
| `maxConnectionAttempts` | Connection attempt এর সর্বোচ্চ সংখ্যা। শুধু consumer configuration এর জন্য apply হয়। Default `-1` (infinite) |

---

## 4. Client

অন্য microservice transporter এর মতোই, একটা RabbitMQ `ClientProxy` instance তৈরি করার জন্য তোমার কাছে [বেশ কিছু option](https://docs.nestjs.com/microservices/basics#client) আছে।

একটা instance তৈরি করার একটা উপায় হলো `ClientsModule` ব্যবহার করা। এটা import করো এবং এর `register()` method call করে উপরে `createMicroservice()` method এ দেখানো একই property গুলো সহ একটা options object পাস করো, সাথে injection token হিসেবে ব্যবহারের জন্য একটা `name` property। `ClientsModule` সম্পর্কে আরো জানতে [microservices overview](https://docs.nestjs.com/microservices/basics#client) দেখো।

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.RMQ,
        options: {
          urls: ['amqp://localhost:5672'],
          queue: 'cats_queue',
          queueOptions: {
            durable: false,
          },
        },
      },
    ]),
  ],
  // ...
})
```

`ClientProxyFactory` অথবা `@Client()` দিয়েও client তৈরি করা যায়, [microservices overview](https://docs.nestjs.com/microservices/basics#client) এ যেভাবে বলা আছে।

---

## 5. Context

আরো complex scenario তে, incoming request সম্পর্কে অতিরিক্ত তথ্য দরকার হতে পারে। RabbitMQ transporter ব্যবহার করার সময়, `RmqContext` object access করা যায়।

```typescript
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(`Pattern: ${context.getPattern()}`);
}
```

> **Hint:** `@Payload()`, `@Ctx()`, আর `RmqContext` — এগুলো `@nestjs/microservices` package থেকে import করা হয়।

Original RabbitMQ message (এর `properties`, `fields`, আর `content` সহ) access করতে, `RmqContext` object এর `getMessage()` method ব্যবহার করো:

```typescript
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getMessage());
}
```

RabbitMQ এর [channel](https://www.rabbitmq.com/channels.html) এর একটা reference পেতে, `RmqContext` object এর `getChannelRef()` method ব্যবহার করো:

```typescript
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getChannelRef());
}
```

---

## 6. Message Acknowledgement

কোনো message যাতে কখনো হারিয়ে না যায় সেটা নিশ্চিত করতে, RabbitMQ [message acknowledgement](https://www.rabbitmq.com/confirms.html) support করে। Consumer RabbitMQ কে জানাতে একটা acknowledgement পাঠায় যে একটা নির্দিষ্ট message receive এবং process হয়ে গেছে, আর RabbitMQ চাইলে সেটা delete করতে পারে। যদি কোনো consumer ack না পাঠিয়ে মারা যায় (তার channel বন্ধ হয়ে যায়, connection বন্ধ হয়ে যায়, অথবা TCP connection হারিয়ে যায়), তাহলে RabbitMQ ধরে নেয় message টা সম্পূর্ণভাবে process হয়নি এবং সেটা re-queue করে দেয়।

Default ভাবে, RabbitMQ transporter কোনো acknowledgement আশা করে না (`noAck` এর value `true`)। Manual acknowledgment mode enable করতে, `noAck` property কে `false` সেট করো:

```typescript
options: {
  urls: ['amqp://localhost:5672'],
  queue: 'cats_queue',
  noAck: false,
  queueOptions: {
    durable: false,
  },
},
```

Manual consumer acknowledgement on থাকলে, worker টা কাজ শেষ হয়েছে সেটা signal করতে অবশ্যই একটা acknowledgement পাঠাতে হবে:

```typescript
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
```

---

## 7. Record Builder

Message option configure করতে, `RmqRecordBuilder` class ব্যবহার করো (এটা event-based flow এর জন্যও কাজ করে)। যেমন, `headers` আর `priority` property সেট করতে, `setOptions()` method ব্যবহার করো:

```typescript
const message = ':cat:';
const record = new RmqRecordBuilder(message)
  .setOptions({
    headers: {
      ['x-version']: '1.0.0',
    },
    priority: 3,
  })
  .build();

this.client.send('replace-emoji', record).subscribe(...);
```

> **Hint:** `RmqRecordBuilder` class টা `@nestjs/microservices` package থেকে export করা হয়।

Server side এও, `RmqContext` access করে এই value গুলো read করা যায়:

```typescript
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: RmqContext): string {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

---

## 8. Instance Status Update

Connection এবং underlying driver instance এর state সম্পর্কে real-time update পেতে, `status` stream এ subscribe করো। এই stream বেছে নেওয়া driver অনুযায়ী নির্দিষ্ট status update দেয়। RMQ driver এর ক্ষেত্রে, `status` stream `connected`, `disconnected`, `blocked`, আর `unblocked` — এই চার ধরনের event emit করে।

```typescript
this.client.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

> **Hint:** `RmqStatus` type টা `@nestjs/microservices` package থেকে import করা হয়।

একইভাবে, server এর status সম্পর্কে notification পেতে, server এর `status` stream এও subscribe করা যায়:

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

---

## 9. RabbitMQ Event Listen করা

কিছু ক্ষেত্রে, microservice এর emit করা internal event গুলো listen করার দরকার হতে পারে। যেমন, কোনো error ঘটলে অতিরিক্ত operation trigger করার জন্য `error` event এর জন্য listen করতে পারো। এর জন্য, `on()` method ব্যবহার করো:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

একইভাবে, server এর internal event ও listen করা যায়:

```typescript
server.on<RmqEvents>('error', (err) => {
  console.error(err);
});
```

> **Hint:** `RmqEvents` type টা `@nestjs/microservices` package থেকে import করা হয়।

---

## 10. Underlying Driver Access

আরো advanced use case এ, underlying driver instance access করার দরকার হতে পারে — যেমন, manually connection close করা বা driver-specific method ব্যবহার করা। তবে, বেশিরভাগ ক্ষেত্রে driver directly access করার **দরকার হয় না**।

এর জন্য, `unwrap()` method ব্যবহার করো, যেটা underlying driver instance return করে। তুমি কোন type এর driver instance আশা করছো সেটা বলতে generic type parameter ব্যবহার করো।

```typescript
const managerRef =
  this.client.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```

একইভাবে, server এর underlying driver instance ও access করা যায়:

```typescript
const managerRef =
  server.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```

---

## 11. Wildcards

Flexible message routing এর জন্য RabbitMQ, routing key তে wildcard ব্যবহার support করে। `#` wildcard শূন্য বা তার বেশি word match করে, আর `*` wildcard ঠিক একটা word match করে।

যেমন, `cats.#` routing key `cats`, `cats.meow`, আর `cats.meow.purr` — সবগুলোর সাথে match করে। আর `cats.*` routing key `cats.meow` এর সাথে match করে, কিন্তু `cats.meow.purr` এর সাথে করে না।

তোমার RabbitMQ microservice এ wildcard support enable করতে, options object এ `wildcards` configuration option কে `true` সেট করো:

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.RMQ,
    options: {
      urls: ['amqp://localhost:5672'],
      queue: 'cats_queue',
      wildcards: true,
    },
  },
);
```

এই configuration দিয়ে, event আর message এ subscribe করার সময় routing key তে wildcard ব্যবহার করা যায়। যেমন, `cats.#` routing key দিয়ে আসা message listen করার জন্য:

```typescript
@MessagePattern('cats.#')
getCats(@Payload() data: { message: string }, @Ctx() context: RmqContext) {
  console.log(`Received message with routing key: ${context.getPattern()}`);

  return {
    message: 'Hello from the cats service!',
  };
}
```

একটা নির্দিষ্ট routing key দিয়ে message পাঠাতে, `ClientProxy` instance এর `send()` method ব্যবহার করো:

```typescript
this.client.send('cats.meow', { message: 'Meow!' }).subscribe((response) => {
  console.log(response);
});
```
