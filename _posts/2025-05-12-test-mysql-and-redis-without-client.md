---
title: "🛠️ Quick Guide: Test MySQL and Redis Without Native Clients"
date: 2025-05-12
description: "Test MySQL and Redis database connections from the command line using telnet, netcat, or curl—no native clients required."
categories:
  - database
tags:
  - mysql
  - redis
  - networking
  - command-line
  - troubleshooting
---

Whether you're working in a containerized environment, on a minimal OS, or just don’t have native CLI tools like `mysql` or `redis-cli` installed, it’s still possible to test database connectivity using standard networking tools. This guide outlines alternative ways to test **MySQL** (port `3306`) and **Redis** (port `6379`) connections.

---

## 🔌 Method 1: Using `telnet`

If you know the IP address and port of the database server, `telnet` can help confirm basic TCP connectivity.

```bash
telnet [server_ip] [port]
```

### MySQL Example:
```bash
telnet 127.0.0.1 3306
```

### Redis Example:
```bash
telnet 127.0.0.1 6379
```

- **Success:** You’ll see some protocol-specific output (random characters or version info).
- **Failure:** You’ll get a connection refused or timeout error.

---

## 📡 Method 2: Using `netcat` (`nc`)

`netcat` is often a more modern and script-friendly alternative to `telnet`.

```bash
nc -zv [server_ip] [port]
```

### MySQL Example:
```bash
nc -zv 127.0.0.1 3306
```

### Redis Example:
```bash
nc -zv 127.0.0.1 6379
```

- `-z`: Scans without sending data.
- `-v`: Enables verbose output.

---

## 🌐 Method 3: Using `curl`

While `curl` isn’t a database client, it can test TCP reachability using the `telnet://` protocol.

```bash
curl -v telnet://[server_ip]:[port]
```

### MySQL Example:
```bash
curl -v telnet://127.0.0.1:3306
```

### Redis Example:
```bash
curl -v telnet://127.0.0.1:6379
```

- If the port is open, the TCP handshake will succeed and you’ll see connection messages in the output.
- If not, curl will report a failure to connect.

---

## 🧪 Bonus: Quick Redis Test Using Raw Command

If you manage to connect using `telnet` or `nc`, you can even try a simple Redis command like `PING`:

```bash
echo -e "PING\r\n" | nc 127.0.0.1 6379
```

- **Expected Output:** `+PONG`

This works because Redis uses a plain-text protocol that responds directly to commands.

---

## ✅ Conclusion

Even without the native clients (`mysql`, `redis-cli`), you can quickly verify database connectivity with general-purpose network tools. This is especially useful in minimal environments, containers, or during initial troubleshooting.

- Use **`telnet`** or **`nc`** for quick TCP checks.
- Use **`curl`** for basic port testing.
- For Redis, try sending a `PING` manually to get a live response.
