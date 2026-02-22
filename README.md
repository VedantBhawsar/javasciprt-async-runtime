# Mini Async Runtime

A lightweight, browser-based cooperative scheduler built with JavaScript **generator coroutines**. Demonstrates how an async runtime can be implemented from scratch — with microtask/macrotask queues, timers, Promise interop, priority scheduling, and task cancellation — all in a single HTML file with no dependencies.

## Demo

Open `index.html` directly in your browser — no build step or server required.

## How It Works

Tasks are **generator functions** (`function*`) that `yield` instructions to the runtime. The scheduler inspects each yielded value and decides what to do next:

| Yielded value | What the runtime does |
|---|---|
| `sleep(ms)` | Registers a timer; resumes the task after `ms` milliseconds |
| `yieldNow()` | Requeues the task as a microtask (cooperative yield) |
| A `Promise` | Waits for the promise to settle, then resumes with the resolved value |
| Any other value | Resumes immediately as a microtask, passing the value back |

This is **cooperative concurrency**: tasks must yield periodically to allow other tasks to run.

### Queue model

```
┌──────────────────────────────────────────────────────┐
│                    AsyncRuntime                       │
│                                                       │
│  microtaskQueue  ← continuations & promise results    │
│  macrotaskQueue  ← user-enqueued callbacks            │
│  timerQueue      ← sorted by wake-up time             │
└──────────────────────────────────────────────────────┘
```

On each tick the runtime:
1. Moves expired timers into the microtask queue.
2. Processes up to `maxMicroPerTick` (default 50) microtasks.
3. Runs one macrotask if the microtask queue is empty.
4. Reschedules the next tick via `setTimeout(0)` to let the browser breathe.

## API

### Helper functions

```js
sleep(ms)   // yield this to pause a task for ms milliseconds
yieldNow()  // yield this to cooperatively hand back control
```

### `AsyncRuntime`

```js
const runtime = new AsyncRuntime({ logCallback: fn });
```

| Method | Description |
|---|---|
| `runtime.start()` | Begin (or resume) the event loop |
| `runtime.stop()` | Pause the event loop (does not cancel tasks) |
| `runtime.spawn(genFn, opts)` | Spawn a new coroutine task; returns a `Task` |
| `runtime.enqueueMacrotask(fn)` | Push a plain callback onto the macrotask queue |

**`spawn` options**

| Option | Type | Default | Description |
|---|---|---|---|
| `priority` | `number` | `0` | Higher values run before lower values in the microtask queue |
| `name` | `string` | `"task#<id>"` | Human-readable label shown in logs and the UI |

### `Task`

| Member | Description |
|---|---|
| `task.asPromise()` | Returns a `Promise` that resolves/rejects when the task finishes |
| `task.cancel()` | Cancels the task immediately and rejects its promise |
| `task.done` | `true` once the generator returns or the task is cancelled |
| `task.priority` | Scheduling priority |

## Usage Examples

### CPU-bound task (yields every chunk)

```js
function* cpuTask(name = 'cpu', chunks = 40) {
  for (let c = 0; c < chunks; c++) {
    let s = 0;
    for (let i = 0; i < 20000; i++) s += Math.sqrt(i + c);
    yield yieldNow(); // cooperatively yield after each chunk
  }
  return `${name} finished`;
}

const task = runtime.spawn(() => cpuTask('my-cpu'), { priority: 1 });
runtime.start();
task.asPromise().then(result => console.log(result));
```

### IO-bound task (sleep + Promise interop)

```js
function* ioTask(name = 'io', waitMs = 1500) {
  yield sleep(waitMs); // pause for a timer

  // yield a real Promise — runtime resumes when it resolves
  const result = yield fetch('/api/data').then(r => r.json());
  console.log('got:', result);
  return result;
}

runtime.spawn(() => ioTask('my-io'), { name: 'my-io' });
runtime.start();
```

### Cancelling a task

```js
const task = runtime.spawn(longRunningGen, { name: 'work' });
runtime.start();

setTimeout(() => task.cancel(), 2000); // cancel after 2 s
task.asPromise().catch(err => console.log(err.message)); // "Task cancelled"
```

## UI Controls

| Button / Control | Description |
|---|---|
| **Start Runtime** | Starts the event loop |
| **Stop Runtime** | Pauses the event loop |
| **Spawn CPU Task** | Adds a new CPU-bound coroutine |
| **Spawn IO Task** | Adds a new IO-bound coroutine |
| **Priority** selector | Sets the priority for the next spawned task |
| **Clear Log** | Clears the runtime event log |
| **Cancel** (per task) | Cancels an individual running task |

The live dashboard updates every 200 ms and shows the number of active tasks, pending microtasks, macrotasks, and timers.

## Project Structure

```
index.html   — entire runtime + demo UI (single self-contained file)
```

## Key Design Decisions

- **No dependencies** — pure browser JavaScript, no frameworks or bundlers.
- **Cooperative scheduling** — tasks control when they yield; the runtime never preempts.
- **Priority queue** — microtasks are sorted by priority (higher first), then FIFO within the same priority.
- **Promise interop** — yielding any thenable automatically integrates with the native Promise system.
- **Safety cap** — `maxMicroPerTick = 50` prevents a flood of microtasks from starving the browser UI thread.

## License

MIT
