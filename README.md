<div align="center">

# 🛡️ S.P.E.A.R.

**Proof‑of‑concept VPN over QUIC with Noise_XX, TUN routing, FEC and anti‑replay**

[![Rust](https://img.shields.io/badge/built%20with-Rust-d33f10?style=for-the-badge&logo=rust)](https://www.rust-lang.org/)
[![QUIC](https://img.shields.io/badge/Transport-QUIC%20(Quinn)-blue?style=for-the-badge)](https://github.com/quinn-rs/quinn)
[![Noise Protocol](https://img.shields.io/badge/Crypto-Noise__XX-green?style=for-the-badge)](https://noiseprotocol.org/)
[![Platform](https://img.shields.io/badge/Platform-Windows-0078D6?style=for-the-badge&logo=windows)]

</div>

---

## 🇷🇺 Русский — Обзор проекта

**S.P.E.A.R.** — экспериментальный Proof‑of‑Concept VPN‑протокол, реализованный поверх QUIC и использующий Noise_XX для аутентифицированного шифрования. Проект состоит из двух компонентов:

- `spear_vpn` — клиент, поднимающий локальный TUN‑интерфейс, инкапсулирующий и шифрующий IP‑пакеты.
- `spear_server` — сервер, принимающий защищённые подключения, обрабатывающий FEC/anti‑replay и маршрутизирующий трафик через TUN.

---

## 📑 Содержание / Contents

- [Архитектура и принцип работы](#-архитектура-и-принцип-работы)
- [Ключевые особенности](#-ключевые-особенности)
- [Быстрый старт](#-быстрый-старт)
- [Переменные окружения](#-переменные-окружения)
- [Продакшн и развертывание](#-продакшн-и-развертывание)
- [English version](#english-version)

---

## ⚙️ Архитектура и принцип работы

### Клиент (`spear_vpn`)
1. Создаёт локальный QUIC endpoint (по умолчанию `0.0.0.0:0`).
2. Поднимает локальный TUN‑интерфейс (Windows) и читает исходящие IP‑пакеты.
3. Группирует пакеты в микробатчи и добавляет padding для обфускации трафика.
4. Выполняет Noise_XX handshake и использует Noise transport для шифрования кадров.
5. Отправляет зашифрованные кадры серверу по bi‑directional QUIC stream.
6. Принимает ответы, проверяет anti‑replay/FEC и записывает расшифрованные пакеты в TUN.

### Сервер (`spear_server`)
1. Слушает UDP‑порт через QUIC (крейt `quinn`).
2. Принимает соединение и открывает bi‑directional поток для рукопожатия и данных.
3. Выполняет Noise_XX handshake поверх первого bi‑stream.
4. После установления канала принимает length‑prefixed AEAD‑защищённые пакеты.
5. Проверяет sequence, поддерживает скользящую anti‑replay bitmap и восстанавливает пропущенные пакеты через FEC.
6. Выводит расшифрованный трафик в локальный TUN‑интерфейс сервера.

---

## 🚀 Ключевые особенности

- QUIC Transport (crates: quinn) — быстрый асинхронный транспорт с низкой задержкой.
- Noise_XX — безопасный handshake и transport для шифрования.
- TUN routing — интеграция на сетевом уровне для Windows.
- Anti‑replay — скользящая bitmap для защиты от повторной отправки пакетов.
- FEC — простая схема восстановления пропущенных пакетов (восстановление до 1 пропуска в группе).
- Obfuscation — adaptive batching и padding для маскировки трафика.

---

## ⚡ Быстрый старт

Требования:
- Rust toolchain (rustc, cargo)
- Платформа: Windows для TUN‑интеграции (сервер можно запускать на Linux)

Сборка и запуск сервера:
```bash
cd spear_server
cargo build --release
# На Windows (PowerShell)
.\target\release\spear_server.exe
# На Linux (если есть сборка под Linux; пример)
./target/release/spear_server
```

Сборка и запуск клиента:
```bash
cd spear_vpn
cargo build --release
# На Windows (PowerShell)
.\target\release\spear_vpn.exe
```

Примечание: для разработки и тестов можно использовать self‑signed сертификаты, но в production обязательно используйте корректные TLS‑сертификаты.

---

## 🎛️ Переменные окружения

Сервер (`spear_server`):

| Переменная        | Описание                                                       | По умолчанию                 |
|-------------------|----------------------------------------------------------------|------------------------------|
| SERVER_CERT_PATH  | Путь до сертификата (PEM/DER)                                   | Авто‑генерация self‑signed   |
| SERVER_KEY_PATH   | Путь до приватного ключа (PEM/DER)                              | Авто‑генерация self‑signed   |
| SERVER_SEQ_PATH   | Файл для сохранения sequence номера отправки                    | server_send_seq.bin          |
| FEC_GROUP         | Размер группы FEC (1 = без FEC)                                 | 1                            |

Клиент (`spear_vpn`):

| Переменная        | Описание                                                       | По умолчанию |
|-------------------|----------------------------------------------------------------|--------------|
| HOME_GW           | Адрес домашнего шлюза для маршрутов Windows                    | 192.168.1.1  |
| CA_CERT_PATH      | Путь до PEM/DER файла CA для проверки сервера                  | —            |
| DEV_ALLOW_INSECURE| Разрешить пропуск проверки сертификата (1 = вкл)               | 0            |
| SEQ_PATH          | Файл для сохранения sequence counter                           | —            |
| FEC_GROUP         | Размер FEC‑группы                                              | —            |
| REKEY_INTERVAL_SEC| Время между rekey операциями (сек)                             | —            |
| REKEY_BYTES       | Объём трафика перед новым rekey                                | —            |

---

## 🛡️ Продакшн и развертывание

- Сертификаты: в production задавайте SERVER_CERT_PATH и SERVER_KEY_PATH с реальным TLS‑сертификатом (правильный CN/SAN). На клиенте указывайте CA_CERT_PATH.
- Порт: рекомендуем валидный публичный IP и порт UDP 443 для маскировки трафика.
- Файлы последовательностей (seq) нужны для устойчивости при перезапусках — храните их в надёжном месте.
- FEC: подбирайте параметр FEC_GROUP в зависимости от качества сети.

Пример systemd unit (Linux, пример):
```ini
[Unit]
Description=S.P.E.A.R. VPN Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/path/to/spear_server
ExecStart=/path/to/spear_server/target/release/spear_server
Restart=on-failure
Environment=SERVER_CERT_PATH=/path/to/cert.pem
Environment=SERVER_KEY_PATH=/path/to/key.pem

[Install]
WantedBy=multi-user.target
```

---

## 🇬🇧 English version

# S.P.E.A.R.

Proof‑of‑concept VPN over QUIC using Noise_XX, TUN routing, FEC, and anti‑replay.

## Project overview

S.P.E.A.R. is an experimental PoC VPN protocol built on top of the QUIC transport and using the Noise_XX protocol for authenticated encryption. The project contains two main components:

- `spear_vpn` — the client that brings up a local TUN interface, encapsulates and encrypts IP packets.
- `spear_server` — the server that accepts encrypted connections, handles FEC/anti‑replay, and routes traffic through a TUN interface.

## Architecture and workflow

Client (`spear_vpn`):
- Creates a local QUIC endpoint (default `0.0.0.0:0`).
- Brings up a local TUN interface on Windows and reads outbound IP packets.
- Batches packets and adds padding for traffic obfuscation.
- Performs a Noise_XX handshake and uses a Noise transport to encrypt frames.
- Sends encrypted frames to the server over a bi‑directional QUIC stream.
- Receives responses, validates anti‑replay/FEC, and writes decrypted packets back to the TUN.

Server (`spear_server`):
- Listens on a UDP port via QUIC (using the `quinn` crate).
- Accepts connections and opens a bi‑directional stream for handshake and data.
- Runs Noise_XX handshake over the initial bi‑stream.
- After the channel is established, receives length‑prefixed AEAD‑protected packets.
- Validates sequence numbers, maintains a sliding anti‑replay bitmap, and recovers missing packets via FEC.
- Emits decrypted traffic to the server's local TUN interface for routing.

## Quick start (English)

Requirements:
- Rust toolchain (rustc, cargo)
- Platform: Windows (for TUN integration). Server can run on Linux.

Build and run server:
```bash
cd spear_server
cargo build --release
# On Windows (PowerShell)
.\target\release\spear_server.exe
# On Linux
./target/release/spear_server
```

Build and run client:
```bash
cd spear_vpn
cargo build --release
# On Windows (PowerShell)
.\target\release\spear_vpn.exe
```

## Environment variables (English)

Server:
- SERVER_CERT_PATH — path to server certificate (PEM/DER). Default: self‑signed auto‑generate.
- SERVER_KEY_PATH — path to server private key (PEM/DER). Default: self‑signed auto‑generate.
- SERVER_SEQ_PATH — file to persist send sequence. Default: server_send_seq.bin
- FEC_GROUP — FEC group size (1 = no FEC). Default: 1

Client:
- HOME_GW — gateway address for Windows routes (default 192.168.1.1)
- CA_CERT_PATH — CA certificate (PEM/DER) to validate server
- DEV_ALLOW_INSECURE — allow skipping certificate verification (1 = enabled)
- SEQ_PATH — file to persist sequence counter
- FEC_GROUP — FEC group size
- REKEY_INTERVAL_SEC — time between rekeys (seconds)
- REKEY_BYTES — bytes threshold for rekey

## Production notes (English)
- Use real TLS certificates with correct CN/SAN in production.
- Deploy the server on a public IP; consider UDP 443 to blend with normal HTTPS traffic.
- Persist sequence files to avoid session inconsistency after restarts.

---

## ⚖️ License

Add a LICENSE file (e.g. MIT or Apache‑2.0) to this repository.

---

## 🏷️ Topics / Tags

rust · quic · noise‑protocol · vpn · tun · windows · security · async
