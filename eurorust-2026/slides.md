---
theme: seriph
background: https://images.unsplash.com/photo-1732883894688-5723938da462

title: Lessons from building a background task processing library
info: |
  apalis, from v0.1 (2021) to v1.0.0 (2026)

class: text-center
transition: slide-left
comark: true
duration: 30min
---


# Just use `tokio::spawn` bro

**or: how I rewrote the same trait five times**

<div class="mt-8 opacity-90">Lessons from building a background task processing library</div>

<!--
~2 min. Open with: "Who here has been told to just tokio::spawn it?" Hands up. Keep them up.
"Good. This talk is for you. Also for the person who said it."
-->

---
layout: center
---

# The comment that started it all

<div class="bg-black text-white p-6 rounded-lg text-2xl max-w-2xl mx-auto text-left">

> Just use `tokio::spawn` + `channel` bro!
>
> <span class="opacity-60 text-base">some guy on Reddit</span>

</div>

```rs
tokio::spawn(async move {
  let _ = tx.send("task 1");
  let _ = tx.send("task 2");
});

while let Some(item) = rx.recv().await {
  println!("Received: {}", item);
}
```

<v-clicks class="mt-8 text-left max-w-2xl mx-auto">

- Process restarts? Your jobs are gone 👻
- Job fails? Nobody retries it 🤷
- Deploy at 5pm Friday? Pray 🙏

</v-clicks>

<!--
~2 min. Be fair to the Reddit guy: for lots of things spawn is exactly right.
It stops being right the moment you care whether the work actually finished.

Interaction: Ask the room: who has lost a job to a restart?
-->

---
layout: center
---

# Who am I?

I am Njuguna Mureithi:
- Kenyan based in Nairobi
- Formerly a Food scientist
- Self taught software engineer
- Previously Worked on Fintech, Cryptography and Systems design
- Plays football, and enjoys nature

---
layout: center
---

# What is apalis?

apalis is a simple background task and messages processing library for Rust

## What does it offer?

- Task handlers as async functions, no macros
- Built-in task management retries, rate limiting, timeouts, idempotency, and error handling
- Lightweight and lazy, workers do nothing until polled or woken: no busy-waiting
- Dependency injection - akin to popular frameworks like [`actix`] and [`axum`]
- Production ready with monitoring, metrics, graceful shutdown, and comprehensive error reporting
- Extensible with middleware based on [`tower`] 

<!--
~1 min: Highlight some of the features, but explain we are focussing on Worker, Task and Backend today

Interaction: Anyone using apalis?
-->

---

# The holy trinity (of background jobs)

<div class="grid grid-cols-3 gap-6 mt-10">
  <div class="p-6 rounded-lg bg-gray-500/10">
    <h3 class="text-orange-400">📦 Task</h3>
    <p>The thing to do later.</p>
    <p class="opacity-60 text-sm">args · id · attempts · context</p>
  </div>
  <div class="p-6 rounded-lg bg-gray-500/10">
    <h3 class="text-orange-400">🏃 Worker</h3>
    <p>The thing that does it.</p>
    <p class="opacity-60 text-sm">concurrency · shutdown · restarts</p>
  </div>
  <div class="p-6 rounded-lg bg-gray-500/10">
    <h3 class="text-orange-400">🗄️ Persistence</h3>
    <p>The place it waits.</p>
    <p class="opacity-60 text-sm">push · fetch · ack · retry</p>
  </div>
</div>


<!--
~3 min. Interaction: Ask the room: which one would you build first? (Wrong answer: all three at once. Ask me how I know.)

Most people say persistence. Fine. The twist: the hard part was never any one box,
it was the arrows between them.
-->

---
layout: two-cols
---

# Attempt 1: just use actors

```rust
// v0.1 worker/mod.rs
trait Worker: Send + Sized + 'static

struct Addr<A: Actor>
struct Recipient<M: Message>

impl<T, W> Handler<JobRequestWrapper<T>>
  for W   // a job arrives as a message
```


::right::

<v-clicks class="mt-12">

- **Why?:** I already had actix. A `Worker` has state and gets messages. Match made in heaven, right?
- **Why not?:** a queue has *one* message type. I was paying for `Addr`, `Recipient` and mailboxes and using none of them.
- **Kept:** the Arbiter idea. Something supervises the workers. That became the `Monitor`.

</v-clicks>

<div class="mt-6 p-3 rounded bg-black text-white text-center font-bold uppercase">
  Using a flamethrower to light a candle 🔥🕯️
</div>

<!--
~4 min. Actors were not wrong, just more than the problem needed.
Lesson: steal the idea (supervision), not the whole model.

Interaction: Ask the room: ever borrowed a whole framework for one idea?
-->

---
layout: two-cols
clicks: 3
---

