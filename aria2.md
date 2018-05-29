### 概要
```
aria2c [<OPTIONS>] [<URI>|<MAGNET>|<TORRENT_FILE>|<METALINK_FILE>] ...
```
### 介绍
aria2 是一个下载工具，支持的协议包括 HTTP(S)、FTP、SFTP、BitTorrent、Metalink 。  
aria2 可以使用多源、多协议下载一个文件并尽量利用你最大的下载带宽，它能够同时使用 HTTP(S)/FTP /SFTP 和 BitTorrent 两种协议下载一个文件；从 HTTP(S)/FTP/SFTP 上下载完成的数据可以上传到 BitTorrent 群；通过使用 Metalink 块校验，aria2 在下载文件的时候能够自动验证数据块。
### 选项
### RPC接口
aria2 在 HTTP 上提供 json-rpc 和 xml-rpc 接口，功能基本相同。aria2 在 WebSocket 上也提供 JSON-RPC 接口，功能和 HTTP 上的 JSON-RPC 一样，只是多提供了服务器消息通知。  
JSON-RPC 接口( HTTP 和 WebSocket )的请求路径是`/jsonrpc`， xml-rpc 接口的请求地址是`/rpc`。  
WebSocket 地址是`ws://HOST:PORT/jsonrpc`，假如你启用 SSL/TLS 加密，地址变更为`wss://HOST:PORT/jsonrpc`。
JSON-RPC 基于[JSON-RPC 2.0](http://www.jsonrpc.org/specification)实现。并且支持 HTTP POST 和 HTTP GET(JSONP)，WebSocket 传输是 aria2 扩展。  
JSON-RPC 接口在 HTTP 上不能发送通知，只能在 Websocket 上发送，不支持浮点数，字符编码必须是 UTF-8。  
#### 术语
*GID*
>GID 是管理每一个下载的关键值，每个下载被分配一个唯一的 GID。GID 在 aria2 中被存储为64位二进制值，为了 RPC 访问，它被表现为16个字符的16进制字符串。查询的时候不需要全部字符，只要能被唯一识别就性。  
#### RPC 授权密钥
服务器启用验证，加参数：`--rpc-secret`。  
客户端使用例子：`aria2.addUri("token:$$secret$$", ["http://example.org/file"])`。  
system.multicall 方式例外，因为 XML-RPC 只能接受数组参数。  
#### 方法
*aria2.addUri([secret, ]uris[, options[, position]])*：  
这个方法是添加一个新的下载，uris 是一个 HTTP/FTP/SFTP/BitTorrent 地址的数组，数据的源必须是唯一的，否则出错。加磁性链接的时候只能是一个地址，不能像上面那样添加多地址。  
options：的成员是一个键值对对象。  
position：是一个从0开始的整数，它可以插入下载到指定队列位置。  
返回：GID。  
例子：  
```
{
'jsonrpc':'2.0', 'id':'gcf',
'method':'aria2.addUri',
'params':[['http://example.org/file'], {}, 0]
}
```
*aria2.addTorrent([secret, ]torrent[, uris[, options[, position]]])*：