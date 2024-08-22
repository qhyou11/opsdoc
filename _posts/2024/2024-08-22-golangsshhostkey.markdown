---
layout: post
title:  "golang ssh hostkey校验问题"
date:   2024-08-22 12:29:00 +0800
category: Dev
---

# 故障现象
前段时间，试用了一个名为wishlist的ssh客户端，这是一个ssh的目录服务，提供基于命令行界面的ssh会话管理。它可以自动加载~/.ssh/config中的配置，并提供基础的检索功能。
用了几次之后，发现有个ssh主机连接时，会报错：
```bash
 Wishlist

Something went wrong:

ssh: handshake failed: possible man-in-the-middle attack: knownhosts: key mismatch - if your host's key
changed, you might need to edit "/home/xxxx/.ssh/known_hosts"

Press any key to go back to the list.
```
但是通过系统自带的openssh客户端连接这个主机却没有任何问题。
```
ssh somesshserver
Last login: Xxx Xxx XX xx:xx:xx xxxx from 192.168.2.1

~ ⌚ xx:xx:xx
```
尝试删除known_hosts的对应记录，通过wishlist再连接就成功了。这时候，通过openssh的客户端连接依然是成功的。
这时候，再把known_hosts的对应主机记录删除，通过openssh客户端连接，会提示新hostkey,接受后，连接成功：
```bash
╰─➤  ssh somesshserver
The authenticity of host '[xxxxxxx]:xx ([xxx.xxx.xxx.xxx]:xxxx)' can't be established.
ED25519 key fingerprint is SHA256:I9+oq5xcfhQYgntxB+MJSor5JhxTuxxxxxxxxHVzThw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[n.003721.xyz]:7127' (ED25519) to the list of known hosts.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Last login: Xxx Xxx XX xx:xx:xx xxxx from 192.168.2.1

~ ⌚ xx:xx:xx
```
这时，再用wishlist连接该主机，故障重现，依旧提示有中间人攻击的可能。
# 故障分析
根据上述故障重现过程，可以大致分析出问题可能是wishlist不接受openssh客户端接受的那个key。进一步检查发现，openssh客户端接受的host key是 ssh-ed25519 类型的，而wishlist接受的是ecdsa-sha2-nistp256。
wishlist是golang客户端实现的，为了搞清楚背后的逻辑，可以走查golang下ssh host key校验的逻辑。

