# 🛡️ S.P.E.A.R.

**Proof-of-concept VPN over QUIC with Noise_XX, TUN routing, FEC, and anti-replay**

[![Rust](https://img.shields.io/badge/built%20with-Rust-d33f10?style=for-the-badge&logo=rust)](https://www.rust-lang.org/)
[![QUIC](https://img.shields.io/badge/Transport-QUIC%20(Quinn)-blue?style=for-the-badge)](https://github.com/quinn-rs/quinn)
[![Noise Protocol](https://img.shields.io/badge/Crypto-Noise__XX-green?style=for-the-badge)](https://noiseprotocol.org/)
[![Platform](https://img.shields.io/badge/Platform-Windows-0078D6?style=for-the-badge&logo=windows)](https://www.microsoft.com/)

</div>

---

## 📖 Обзор проекта

**S.P.E.A.R.** — это экспериментальный PoC VPN-протокол, построенный на базе современного транспортного протокола QUIC и криптографического протокола Noise_XX. Проект состоит из двух ключевых компонентов:

* **`spear_vpn` (Клиент)** — поднимает локальный TUN-интерфейс, инкапсулирует IP-пакеты, шифрует их и управляет туннельным соединением.
* **`spear_server` (Сервер)** — принимает зашифрованные подключения, производит расшифровку, обрабатывает FEC/anti-replay и маршрутизирует трафик через TUN-интерфейс.

---

## 📑 Содержание

- [Архитектура и принцип работы](#-архитектура-и-принцип-работы)
- [Ключевые особенности](#-ключевые-особенности)
- [Быстрый старт](#-быстрый-старт)
- [Переменные окружения](#-переменные-окружения)
- [Продакшн и развертывание](#-продакшн-и-развертывание)

---

## ⚙️ Архитектура и принцип работы

### 1. Клиентская часть (`spear_vpn`)
1. Открывает QUIC endpoint локально на `0.0.0.0:0`.
2. Поднимает локальный TUN-интерфейс для Windows и читает из него исходящие IP-пакеты.
3. Собирает пакеты в микробатчи и добавляет padding для маскировки трафика.
4. Выполняет Noise_XX handshake и шифрует кадры через Noise transport.
5. Отправляет данные серверу через QUIC bi-directional stream.
6. Принимает ответы, проверяет anti-replay и FEC, а затем записывает пакеты обратно в TUN.

### 2. Серверная часть (`spear_server`)
1. Слушает UDP-порт через QUIC (с использованием крейта `quinn`).
2. Принимает соединение и открывает bi-directional QUIC поток для рукопожатия и данных.
3. Выполняет Noise_XX handshake поверх первого bi-stream.
4. После установления защищенного канала получает length-prefixed AEAD-защищенные пакеты.
5. Проверяет sequence, скользящую anti-replay bitmap и восстанавливает пропущенные пакеты через FEC.
6. Выводит расшифрованный трафик в локальный TUN-интерфейс сервера для маршрутизации.

---

## 🚀 Ключевые особенности

| Компонент | Описание |
| :--- | :--- |
| **QUIC Transport** | Высокопроизводительный асинхронный транспорт на базе библиотеки `quinn`. |
| **Noise Protocol** | Надежный Noise_XX handshake поверх bi-directional стрима. |
| **TUN Routing** | Интеграция на сетевом уровне IP для Windows. |
| **Security & Anti-Replay** | Защита от повторения пакетов (скользящая bitmap) + AEAD шифрование длины и полезной нагрузки. |
| :--- | :--- |
| **FEC (Forward Error Correction)** | Базовое восстановление одного пропущенного фрейма/пакета. |
| **Obfuscation** | Adaptive batching и padding на стороне клиента. |

---

## ⚡ Быстрый старт

### Сборка и запуск сервера (`spear_server`)
```bash
cd spear_server
cargo build --release
