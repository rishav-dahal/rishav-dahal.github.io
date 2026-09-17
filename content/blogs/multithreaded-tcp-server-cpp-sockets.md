---
title: "Writing a High-Throughput Multithreaded TCP Server in C++: Lessons in Concurrency, Sockets & SIGPIPE"
date: 2026-07-28T10:00:00+05:45
slug: multithreaded-tcp-server-cpp-sockets
categories:
  - Systems
  - Networking
tags:
  - C++
  - Networking
  - Sockets
  - TCP/IP
  - Multithreading
  - Concurrency
  - Distributed Systems
summary: "A practical systems programming guide to writing a high-throughput multithreaded TCP server in modern C++ with raw sockets, thread pools, TCP stream defragmentation, and crash prevention."
description: "Learn how to build a robust multithreaded TCP server in modern C++ with thread pools, condition variables, raw POSIX/WinSock sockets, and proper packet framing."
author: "Rishav Dahal"
keywords: ["C++ Sockets", "Multithreaded TCP Server", "Thread Pool C++20", "POSIX Socket Programming", "TCP Packet Framing", "SIGPIPE Crash Fix"]
cover:
  image: "https://cdn.rishavdahal.com.np/cpp-tcp-socket-server.jpg"
  alt: "Futuristic technical illustration of multithreaded C++ socket programming with worker threads and packet streams"
  caption: "Low-level socket programming and thread pool concurrency architecture in modern C++"
  relative: false
showtoc: true
draft: false
---