# 代码分析
## ssh连接示例
一个基本golang连接ssh示例代码如下：
```go
package main

import (
	"bytes"
	"fmt"
	"log"

	"golang.org/x/crypto/ssh"
)

func main() {
	var hostKey ssh.PublicKey
	// An SSH client is represented with a ClientConn.
	//
	// To authenticate with the remote server you must pass at least one
	// implementation of AuthMethod via the Auth field in ClientConfig,
	// and provide a HostKeyCallback.
	config := &ssh.ClientConfig{
		User: "username",
		Auth: []ssh.AuthMethod{
			ssh.Password("yourpassword"),
		},
		HostKeyCallback: ssh.FixedHostKey(hostKey),
	}
	client, err := ssh.Dial("tcp", "yourserver.com:22", config)
	if err != nil {
		log.Fatal("Failed to dial: ", err)
	}
	defer client.Close()

	// Each ClientConn can support multiple interactive sessions,
	// represented by a Session.
	session, err := client.NewSession()
	if err != nil {
		log.Fatal("Failed to create session: ", err)
	}
	defer session.Close()

	// Once a Session is created, you can execute a single command on
	// the remote side using the Run method.
	var b bytes.Buffer
	session.Stdout = &b
	if err := session.Run("/usr/bin/whoami"); err != nil {
		log.Fatal("Failed to run: " + err.Error())
	}
	fmt.Println(b.String())
}
```
## Dial函数
ClientConfig 中定义了hostkey的回调函数， 这个函数将在Dial里触发 (ssh/client.go)：
``` go
func Dial(network, addr string, config *ClientConfig) (*Client, error) {
    conn, err := net.DialTimeout(network, addr, config.Timeout)
    if err != nil {
        return nil, err
    }
    c, chans, reqs, err := NewClientConn(conn, addr, config)
    if err != nil {
        return nil, err
    }
    return NewClient(c, chans, reqs), nil
}
```
## NewClientConn 函数
Dial里的代码调用NewClientConn 函数(ssh/client.go)：
```go
func NewClientConn(c net.Conn, addr string, config *ClientConfig) (Conn, <-chan NewChannel, <-chan *Request, error) {
	fullConf := *config
	fullConf.SetDefaults()
	if fullConf.HostKeyCallback == nil {
		c.Close()
		return nil, nil, nil, errors.New("ssh: must specify HostKeyCallback")
	}

	conn := &connection{
		sshConn: sshConn{conn: c, user: fullConf.User},
	}

	if err := conn.clientHandshake(addr, &fullConf); err != nil {
		c.Close()
		return nil, nil, nil, fmt.Errorf("ssh: handshake failed: %w", err)
	}
	conn.mux = newMux(conn.transport)
	return conn, conn.mux.incomingChannels, conn.mux.incomingRequests, nil
}


```
## clientHandshake函数
NewClientConn里调用了clientHandshake函数 (ssh/client.go)：
```go
func (c *connection) clientHandshake(dialAddress string, config *ClientConfig) error {
	if config.ClientVersion != "" {
		c.clientVersion = []byte(config.ClientVersion)
	} else {
		c.clientVersion = []byte(packageVersion)
	}
	var err error
	c.serverVersion, err = exchangeVersions(c.sshConn.conn, c.clientVersion)
	if err != nil {
		return err
	}

	c.transport = newClientTransport(
		newTransport(c.sshConn.conn, config.Rand, true /* is client */),
		c.clientVersion, c.serverVersion, config, dialAddress, c.sshConn.RemoteAddr())
	if err := c.transport.waitSession(); err != nil {
		return err
	}

	c.sessionID = c.transport.getSessionID()
	return c.clientAuthenticate(config)
}
```
## newClientTransport函数
clientHandshake里调用newClientTransport函数(ssh/handshake.go)：
```go
func newClientTransport(conn keyingTransport, clientVersion, serverVersion []byte, config *ClientConfig, dialAddr string, addr net.Addr) *handshakeTransport {
	t := newHandshakeTransport(conn, &config.Config, clientVersion, serverVersion)
	t.dialAddress = dialAddr
	t.remoteAddr = addr
	t.hostKeyCallback = config.HostKeyCallback
	t.bannerCallback = config.BannerCallback
	if config.HostKeyAlgorithms != nil {
		t.hostKeyAlgorithms = config.HostKeyAlgorithms
	} else {
		t.hostKeyAlgorithms = supportedHostKeyAlgos
	}
	go t.readLoop()
	go t.kexLoop()
	return t
}
```
newClientTransport里需要注意的是，它设置了hostKeyAlgorithms，然后走到kexLoop.
### supportedHostKeyAlgos
假如在入口的ssh.ClientConfig中没有传入HostKeyAlgorithms，代码就会默认使用supportedHostKeyAlgos。supportedHostKeyAlgos(ssh/common.go)定义了host key相关算法的排序，后续会根据这个顺序从服务端提供的hostkey里匹配客户端支持的第一个算法类型。具体的算法类型顺序如下所示：
```go
var supportedHostKeyAlgos = []string{
	CertAlgoRSASHA256v01, CertAlgoRSASHA512v01,
	CertAlgoRSAv01, CertAlgoDSAv01, CertAlgoECDSA256v01,
	CertAlgoECDSA384v01, CertAlgoECDSA521v01, CertAlgoED25519v01,

	KeyAlgoECDSA256, KeyAlgoECDSA384, KeyAlgoECDSA521,
	KeyAlgoRSASHA256, KeyAlgoRSASHA512,
	KeyAlgoRSA, KeyAlgoDSA,

	KeyAlgoED25519,
}
```
## kexLoop函数
再回到newClientTransport里调用的kexLoop函数(ssh/handshake.go)：
```go
func (t *handshakeTransport) kexLoop() {

write:
	for t.getWriteError() == nil {
		var request *pendingKex
		var sent bool

		for request == nil || !sent {
			var ok bool
			select {
			case request, ok = <-t.startKex:
				if !ok {
					break write
				}
			case <-t.requestKex:
				break
			}

			if !sent {
				if err := t.sendKexInit(); err != nil {
					t.recordWriteError(err)
					break
				}
				sent = true
			}
		}

		if err := t.getWriteError(); err != nil {
			if request != nil {
				request.done <- err
			}
			break
		}

		// We're not servicing t.requestKex, but that is OK:
		// we never block on sending to t.requestKex.

		// We're not servicing t.startKex, but the remote end
		// has just sent us a kexInitMsg, so it can't send
		// another key change request, until we close the done
		// channel on the pendingKex request.

		err := t.enterKeyExchange(request.otherInit)

		t.mu.Lock()
		t.writeError = err
		t.sentInitPacket = nil
		t.sentInitMsg = nil

		t.resetWriteThresholds()

		// we have completed the key exchange. Since the
		// reader is still blocked, it is safe to clear out
		// the requestKex channel. This avoids the situation
		// where: 1) we consumed our own request for the
		// initial kex, and 2) the kex from the remote side
		// caused another send on the requestKex channel,
	clear:
		for {
			select {
			case <-t.requestKex:
				//
			default:
				break clear
			}
		}

		request.done <- t.writeError

		// kex finished. Push packets that we received while
		// the kex was in progress. Don't look at t.startKex
		// and don't increment writtenSinceKex: if we trigger
		// another kex while we are still busy with the last
		// one, things will become very confusing.
		for _, p := range t.pendingPackets {
			t.writeError = t.pushPacket(p)
			if t.writeError != nil {
				break
			}
		}
		t.pendingPackets = t.pendingPackets[:0]
		t.mu.Unlock()
	}

	// Unblock reader.
	t.conn.Close()

	// drain startKex channel. We don't service t.requestKex
	// because nobody does blocking sends there.
	for request := range t.startKex {
		request.done <- t.getWriteError()
	}

	// Mark that the loop is done so that Close can return.
	close(t.kexLoopDone)
}
```
kexLoop中有两个函数需要关注，一个是sendKexInit，另一个是enterKeyExchange。
## sendKexInit
sendKexInit(ssh/handshake.go)：
```go
func (t *handshakeTransport) sendKexInit() error {
	t.mu.Lock()
	defer t.mu.Unlock()
	if t.sentInitMsg != nil {
		// kexInits may be sent either in response to the other side,
		// or because our side wants to initiate a key change, so we
		// may have already sent a kexInit. In that case, don't send a
		// second kexInit.
		return nil
	}

	msg := &kexInitMsg{
		CiphersClientServer:     t.config.Ciphers,
		CiphersServerClient:     t.config.Ciphers,
		MACsClientServer:        t.config.MACs,
		MACsServerClient:        t.config.MACs,
		CompressionClientServer: supportedCompressions,
		CompressionServerClient: supportedCompressions,
	}
	io.ReadFull(rand.Reader, msg.Cookie[:])

	// We mutate the KexAlgos slice, in order to add the kex-strict extension algorithm,
	// and possibly to add the ext-info extension algorithm. Since the slice may be the
	// user owned KeyExchanges, we create our own slice in order to avoid using user
	// owned memory by mistake.
	msg.KexAlgos = make([]string, 0, len(t.config.KeyExchanges)+2) // room for kex-strict and ext-info
	msg.KexAlgos = append(msg.KexAlgos, t.config.KeyExchanges...)

	isServer := len(t.hostKeys) > 0
	if isServer {
		for _, k := range t.hostKeys {
			// If k is a MultiAlgorithmSigner, we restrict the signature
			// algorithms. If k is a AlgorithmSigner, presume it supports all
			// signature algorithms associated with the key format. If k is not
			// an AlgorithmSigner, we can only assume it only supports the
			// algorithms that matches the key format. (This means that Sign
			// can't pick a different default).
			keyFormat := k.PublicKey().Type()

			switch s := k.(type) {
			case MultiAlgorithmSigner:
				for _, algo := range algorithmsForKeyFormat(keyFormat) {
					if contains(s.Algorithms(), underlyingAlgo(algo)) {
						msg.ServerHostKeyAlgos = append(msg.ServerHostKeyAlgos, algo)
					}
				}
			case AlgorithmSigner:
				msg.ServerHostKeyAlgos = append(msg.ServerHostKeyAlgos, algorithmsForKeyFormat(keyFormat)...)
			default:
				msg.ServerHostKeyAlgos = append(msg.ServerHostKeyAlgos, keyFormat)
			}
		}

		if t.sessionID == nil {
			msg.KexAlgos = append(msg.KexAlgos, kexStrictServer)
		}
	} else {
		msg.ServerHostKeyAlgos = t.hostKeyAlgorithms

		// As a client we opt in to receiving SSH_MSG_EXT_INFO so we know what
		// algorithms the server supports for public key authentication. See RFC
		// 8308, Section 2.1.
		//
		// We also send the strict KEX mode extension algorithm, in order to opt
		// into the strict KEX mode.
		if firstKeyExchange := t.sessionID == nil; firstKeyExchange {
			msg.KexAlgos = append(msg.KexAlgos, "ext-info-c")
			msg.KexAlgos = append(msg.KexAlgos, kexStrictClient)
		}

	}

	packet := Marshal(msg)

	// writePacket destroys the contents, so save a copy.
	packetCopy := make([]byte, len(packet))
	copy(packetCopy, packet)

	if err := t.pushPacket(packetCopy); err != nil {
		return err
	}

	t.sentInitMsg = msg
	t.sentInitPacket = packet

	return nil
}

```
sendKexInit里将hostKeyAlgorithms赋给ServerHostKeyAlgos。
## enterKeyExchange
enterKeyExchange(ssh/handshake.go)
```go
func (t *handshakeTransport) enterKeyExchange(otherInitPacket []byte) error {
	if debugHandshake {
		log.Printf("%s entered key exchange", t.id())
	}

	otherInit := &kexInitMsg{}
	if err := Unmarshal(otherInitPacket, otherInit); err != nil {
		return err
	}

	magics := handshakeMagics{
		clientVersion: t.clientVersion,
		serverVersion: t.serverVersion,
		clientKexInit: otherInitPacket,
		serverKexInit: t.sentInitPacket,
	}

	clientInit := otherInit
	serverInit := t.sentInitMsg
	isClient := len(t.hostKeys) == 0
	if isClient {
		clientInit, serverInit = serverInit, clientInit

		magics.clientKexInit = t.sentInitPacket
		magics.serverKexInit = otherInitPacket
	}

	var err error
	t.algorithms, err = findAgreedAlgorithms(isClient, clientInit, serverInit)
	if err != nil {
		return err
	}

	if t.sessionID == nil && ((isClient && contains(serverInit.KexAlgos, kexStrictServer)) || (!isClient && contains(clientInit.KexAlgos, kexStrictClient))) {
		t.strictMode = true
		if err := t.conn.setStrictMode(); err != nil {
			return err
		}
	}

	// We don't send FirstKexFollows, but we handle receiving it.
	//
	// RFC 4253 section 7 defines the kex and the agreement method for
	// first_kex_packet_follows. It states that the guessed packet
	// should be ignored if the "kex algorithm and/or the host
	// key algorithm is guessed wrong (server and client have
	// different preferred algorithm), or if any of the other
	// algorithms cannot be agreed upon". The other algorithms have
	// already been checked above so the kex algorithm and host key
	// algorithm are checked here.
	if otherInit.FirstKexFollows && (clientInit.KexAlgos[0] != serverInit.KexAlgos[0] || clientInit.ServerHostKeyAlgos[0] != serverInit.ServerHostKeyAlgos[0]) {
		// other side sent a kex message for the wrong algorithm,
		// which we have to ignore.
		if _, err := t.conn.readPacket(); err != nil {
			return err
		}
	}

	kex, ok := kexAlgoMap[t.algorithms.kex]
	if !ok {
		return fmt.Errorf("ssh: unexpected key exchange algorithm %v", t.algorithms.kex)
	}

	var result *kexResult
	if len(t.hostKeys) > 0 {
		result, err = t.server(kex, &magics)
	} else {
		result, err = t.client(kex, &magics)
	}

	if err != nil {
		return err
	}

	firstKeyExchange := t.sessionID == nil
	if firstKeyExchange {
		t.sessionID = result.H
	}
	result.SessionID = t.sessionID

	if err := t.conn.prepareKeyChange(t.algorithms, result); err != nil {
		return err
	}
	if err = t.conn.writePacket([]byte{msgNewKeys}); err != nil {
		return err
	}

	// On the server side, after the first SSH_MSG_NEWKEYS, send a SSH_MSG_EXT_INFO
	// message with the server-sig-algs extension if the client supports it. See
	// RFC 8308, Sections 2.4 and 3.1, and [PROTOCOL], Section 1.9.
	if !isClient && firstKeyExchange && contains(clientInit.KexAlgos, "ext-info-c") {
		supportedPubKeyAuthAlgosList := strings.Join(t.publicKeyAuthAlgorithms, ",")
		extInfo := &extInfoMsg{
			NumExtensions: 2,
			Payload:       make([]byte, 0, 4+15+4+len(supportedPubKeyAuthAlgosList)+4+16+4+1),
		}
		extInfo.Payload = appendInt(extInfo.Payload, len("server-sig-algs"))
		extInfo.Payload = append(extInfo.Payload, "server-sig-algs"...)
		extInfo.Payload = appendInt(extInfo.Payload, len(supportedPubKeyAuthAlgosList))
		extInfo.Payload = append(extInfo.Payload, supportedPubKeyAuthAlgosList...)
		extInfo.Payload = appendInt(extInfo.Payload, len("ping@openssh.com"))
		extInfo.Payload = append(extInfo.Payload, "ping@openssh.com"...)
		extInfo.Payload = appendInt(extInfo.Payload, 1)
		extInfo.Payload = append(extInfo.Payload, "0"...)
		if err := t.conn.writePacket(Marshal(extInfo)); err != nil {
			return err
		}
	}

	if packet, err := t.conn.readPacket(); err != nil {
		return err
	} else if packet[0] != msgNewKeys {
		return unexpectedMessageError(msgNewKeys, packet[0])
	}

	if firstKeyExchange {
		// Indicates to the transport that the first key exchange is completed
		// after receiving SSH_MSG_NEWKEYS.
		t.conn.setInitialKEXDone()
	}

	return nil
}
```
enterKeyExchange里findAgreedAlgorithms找出服务端和客户端算法中相同的第一项，然后取出对应的key，在t.client(kex, &magics)调用hostKeyCallback做校验。
### findAgreedAlgorithms
findAgreedAlgorithms(ssh/common.go)
```go
func findAgreedAlgorithms(isClient bool, clientKexInit, serverKexInit *kexInitMsg) (algs *algorithms, err error) {
	result := &algorithms{}

	result.kex, err = findCommon("key exchange", clientKexInit.KexAlgos, serverKexInit.KexAlgos)
	if err != nil {
		return
	}

	result.hostKey, err = findCommon("host key", clientKexInit.ServerHostKeyAlgos, serverKexInit.ServerHostKeyAlgos)
	if err != nil {
		return
	}

	stoc, ctos := &result.w, &result.r
	if isClient {
		ctos, stoc = stoc, ctos
	}

	ctos.Cipher, err = findCommon("client to server cipher", clientKexInit.CiphersClientServer, serverKexInit.CiphersClientServer)
	if err != nil {
		return
	}

	stoc.Cipher, err = findCommon("server to client cipher", clientKexInit.CiphersServerClient, serverKexInit.CiphersServerClient)
	if err != nil {
		return
	}

	if !aeadCiphers[ctos.Cipher] {
		ctos.MAC, err = findCommon("client to server MAC", clientKexInit.MACsClientServer, serverKexInit.MACsClientServer)
		if err != nil {
			return
		}
	}

	if !aeadCiphers[stoc.Cipher] {
		stoc.MAC, err = findCommon("server to client MAC", clientKexInit.MACsServerClient, serverKexInit.MACsServerClient)
		if err != nil {
			return
		}
	}

	ctos.Compression, err = findCommon("client to server compression", clientKexInit.CompressionClientServer, serverKexInit.CompressionClientServer)
	if err != nil {
		return
	}

	stoc.Compression, err = findCommon("server to client compression", clientKexInit.CompressionServerClient, serverKexInit.CompressionServerClient)
	if err != nil {
		return
	}

	return result, nil
}

```
### findCommon
这里的findCommon(ssh/common.go)：
```go
func findCommon(what string, client []string, server []string) (common string, err error) {
	for _, c := range client {
		for _, s := range server {
			if c == s {
				return c, nil
			}
		}
	}
	return "", fmt.Errorf("ssh: no common algorithm for %s; client offered: %v, server offered: %v", what, client, server)
}
```
## client函数
enterKeyExchange里以`t.client(kex, &magics)`方式调用的client函数(ssh/handshake.go)：
```go
func (t *handshakeTransport) client(kex kexAlgorithm, magics *handshakeMagics) (*kexResult, error) {
	result, err := kex.Client(t.conn, t.config.Rand, magics)
	if err != nil {
		return nil, err
	}

	hostKey, err := ParsePublicKey(result.HostKey)
	if err != nil {
		return nil, err
	}

	if err := verifyHostKeySignature(hostKey, t.algorithms.hostKey, result); err != nil {
		return nil, err
	}

	err = t.hostKeyCallback(t.dialAddress, t.remoteAddr, hostKey)
	if err != nil {
		return nil, err
	}

	return result, nil
}
```
这里触发了hostKeyCallback回调函数的执行。
到这一步，go的ssh类库就已经实现客户端和服务端的hostkey校验算法协商，并选择了一个服务端的HostKey提交给回调函数hostKeyCallback进行校验。
## FixedHostKey
再来看看回调函数执行了什么逻辑。文章开头的那个例子里的回调是ssh.FixedHostKey（ssh/client.go）：
```go
type fixedHostKey struct {
key PublicKey
}

func (f *fixedHostKey) check(hostname string, remote net.Addr, key PublicKey) error {
if f.key == nil {
return fmt.Errorf("ssh: required host key was nil")
}

if !bytes.Equal(key.Marshal(), f.key.Marshal()) {
return fmt.Errorf("ssh: host key mismatch")
}
return nil

}

// FixedHostKey returns a function for use in

// ClientConfig.HostKeyCallback to accept only a specific host key.

func FixedHostKey(key PublicKey) HostKeyCallback {
hk := &fixedHostKey{key}
return hk.check
}
```
简单判断服务端的key是否和代码传入的key相匹配。这种案例和known_hosts没有关系。
## 通用hostkey回调
可以通过一个通用的host key回调函数来模拟wishlist的判断逻辑：
```go
func hostKeyCallback(e *Endpoint, path string) gossh.HostKeyCallback {
	return func(hostname string, remote net.Addr, key gossh.PublicKey) error {
		kh, err := os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0o600) //nolint:mnd
		if err != nil {
			return fmt.Errorf("failed to open known_hosts: %w", err)
		}
		defer func() { _ = kh.Close() }()

		callback, err := knownhosts.New(kh.Name())
		if err != nil {
			return fmt.Errorf("failed to check known_hosts: %w", err)
		}

		if err := callback(hostname, remote, key); err != nil {
			var kerr *knownhosts.KeyError
			if errors.As(err, &kerr) {
				if len(kerr.Want) > 0 {
					return fmt.Errorf("possible man-in-the-middle attack: %w - if your host's key changed, you might need to edit %q", err, kh.Name())
				}
				// if want is empty, it means the host was not in the known_hosts file, so lets add it there.
				fmt.Fprintln(kh, knownhosts.Line([]string{e.Address}, key)) //nolint: errcheck
				return nil
			}
			return fmt.Errorf("failed to check known_hosts: %w", err)
		}
		return nil
	}
}
```
这里用knownhosts的New进行判断：
```go
func New(files ...string) (ssh.HostKeyCallback, error) {
	db := newHostKeyDB()
	for _, fn := range files {
		f, err := os.Open(fn)
		if err != nil {
			return nil, err
		}
		defer f.Close()
		if err := db.Read(f, fn); err != nil {
			return nil, err
		}
	}

	var certChecker ssh.CertChecker
	certChecker.IsHostAuthority = db.IsHostAuthority
	certChecker.IsRevoked = db.IsRevoked
	certChecker.HostKeyFallback = db.check

	return certChecker.CheckHostKey, nil
}
```
CheckHostKey里对HostKey是fallback到db.check里做检查：
```go
// check checks a key against the host database. This should not be
// used for verifying certificates.
func (db *hostKeyDB) check(address string, remote net.Addr, remoteKey ssh.PublicKey) error {
	if revoked := db.revoked[string(remoteKey.Marshal())]; revoked != nil {
		return &RevokedError{Revoked: *revoked}
	}

	host, port, err := net.SplitHostPort(remote.String())
	if err != nil {
		return fmt.Errorf("knownhosts: SplitHostPort(%s): %v", remote, err)
	}

	hostToCheck := addr{host, port}
	if address != "" {
		// Give preference to the hostname if available.
		host, port, err := net.SplitHostPort(address)
		if err != nil {
			return fmt.Errorf("knownhosts: SplitHostPort(%s): %v", address, err)
		}

		hostToCheck = addr{host, port}
	}

	return db.checkAddr(hostToCheck, remoteKey)
}
```
调用checkAddr:
```go
// checkAddr checks if we can find the given public key for the
// given address.  If we only find an entry for the IP address,
// or only the hostname, then this still succeeds.
func (db *hostKeyDB) checkAddr(a addr, remoteKey ssh.PublicKey) error {
	// TODO(hanwen): are these the right semantics? What if there
	// is just a key for the IP address, but not for the
	// hostname?

	// Algorithm => key.
	knownKeys := map[string]KnownKey{}
	for _, l := range db.lines {
		if l.match(a) {
			typ := l.knownKey.Key.Type()
			if _, ok := knownKeys[typ]; !ok {
				knownKeys[typ] = l.knownKey
			}
		}
	}

	keyErr := &KeyError{}
	for _, v := range knownKeys {
		keyErr.Want = append(keyErr.Want, v)
	}

	// Unknown remote host.
	if len(knownKeys) == 0 {
		return keyErr
	}

	// If the remote host starts using a different, unknown key type, we
	// also interpret that as a mismatch.
	if known, ok := knownKeys[remoteKey.Type()]; !ok || !keyEq(known.Key, remoteKey) {
		return keyErr
	}

	return nil
}
```
如果known_hosts文件里没有对应服务端主机的hostkey，返回一个空的keyErr，也就是说之前没有针对这个主机确认过hostkey，这种情况下wishlist的逻辑是默认接受这个hostkey，并追加known_hosts文件。
如果known_hosts文件有这个主机的hostkey，会将涉及到这个主机的所有hostkey以算法类型为key构建一个字典，然后根据服务端提供的hostkey的算法类型在这个字典中查找对应的key，假如这个存在这个key，取出它，和服务端的hostkey比对，两者一致才能最终返回nil，表示匹配成功。
# 案例推演
用文章开头的故障例子，假设服务端只有两个hostkey，一个是ssh-ed25519，一个是ecdsa-sha2-nistp256。通过openssh客户端连接后，known_hosts文件里记录了ssh-ed25519的指纹。这时，再用wishlist去连接，由于wishlist以supportedHostKeyAlgos里定义的hostkey算法顺序从服务端匹配hostkey，它获取到的服务端hostkey是ecdsa-sha2-nistp256。而known_hosts文件根据这个主机生成的字典里只有一个ssh-ed25519类型的hostkey，自然和ecdsa-sha2-nistp256的key匹配不上，结果就是提示中间人攻击。

