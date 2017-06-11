+++
date = "2013-03-29T20:57:00+08:00"
draft = false
title = "Asio and Net-SNMP"
topics = [ "Development" ]
tags = [ "Asio", "Net-SNMP" ]
+++

在监控系统里，我们会有获得网络设备的各种信息的功能需求，比如CPU，内存，各端口流量等，之前这些都是使用 snmpget/snmpwalk 等脚本去采集的，随着机器规模的增大，这块造成了不小的性能瓶颈，于是我们决定将这些脚本，转换成内置的函数来调用。有同事用 [Asio](http://www.boost.org/libs/asio/) 和 [Net-SNMP](http://www.net-snmp.org/) 库实现了一个非常原始的版本，一个请求过来就有一个线程去调用 snmp_sess_synch_response 这个阻塞的函数去获取数据，这相比之前而言，不过是用线程替换了原来的 fork 执行系统脚本，有点换汤不换药的感觉，而实际的测试也表明，性能的提升并不明显。对于这种实现方式，我是持很大反对意见的，因为在我看来，要实现成异步的调用才能显著的提升性能，在生产环境里，有时一个请求得几十秒才能返回，这种阻塞的方式是不可取的，讨论了很多次，但他坚持异步是不可能实现的，这是 [Net-SNMP](http://www.net-snmp.org/) 库的限制。我很无奈，如果真的没法做到，那么可以换别的库啊，何必在一棵树上吊死呢。后来同事去支援别的组去了，这个项目扔给了我，在改了些 Bug 后，我实在是不能忍受这种玩具般的实现了，于是决定按我设想的方式来大改。花了些时间查各种资料以及翻源码，通过 [Asio](http://www.boost.org/libs/asio/) 来托管 [Net-SNMP](http://www.net-snmp.org/) 的 socket 的方式，达到了我的预期目标。

<!--more-->

当然在实现的过程也不是一帆风顺的，也遇到了不少坑，所以分享下:

使用 [Asio](http://www.boost.org/libs/asio/) 来接管 [Net-SNMP](http://www.net-snmp.org/)

<pre>
<code class="language-cpp">
{
    ...

    snmp_ = snmp_sess_open(&sess);

    netsnmp_transport *transport  = snmp_sess_transport(snmp_);
    snmp_socket_.assign(ip::udp::v4(), transport->sock);
    snmp_socket_.non_blocking(true);

    snmp_socket_.async_receive(null_buffers(), 
        boost::bind(&SnmpConnection::HandleSnmpResponse,
                    shared_from_this(), placeholders::error)));

    ...
}

void SnmpConnection::HandleSnmpResponse(const boost::system::error_code &ec) 
{
    fd_set snmp_fds;
    FD_ZERO(&snmp_fds);
    FD_SET(snmp_socket_.native(), &snmp_fds);
    snmp_sess_read(snmp_, &snmp_fds);
}
</code>
</pre>

关闭的顺序

<pre>
<code class="language-cpp">
snmp_socket_.shutdown(ip::icmp::socket::shutdown_both);
snmp_socket_.close();

netsnmp_transport *transport  = snmp_sess_transport(snmp_);
if (NULL != transport) {
    transport->sock = -1;
    if (NULL != transport->base_transport) {
        transport->base_transport->sock = -1;
    }
}
</code>
</pre>

[Net-SNMP](http://www.net-snmp.org/) 库一个比较坑爹的地方就是，用于发送请求的 snmp_sess_async_send 函数在使用 v3 协议的时候，并不是异步的，因为它需要先发送一个 probe 包，然后调用 snmp_sess_synch_response 成功验证后，才会发送我们想要的请求。这就造成一个问题，如果网络设备挂掉了，或者网络不可达，那么就会堵在这直到超时。([#2310 snmp_async_send blocks during snmpv3 probe](http://sourceforge.net/p/net-snmp/bugs/2310/))

下面给的是使用 [USM](http://www.net-snmp.org/wiki/index.php/USM) 方式的解决方案:

<pre>
<code class="language-cpp">
// 先自行发送 probe 包
{
    snmp_pdu *pdu = NULL;
    usm_build_probe_pdu(&pdu)

    snmp_session *session = snmp_sess_session(snmp_);
    session->flags |= SNMP_FLAGS_DONT_PROBE;

    snmp_sess_async_send(snmp_, pdu, probe_cb, data);
}

// 收到成功的返回的处理 (对于超时的情况使用 Timer 来处理)
{
    snmp_session *session = snmp_sess_session(snmp_);

    if (session->securityEngineIDLen == 0) {
        return;
    }

    if (create_user_from_session(session) != SNMPERR_SUCCESS) {
        return;
    }
}
</code>
</pre>

由于使用 snmp_sess_async_send 后，对返回结果的处理是使用回调方式来进行的，这块可以参考 [Net-SNMP](http://www.net-snmp.org/) 库源码下的 perl/SNMP/SNMP.xs。