# Persistence: one trait to rule them all

```rust {all|3-5,7,9|all|0}{at:0}
trait Storage: Clone {
  type Output: Job;
  async fn push(..);
  async fn schedule(.., on);
  async fn consume(.., interval);
  async fn ack(..);
  async fn retry(..);
  async fn kill(..);
  async fn heartbeat(..);
  async fn reschedule(..);
  // ...and more
}
```

::right::

<v-clicks class="mt-12">

- **The trait did everything:** store tasks, deliver them, and track worker liveness.
- **Adding a backend:** implement everything, or nothing.
- **Also** I introduced `MessageQueue` for `enqueue` and `dequeue`. Two fat traits. Congratulations to me.

</v-clicks>

<div class="mt-6 p-3 rounded bg-black text-white text-center font-bold uppercase">
  Implement 12 methods 😰 &nbsp;|&nbsp; Implement 12 methods 😰
</div>

<!--
~3 min. Invite war stories. The root cause is coupling: every change to one concern touched
every backend. The two-button meme: both buttons say the same thing, the sweating is real.

Interaction · Ask the room: what is the fattest trait you have ever implemented?
-->

---
layout: two-cols
---

# Are we there yet?

```rust
// Storage, v0.3
fn consume(
  &mut self,
  worker_id: &String,
  interval: Duration,
) -> JobStreamResult<Self::Output> {

  loop {
    interval.next().await;
    /// Fetch from DB
    /// TODO: use reddit guy's channel
  }

}
```

::right::

<v-clicks class="mt-12">

- The backend is told *how often* to look. Never *when* there is work.
- **Short interval:** burn CPU checking an empty queue.
- **Long interval:** tasks sit around waiting for the next tick.
- Either way, nobody can wake the worker when a task lands.

</v-clicks>

<div class="mt-6 p-3 rounded bg-black text-white text-center font-bold uppercase">
  Worker: "Is there work yet?" &nbsp; DB: "No." &nbsp; Worker: "Is there work yet?"
</div>

<!--
~3 min. The pivot of the talk. Road-trip kid in the back seat. Everything from here on is
replacing "how often" with "when".
-->

---
layout: two-cols
---

# Attempt 3: streams to the rescue

```rust
// v0.5.0-alpha, Feb 2024
trait Backend<Req> {
  type Stream: Stream<
    Item = Result<Option<Req>, Error>>;
  type Layer;
  fn layer(&self, w: &WorkerId)
    -> Self::Layer;
  fn poll(self, w: &WorkerId)
    -> Poller<Self::Stream> {
      let stream = stream::unfold({ ... }); // example
      Poller::new(stream)
    }
}
```

::right::

<v-clicks class="mt-12">

- A backend is just a thing that yields tasks. Pass a stream/compose a stream.
- Streams provide better control.
- **Problem:** Our sources are not streams (ideally)
- **Problem:** No way to communicate with the worker


</v-clicks>

<div class="mt-6 grid grid-cols-2 gap-2 text-sm">
  <div class="p-3 rounded bg-red-500/20">😖 Invent my own task delivery protocol</div>
  <div class="p-3 rounded bg-green-500/20">😎 Use `Stream` like everyone else</div>
</div>

<!--
~3 min. First appearance of Backend. The Layer type lets a backend decorate the worker service.
The two-row Drake format is on the slide in text form: reject the left, approve the right.

Interaction: Ask the room: stream or trait object, what would you pick?
-->

---
layout: two-cols
---

# It worked! ...mostly

````md magic-move
```rust
// v0.5
trait Backend<Req> {
  type Stream: Stream<Item = Result<Option<Req>, Error>>;
  type Layer;
  fn poll(self, w: &Worker) -> Poller<Self::Stream>;
}
```
```rust
trait Backend<Req, Res> {
  type Stream: Stream<Item = Result<Option<Req>, Error>>;
  type Layer;
  fn poll<Svc: Service<Req, Response = Res>>(
    self, worker: &Worker,
  ) -> Poller<Self::Stream, Self::Layer>;
}
```
```rust
trait Backend<Req> {
  type Stream: Stream<Item = Result<Option<Req>, Error>>;
  type Layer;
  type Codec: Codec;
  fn poll(self, worker: &Worker)
    -> Poller<Self::Stream, Self::Layer>;
}
```
````

::right::

<v-clicks class="mt-4">

- **`poll(self)` consumes the backend,** so you can't react when something is pushed.
- **Every gap = one more associated type.** Gotta catch 'em all.
- **No connection between worker and backend** Backend may poll when worker is not ready

</v-clicks>

<div class="mt-4 p-3 rounded bg-black text-white text-center font-bold uppercase">
  🔥 This is fine 🔥 &nbsp; (production was, in fact, fine)