# 附录
## 算法类型常量
https://github.com/golang/crypto/blob/master/ssh/certs.go
const (
	CertAlgoRSAv01        = "ssh-rsa-cert-v01@openssh.com"
	CertAlgoDSAv01        = "ssh-dss-cert-v01@openssh.com"
	CertAlgoECDSA256v01   = "ecdsa-sha2-nistp256-cert-v01@openssh.com"
	CertAlgoECDSA384v01   = "ecdsa-sha2-nistp384-cert-v01@openssh.com"
	CertAlgoECDSA521v01   = "ecdsa-sha2-nistp521-cert-v01@openssh.com"
	CertAlgoSKECDSA256v01 = "sk-ecdsa-sha2-nistp256-cert-v01@openssh.com"
	CertAlgoED25519v01    = "ssh-ed25519-cert-v01@openssh.com"
	CertAlgoSKED25519v01  = "sk-ssh-ed25519-cert-v01@openssh.com"

	// CertAlgoRSASHA256v01 and CertAlgoRSASHA512v01 can't appear as a
	// Certificate.Type (or PublicKey.Type), but only in
	// ClientConfig.HostKeyAlgorithms.
	CertAlgoRSASHA256v01 = "rsa-sha2-256-cert-v01@openssh.com"
	CertAlgoRSASHA512v01 = "rsa-sha2-512-cert-v01@openssh.com"
)

CertAlgoECDSA256v01 对应 ecdsa-sha2-nistp256-cert-v01@openssh.com

var certKeyAlgoNames = map[string]string{
	CertAlgoRSAv01:        KeyAlgoRSA,
	CertAlgoRSASHA256v01:  KeyAlgoRSASHA256,
	CertAlgoRSASHA512v01:  KeyAlgoRSASHA512,
	CertAlgoDSAv01:        KeyAlgoDSA,
	CertAlgoECDSA256v01:   KeyAlgoECDSA256,
	CertAlgoECDSA384v01:   KeyAlgoECDSA384,
	CertAlgoECDSA521v01:   KeyAlgoECDSA521,
	CertAlgoSKECDSA256v01: KeyAlgoSKECDSA256,
	CertAlgoED25519v01:    KeyAlgoED25519,
	CertAlgoSKED25519v01:  KeyAlgoSKED25519,
}


KeyAlgoED25519 对应CertAlgoED25519v01，也就是 ssh-ed25519-cert-v01@openssh.com
