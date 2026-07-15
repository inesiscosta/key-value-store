# Key Value Store

A concurrent key-value store server written in C. It processes batches of commands from job files with a pool of threads, takes non-blocking backups with `fork`, and lets clients connect over named pipes to subscribe to keys and get notified when their values change.

Built with Miguel Raposo for the Operating Systems course at Instituto Superior Técnico (2024/2025).

## How it works

- **Hash table:** keys are stored in a 26-bucket hash table, and each bucket has its own read-write lock. A command locks only the buckets its keys hash to, always in the same order, so commands on different keys run in parallel without deadlocks.
- **Jobs:** the server reads every `.job` file in a directory and spreads them across up to `max_threads` threads. Each job's output goes to a matching `.out` file.
- **Backups:** `BACKUP` forks the process, and the child writes a snapshot of the table to a `-N.bck` file while the parent keeps going. A semaphore caps how many backups can run at once.
- **Clients:** a client registers through the server's FIFO and gets its own request, response and notification pipes. The server queues sessions in a producer-consumer buffer served by a fixed pool of worker threads, with up to 8 clients at a time.
- **Notifications:** when a subscribed key is written or deleted, the server pushes the new value (or `DELETED`) to every subscriber's notification pipe.
- **Signals:** `SIGUSR1` disconnects every client and clears all subscriptions. `SIGINT` and `SIGTERM` shut the server down cleanly.

| Module | Role |
|---|---|
| `src/server` | hash table, job processing, backups, client sessions and subscriptions |
| `src/client` | command-line client and the client API (`kvs_connect`, `kvs_subscribe`, ...) |
| `src/common` | protocol opcodes and I/O helpers shared by both |

## Commands

Job files (`.job`) run against the store:

| Command | Example | Description |
|---|---|---|
| `WRITE` | `WRITE [(a,anna)(b,bernardo)]` | write one or more key-value pairs |
| `READ` | `READ [a,b]` | read one or more keys |
| `DELETE` | `DELETE [a]` | delete one or more keys |
| `SHOW` | `SHOW` | print every pair in the store |
| `WAIT` | `WAIT 1000` | wait the given number of milliseconds |
| `BACKUP` | `BACKUP` | save a snapshot of the store |

Clients read their commands from standard input:

| Command | Example | Description |
|---|---|---|
| `SUBSCRIBE` | `SUBSCRIBE [a]` | get notified when `a` changes |
| `UNSUBSCRIBE` | `UNSUBSCRIBE [a]` | stop getting notifications for `a` |
| `DELAY` | `DELAY 1000` | wait the given number of milliseconds |
| `DISCONNECT` | `DISCONNECT` | close the session |

## Running it

Requires Linux with `gcc` and `make`. It compiles on macOS, but macOS doesn't support unnamed POSIX semaphores (`sem_init`), so the server won't run correctly there.

```sh
cd src
make
./server/kvs <jobs_dir> <max_threads> <max_backups> <server_fifo_path>
./client/client <client_id> <server_fifo_path>
```

For example, start the server on the sample jobs and connect a client in another terminal:

```sh
./server/kvs server/jobs 2 2 /tmp/kvs.pipe
./client/client 1 /tmp/kvs.pipe < tests/client_tests.txt
```

`make test` runs scripted scenarios with one server and several clients.

## License

[MIT](LICENSE) © 2025 Inês Costa and Miguel Raposo