> 📦 **Open Source Repository**: The complete codebase, architecture schemas, and implementation files for this system are available on GitHub at [**rishav-dahal/LAN-TODO**](https://github.com/rishav-dahal/LAN-TODO).

Most web developers spend their careers comfortably cushioned by high-level abstractions like Express, Django, or FastAPI. You call `app.get()`, and the framework magically handles HTTP handshakes, keep-alive headers, and concurrent threads.

Then you decide to build something low-level—a custom telemetry ingestor, a multiplayer game server, or an order routing engine—where a 20ms HTTP overhead is unacceptable. You open up C++, invoke `socket()`, and immediately enter a world where:
- TCP doesn't give you "messages"—it gives you an endless stream of fragmented bytes.
- Spawning a thread per connection works fine for 10 users and crashes your machine at 1,000.
- A disconnected client can send a `SIGPIPE` signal that instantly kills your entire process.
- Windows (`WinSock`) and Linux (POSIX) have just enough subtle differences to ruin your afternoon.

Here is the exact architectural blueprint, thread pool implementation, and packet framing protocol I use to build robust, low-latency TCP servers in modern C++ (C++17/20).

---

## 1. The Concurrency Model: Why Thread Pools Win

The textbook way to write a multithreaded server is the "thread-per-client" model:
```cpp
// THE NAIVE APPROACH: DO NOT DO THIS IN PRODUCTION!
while (true) {
    int client_fd = accept(server_fd, ...);
    std::thread t(handle_client, client_fd);
    t.detach(); // Leaks threads, hits OS pthread limits, explodes context switching!
}
```

If 1,500 clients connect simultaneously:
1. The kernel spends more CPU cycles swapping CPU registers and stack pointers between 1,500 threads than actually processing socket data.
2. Each thread defaults to an 8MB stack allocation on Linux, consuming 12 GB of virtual RAM just for idle threads.

### The Production Solution: Fixed Worker Thread Pool
Instead of creating threads on the fly, we spin up a fixed pool of worker threads matched to the machine's hardware concurrency (`std::thread::hardware_concurrency()`). The main thread's only job is to call `accept()` and push client socket file descriptors into a thread-safe task queue.

```
                  ┌──────────────────────────────────────────────┐
                  │              Acceptor Thread                 │
                  │  (Calls accept() and pushes to Task Queue)   │
                  └──────────────────────┬───────────────────────┘
                                         │ Enqueues Socket FD
                                         ▼
                  ┌──────────────────────────────────────────────┐
                  │           Thread-Safe Task Queue             │
                  │         (Mutex + Condition Variable)         │
                  └──────────────────────┬───────────────────────┘
                                         │
                 ┌───────────────────────┼───────────────────────┐
                 ▼                       ▼                       ▼
          [Worker Thread 1]       [Worker Thread 2]       [Worker Thread N]
          (Decodes Frames)        (Decodes Frames)        (Decodes Frames)
```

---

## 2. Implementing a Robust Thread-Safe Queue in C++

Here is the thread pool task queue implementation using `std::mutex` and `std::condition_variable`:

```cpp
// ThreadPool.hpp
#pragma once
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <atomic>

class ThreadPool {
public:
    explicit ThreadPool(size_t num_threads) : stop_(false) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this]() {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex_);
                        condition_.wait(lock, [this]() {
                            return stop_ || !tasks_.empty();
                        });

                        if (stop_ && tasks_.empty()) {
                            return; // Thread gracefully exits
                        }

                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    // Execute task outside the lock!
                    task();
                }
            });
        }
    }

    template<class F>
    void enqueue(F&& f) {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            tasks_.emplace(std::forward<F>(f));
        }
        condition_.notify_one();
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            stop_ = true;
        }
        condition_.notify_all();
        for (std::thread& worker : workers_) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }

private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex queue_mutex_;
    std::condition_variable condition_;
    std::atomic<bool> stop_;
};
```

---

## 3. The Core Socket Engine

Here is the server setup using POSIX sockets (with `SO_REUSEADDR` to prevent the dreaded `Address already in use` error when restarting during development):

```cpp
// Server.cpp
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include "ThreadPool.hpp"

constexpr int PORT = 8080;
constexpr int BUFFER_SIZE = 4096;

void handle_client(int client_fd) {
    char buffer[BUFFER_SIZE];
    
    while (true) {
        ssize_t bytes_read = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
        
        if (bytes_read == 0) {
            // Client cleanly closed connection (FIN packet)
            std::cout << "[INFO] Client " << client_fd << " disconnected cleanly.\n";
            break;
        } else if (bytes_read < 0) {
            // Read error or timeout
            std::cerr << "[ERROR] recv failed on socket " << client_fd << "\n";
            break;
        }

        buffer[bytes_read] = '\0';
        std::cout << "[RECV " << client_fd << "]: " << buffer << "\n";

        // Echo back response
        const char* response = "ACK\n";
        send(client_fd, response, strlen(response), 0);
    }

    close(client_fd);
}

int main() {
    // CRITICAL: Prevent terminated client connections from killing the server!
    signal(SIGPIPE, SIG_IGN);

    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) {
        perror("Socket creation failed");
        return 1;
    }

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("Bind failed");
        close(server_fd);
        return 1;
    }

    if (listen(server_fd, SOMAXCONN) < 0) {
        perror("Listen failed");
        close(server_fd);
        return 1;
    }

    std::cout << "[SERVER] Listening on port " << PORT << "...\n";

    ThreadPool pool(std::thread::hardware_concurrency());

    while (true) {
        sockaddr_in client_addr{};
        socklen_t client_len = sizeof(client_addr);
        
        int client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_len);
        if (client_fd < 0) {
            perror("Accept failed");
            continue;
        }

        char client_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, INET_ADDRSTRLEN);
        std::cout << "[ACCEPTED] Connection from " << client_ip << " on fd " << client_fd << "\n";

        // Offload to worker pool
        pool.enqueue([client_fd]() {
            handle_client(client_fd);
        });
    }

    close(server_fd);
    return 0;
}
```

---

## 4. The Two Silent Killers in TCP Server Engineering

### Killer #1: The `SIGPIPE` Crash
On Unix systems, if a client abruptly disconnects (e.g. user pulls the ethernet cable or force-quits their app) and your server attempts to call `send()` on that socket, the kernel raises a `SIGPIPE` signal.

By default in Linux, the handler for `SIGPIPE` is **terminate process**! Your entire multithreaded server instantly evaporates without even logging an error.

**The Fix**:
Always ignore `SIGPIPE` in `main()`:
```cpp
signal(SIGPIPE, SIG_IGN);
```
Or pass the `MSG_NOSIGNAL` flag during `send()`:
```cpp
send(client_fd, buffer, len, MSG_NOSIGNAL);
```

### Killer #2: Assuming `recv()` Returns One "Message"
Beginners assume that if the client calls `send("Hello")` and then `send("World")`, the server will receive `"Hello"` in the first `recv()` and `"World"` in the second.

**In TCP, this is not guaranteed.** TCP is a continuous byte stream. The kernel may bundle both into a single 10-byte buffer (`"HelloWorld"`), or split `"Hello"` into two fragments (`"Hel"` and `"lo"`).

**The Fix: Length-Prefixed Framing Protocol**:
Before sending payload bytes, prefix every message with a fixed 4-byte big-endian integer indicating message length:
```
[4 Bytes: Message Length N] [N Bytes: Raw Data Payload]
```
The receiving worker loop reads exactly 4 bytes first, extracts `N`, and then loops `recv()` until exactly `N` payload bytes have been reassembled into a complete packet.

---

## Summary

Moving from high-level web frameworks to low-level C++ sockets reveals how the internet actually operates under the hood. By utilizing a fixed worker thread pool, ignoring `SIGPIPE`, framing streams with length prefixes, and tuning socket reuse flags, you can build a server capable of pushing hundreds of thousands of low-latency packets per second with minimal CPU and memory overhead.


---

## 🛠️ GitHub Repository & Next Steps

The complete open-source source code and architecture discussed in this guide are publicly available:

- **Project Repository**: [Multithreaded C++ Socket Server (LAN-TODO) on GitHub](https://github.com/rishav-dahal/LAN-TODO)
- **Developer Profile**: [@rishav-dahal](https://github.com/rishav-dahal)

If you're building a similar system or encounter edge cases in your deployment, feel free to star the repo, file an issue, or submit an optimization pull request!