</div>

<!--
~3 min. Interaction: Ask the room: when does a trait need a redesign, not another type?

Honest middle. People ran this in production. The smell: fixing limitations by adding
parameters instead of questioning the model.

-->

---
layout: two-cols
clicks: 3
---

# Don't call us, we'll call you

So what is the solution?

Almost like `Future`, `Stream` and `Sink`

```rust {all|6-7|2-4|5-6}{at:0}
// sketch, not the real source
fn poll_next(&mut self, cx, worker) {
  if let Some(t) = self.take_ready() {
    return Poll::Ready(Some(Ok(t)));
  }
  self.register(cx.waker());
  Poll::Pending
}
```

::right::

<v-clicks class="mt-12">

- **Nothing to do?** Return `Pending`, store the waker, go to sleep. No timer. No loop.
- **Cost of an idle queue:** roughly zero.
- **Task pushed?** Something calls `wake()`. The executor polls again. Task flows out.

</v-clicks>

<div class="mt-6 p-3 rounded bg-gray-500/10">
  🚪 Checking the door every minute vs. 🔔 waiting for the doorbell
</div>

<!--
~4 min. Interaction: Ask the room: who has debugged a busy-loop poller at 3am?

Slow down. If the room knows Future and Waker, make them explain it. If not,
use the doorbell. Stress: this is the real reason 1.0 exists.
-->

---
layout: two-cols
clicks: 4
---

# Three methods. Three.

```rust {all|1-3|4-5|6-8|9-11|all}{at:0}
trait Backend: Sized {
  type Task;
  type Error;
  fn poll_ready(&mut self, cx, worker)
    -> Poll<Result<(), Self::Error>>;
  fn poll_next(&mut self, cx, worker)
    -> Poll<Option<Result<Self::Task,
                          Self::Error>>>;
  fn poll_close(&mut self, cx, worker)
    -> Poll<Result<(), Self::Error>>;
}
```

::right::

<v-clicks class="mt-12">

- **Associated types:** What does this backend emit?
- **`poll_ready`:** can I take work?
- **`poll_next`:** give me a task, or wake me when there is one.
- **`poll_close`:** tidy up, we're leaving.
- Config, encoding and pushing moved out: `BackendConfig`, `WireFormatBackend`, `TaskSink`.

</v-clicks>

<div class="mt-6 opacity-70">Down from a dozen methods. I did not cry. (I cried.)</div>

<!--
~2 min. Compare with the Storage slide: twelve methods against three.

Interaction: Ask the room: what would you cut from this?
-->

---
layout: two-cols
clicks: 5
---

# Ready. Ready. Run.

```rust {all|1-3|6-8|9-11|13-15|all}{at:0}
impl Stream for Worker
{
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<Self::Item>>
    {
        // 1. Backend ready?
        ready!(backend.poll_ready(cx))?;

        // 2. Service ready?
        ready!(service.poll_ready(cx))?;

        // 3. Get task, run service
        let task = ready!(backend.poll_next(cx))?;
        queue.push(service.call(task));
    }
}
```

::right::

<v-clicks class="mt-12">

- **Worker as Stream** emits `Event`
- **Backend ready:** can I take work?
- **Service ready:** can I process it?
- **Then fetch:** only pull a task when both are ready.
- Responses come from the queue; shutdown drains it before closing the backend.

</v-clicks>

<div class="mt-6 opacity-70">Two gates before one task. No pushing work into a service that isn't ready.</div>

<!--
~2 min. Walk through the order: backend readiness, service readiness, then fetch.
The ordering is deliberate: don't pull work until both sides can accept it.

Interaction: Ask the room: would you fetch first and buffer, or wait for both sides?
-->

---
layout: two-cols
---

# It's wrappers all the way down 🐢

With the `Backend` trait, composition becomes easy.

```rust
impl<B, F> Backend for InspectErr<B, F>
    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>,
        worker: &WorkerContext,
    ) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx, worker).map_err(|err| {
            (self.f)(&err);
            err
        })
    }
```

::right::

<v-clicks class="mt-12">

- Each one is a `Backend` wrapping a `Backend`.
- Behaviour gets **added**, not baked in.
- The control we lacked in the stream era (react to a push!) is now just a wrapper.
- You can now have composition eg `BeforeStart`, `MapErr`, `PollWith`

</v-clicks>

<!--
~2 min. Let someone pick a wrapper and sketch it live. Cheap interactivity, good laughs.

Interaction: Ask the room: which wrapper would you write first?
-->

---
layout: two-cols
clicks: 4
---

# Task: please stop implementing `Job`

