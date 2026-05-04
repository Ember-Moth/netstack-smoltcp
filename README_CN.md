# Netstack Smoltcp

一个用于特殊场景的网络栈：把来自/发往 TUN 接口的 IP 包转换为 TCP 流和 UDP 数据报。底层使用 smoltcp-rs 作为网络栈实现。

[![Crates.io][crates-badge]][crates-url]
[![MIT licensed][mit-badge]][mit-url]
[![Apache licensed][apache-badge]][apache-url]
[![Build Status][actions-badge]][actions-url]

[crates-badge]: https://img.shields.io/crates/v/netstack-smoltcp.svg
[crates-url]: https://crates.io/crates/netstack-smoltcp
[mit-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[mit-url]: https://github.com/automesh-network/netstack-smoltcp/blob/master/LICENSE-MIT
[apache-badge]: https://img.shields.io/badge/license-APACHE2.0-blue.svg
[apache-url]: https://github.com/automesh-network/netstack-smoltcp/blob/master/LICENSE-APACHE
[actions-badge]: https://github.com/automesh-network/netstack-smoltcp/workflows/CI/badge.svg
[actions-url]: https://github.com/automesh-network/netstack-smoltcp/actions?query=workflow%3ACI+branch%3Amain

## 特性

- 支持 Future 的 Send 与非 Send 两种用法（多数用户使用 Send）。
- 支持由 TCP runner 驱动的 ICMP 协议，可用于 ICMP ping。
- 支持按源/目的 IP 地址过滤包。
- 可从 netstack 读取 IP 包，也可向 netstack 写入 IP 包。
- 可从 netstack 暴露的 TcpListener 接收 TcpStream。
- 可从 netstack 暴露的 UdpSocket 接收 UDP 数据报。
- 实现了常见的 Future 流式 trait 和异步 IO trait：
    * TcpListener 实现 futures 的 Stream/Sink trait
    * TcpStream 实现 tokio 的 AsyncRead/AsyncWrite trait
    * UdpSocket（ReadHalf/WriteHalf）实现 futures 的 Stream/Sink trait

## 平台

该 crate 为 Linux、iOS、macOS、Android 和 Windows 提供轻量级网络栈支持。
目前能在大多数目标上工作，但主要在以下常用平台上测试：
- linux-amd64: x86_64-unknown-linux-gnu
- android-arm64: aarch64-linux-android
- android-amd64: x86_64-linux-android
- macos-amd64: x86_64-apple-darwin
- macos-arm64: aarch64-apple-darwin
- ios-arm64: aarch64-apple-ios
- windows-amd64: x86_64-pc-windows-msvc
- windows-arm64: aarch64-pc-windows-msvc

## 示例

```rust
// let device = tun2::create_as_async(&cfg)?;
// let framed = device.into_framed();

let (stack, runner, udp_socket, tcp_listener) = netstack_smoltcp::StackBuilder::default()
    .stack_buffer_size(512)
    .tcp_buffer_size(4096)
    .enable_udp(true)
    .enable_tcp(true)
    .enable_icmp(true)
    .mtu(9000) // virtual device usually benefits from larger MTU
    .build()
    .unwrap();
let mut udp_socket = udp_socket.unwrap(); // udp enabled
let mut tcp_listener = tcp_listener.unwrap(); // tcp/icmp enabled
if let Some(runner) = runner {
    tokio::spawn(runner);
}

let (mut stack_sink, mut stack_stream) = stack.split();
let (mut tun_sink, mut tun_stream) = framed.split();

// Reads packet from stack and sends to TUN.
tokio::spawn(async move {
    while let Some(pkt) = stack_stream.next().await {
        if let Ok(pkt) = pkt {
            tun_sink.send(pkt).await.unwrap();
        }
    }
});

// Reads packet from TUN and sends to stack.
tokio::spawn(async move {
    while let Some(pkt) = tun_stream.next().await {
        if let Ok(pkt) = pkt {
            stack_sink.send(pkt).await.unwrap();
        }
    }
});

// Extracts TCP connections from stack and sends them to the dispatcher.
tokio::spawn(async move {
    handle_inbound_stream(tcp_listener).await;
});

// Receive and send UDP packets between netstack and NAT manager. The NAT
// manager would maintain UDP sessions and send them to the dispatcher.
tokio::spawn(async move {
    handle_inbound_datagram(udp_socket).await;
});
```

## 性能

通常 `netstack-smoltcp` 会配合 TUN 设备使用，因此选择合适的 TUN crate 很重要。

在 **Linux** 上，[tun-rs](https://github.com/tun-rs/tun-rs) 的性能优于 [rust-tun](https://github.com/meh/rust-tun/)，原因是 tun-rs 支持 GSO/GRO，可批量处理数据包。

执行 `bash scripts/bench-offload.sh` 可以验证 tun-rs 带来的 4x 性能提升。建议在你的 Linux 机器上试试。

使用 tun-rs 搭配 netstack-smoltcp 的示例见 [forward-offload-linux.rs](examples/forward-offload-linux.rs)。

更多调优内容可参考 tun-rs 的详细 [README](https://github.com/tun-rs/tun-rs/blob/main/README.md)。

## 许可证

本项目采用以下许可证之一：

 * Apache License, Version 2.0（[LICENSE-APACHE](LICENSE-APACHE) 或
   https://www.apache.org/licenses/LICENSE-2.0）
 * MIT license（[LICENSE-MIT](LICENSE-MIT) 或
   https://opensource.org/licenses/MIT）

你可自行选择其一。

### 贡献

除非你明确声明，否则你有意提交并被纳入 netstack-smoltcp 的任何贡献（按 Apache-2.0 许可证定义）均按上述双许可证授权，不附加任何额外条款或条件。

## 灵感来源

特别感谢以下优秀项目对 netstack-smoltcp 的启发（排名不分先后）：
- [shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust/)
- [netstack-lwip](https://github.com/eycorsican/netstack-lwip/)
- [rust-tun-active](https://github.com/tun2proxy/rust-tun)
- [rust-tun](https://github.com/meh/rust-tun/)
- [tun-rs](https://github.com/tun-rs/tun-rs)
- [smoltcp](https://github.com/smoltcp-rs/smoltcp)