````md magic-move {at:2}
```rust
// v0.3
trait Job {
  const NAME: &'static str;
}
```
```rust
// v0.6
struct Request<Args, Ctx> {
  pub args: Args,
  pub parts: Parts<Ctx>,
}
```
```rust
// v1.0.0-beta.1
struct Task<Args, Context, IdType = RandomId> {
  pub args: Args,
  pub parts: Parts<Context, IdType>,
}
```
```rust
// v1.0.0
struct Task<Args> {
  pub args: Args,
  ctx: Arc<ExecutionContext>,
}
```
````

::right::

<v-clicks class="mt-4">

- **Before:** every payload needed a trait impl with a `NAME`.
- **Then:** your struct, plain and simple. Carries info like `queue` in Parts.
- **Later:** Each Backend has a `<Ctx>` and `<Id>`.
- **Finally:** Generics went from three to one. Have a single type for Execution context.

</v-clicks>

<div at="1" class="mt-4 p-3 rounded bg-black text-white text-center font-bold uppercase">
  "Please implement Job for your struct." &nbsp; Users: 😐
</div>


<!--
~2 min. The dual approach from the abstract: both existed for a while, trait-free won on
ergonomics.

Interaction: Ask the room: trait-based or trait-free, which side are you on?
-->

---
layout: two-cols
clicks: 4
---

# Worker: what you use

````md magic-move {at:1}
```rust
/// A function called for every task
async fn send_email(email: Email) {
  // execute task
}
WorkerBuilder::new("email-worker")
  .backend(storage)
  .build(send_email)
  .run();
```
```rust
/// A function called for every task
async fn send_email(email: Email) {
  // execute task
}
WorkerBuilder::new("email-worker")
  .backend(storage)
  .layer(retry) // Add middleware
  .catch_panic() // Use builtin middleware
  .build(send_email)
  .run();
```
```rust
/// A function called for every task
async fn send_email(email: Email, task: TaskContext) {
  if !email.is_valid() {
    task.stop().unwrap();
  }
}
WorkerBuilder::new("email-worker")
  .backend(storage)
  .layer(retry) // Add middleware
  .catch_panic() // Use builtin middleware
  .data(pool) // Deps injection
  .build(send_email)
  .run();
```
```rust
/// A function called for every task
async fn send_email(email: Email, pool: Data<Pool>) {
  // execute task
}
WorkerBuilder::new("email-worker")
  .backend(storage)
  .layer(retry) // Add middleware
  .catch_panic() // Use builtin middleware
  .data(pool) // Deps injection
  .build(send_email)
  .run();
```
```rust
/// A function called for every task
async fn send_email(email: Email, pool: Data<Pool>) {
  // execute task
}
WorkerBuilder::new("email-worker")
  .backend(storage)
  .layer(retry) // Add middleware
  .catch_panic() // Use builtin middleware
  .data(pool) // Deps injection
  .build(send_email)
  .run_until(ctrl_c()); // Graceful shutdown
```
````

::right::

<v-clicks class="mt-12">

- **Middleware:** a worker drives service. Retries, timeouts, tracing = layers.
- **Ergonomics:** accepts any tower service including a plain async function.
- **Injection:** `data(...)` instead of globals. Your tests will thank you.
- **Shutdown:** You can use `run_until_*` methods; `WorkerContext` can pause, resume, stop.

</v-clicks>

<!--
~3 min. Shape of the builder, not copy-paste code (retry is a placeholder layer).
Cover rolling restarts here.

Interaction: Ask the room: how do you shut down gracefully today?
-->

---
layout: center
class: text-center
---

# So where are we now?

<v-clicks class="text-left mt-8">

- **Stable Core API:**: Worker, Task and Backend are stable.
- **Composition Exts:** Like `map_err`, `shared`, `with_codec` etc
- **Popular integrations:** Postgres, Sqlite, Redis and many more.

- **We are looking for sponsors:** (Does opensource even pay?)

</v-clicks>

<!--
~2 min
-->

---
layout: center
class: text-center
---

# Four lessons (no sticker, sorry)

<v-clicks class="text-left mt-8">

- **Split the concerns.** Fat traits make everyone pay.
- **Wake, don't poll.** Let the runtime ring the doorbell.
- **Compose, don't configure.** Wrappers beat flags.
- **Reuse what exists.** `Stream`, `Sink`, tower. They're free.

</v-clicks>

<!--
~2 min. Land the plane. None of this came from a perfect first design: a model that worked,
a limit that showed up, a redesign.
-->

---
layout: center
class: text-center
---

# Questions? 🙋

Yes, you can still `tokio::spawn`. I won't judge.

<div class="opacity-60 mt-4">(I will, a little.)</div>

<div class="mt-8 opacity-70">Which design would you have shipped first?</div>

<!--
Open Q&A. Keep a spare joke for the silence: "Nobody? The Reddit guy is shy."
-->
