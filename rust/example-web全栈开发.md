# 主要内容

Web Service

服务端 Web App

客户端 Web App

Web 框架：Actix

数据库：PostgreSQL

数据库连接：SQLx



# 构建TCP Server

> 编写TCP Server和Client



## std::net模块

1. 标准库的 std::net 模块，提供网络基本功能
2. 支持TCP和UDP通信
3. TCPListener和TcpStream



### TcpListener

1. 引入TcpListener：

```rust
use std::net::TcpListener;
```



2. 创建TcpListener并绑定到指定端口

```rust
// 绑定到本地端口
let listener = TcpListener::bind("127.0.0.1:3000").unwrap();
```



3. 只接收一次请求(少用)

```rust
let accepter = listener.accept().unwrap();
```



4. 持续监听传入的连接(配合循环使用）：

```rust
for stream in listener.incoming() {
    let _stream = stream.unwrap(); // 传输以原始字节流
}
```



### TcpStream

1. 引入TcpStream

```rust
use std::net::TcpStream;
```



2. 创建一个连接到至指定地址的tcpstream

```rust
let _stream = TcpStream::connect("localhost:3000").unwrap();
```



### 读写消息

1. 使用TcpStream进行消息读写，其实现了 `std::io::Read/Write` 的trait，故需引入
2. 传输时需使用原始字节
3. 传入u8的切片

```rust
use std::io::{Read,Write};

let mut buffer = [0_u8; 1024];

stream.read(&mut buffer).unwrap();
stream.write(&buffer).unwrap();
```



# 构建HTTP Server

1. rust没有内置的http支持

## 消息流动

1. 客户端发出请求给server
2. server引用一个http库，发送给路由器
3. 路由器决定使用哪个handler
4. handler处理请求，并返回一个响应给客户端



## Web Server

### 组成

1. Sever：监听进来的TCP字节流
2. Router：接受HTTP请求，并决定调用哪个handler。
3. handler：处理http请求，构建http响应。
4. http library：
   1. 解释字节流，把它转化为http请求
   2. 把http响应转化会字节流



### 构建步骤

1. 解析http请求消息
2. 构建http响应消息
3. 路由与handler
4. 测试web server



## 解析http请求（消息）

1. 三个数据结构：

| 数据结构名称 | 数据类型 | 描述                 |
| ------------ | -------- | -------------------- |
| HttpRequest  | struct   | 表示Http请求         |
| Method       | enum     | 指定所允许的Http方法 |
| Version      | enum     | 指定所允许的Http版本 |



HttpRequest 需要实现的trait：

| Trait      | 描述                                      |
| ---------- | ----------------------------------------- |
| From<&str> | 用于把传进来的字符串切片转化为HttpRequest |
| Debug      | 打印调试消息                              |
| PartialEq  | 用于解析和自动化测试脚本里做比较          |



2. 在工作空间里：

```shell
cargo new httpserver
cargo new --lib http # 创建一个http库
```



3. http lib（用于解析http请求）：

```rust
// src/HttpRequest.rs
use std::{collections::HashMap, os::windows::process};

#[derive(Debug, PartialEq)]
pub enum Method {
    Get,
    Post,
    Uninitialized, // 用于初始化
}

// 实现trait, 而不是关联函数
impl From<&str> for Method  {
    fn from(s: &str) -> Method {
        match s {
            "GET" => Method::Get,
            "POST" => Method::Post,
            _ => Method::Uninitialized,
        }
    }
}

#[derive(Debug, PartialEq)]
pub enum Version {
    V1_1, // 目前只考虑
    V2_0,
    Uninitialized,
}

impl From<&str> for Version {
    fn from(s: &str) -> Self {
        match s {
            "HTTP/1.1" => Version::V1_1,
            _ => Version::Uninitialized,
        }
    }
}

#[derive(Debug, PartialEq)]
pub enum Resource {
    Path(String),
}

#[derive(Debug)]
pub struct HttpRequest {
    pub method: Method,
    pub version: Version,
    pub resource: Resource,
    pub headers: HashMap<String, String>,
    pub msg_body: String,
}

fn process_req_line(s: &str) -> (Method, Resource, Version) {
    let mut words = s.split_whitespace(); // 按空白分成多个单词
    let method = words.next().unwrap(); // next方法挨个遍历单词
    let resource = words.next().unwrap();
    let version = words.next().unwrap();
    return (method.into(), Resource::Path(resource.to_string()), version.into());
}

fn process_header_line(s: &str) -> (String, String) {
    let mut header_items = s.split(":"); // 按:分成多个单词
    let mut key = String::new();
    let mut value = String::new();

    if let Some(k) = header_items.next() {
        key = k.to_string();
    }
    if let Some(v) = header_items.next() {
        value = v.to_string();
    }
    return (key, value);
}

impl From<String> for HttpRequest {
    fn from(req: String) -> Self {
        // 设置初始值
        let mut parsed_method = Method::Uninitialized;
        let mut parsed_version = Version::V1_1;
        let mut parsed_resource = Resource::Path("".to_string());
        let mut parsed_headers = HashMap::new();
        let mut parsed_msg_body = "";

        for line in req.lines() {
            if line.contains("HTTP") {
                let (method, resource, version) = process_req_line(&line);
                parsed_method = method;
                parsed_resource = resource;
                parsed_version = version;
            }
            else if line.contains(":") {
                let (key, value) = process_header_line(&line);
                parsed_headers.insert(key, value);
            }
            else if line.len() == 0 {

            }
            else {
                parsed_msg_body = line;
            }
        }

        return HttpRequest {
            method: parsed_method,
            version: parsed_version,
            resource: parsed_resource,
            headers: parsed_headers,
            msg_body: parsed_msg_body.to_string(),
        };
    }
}

// 测试
#[cfg(test)]
mod tests {
    use std::hash::Hash;

    use super::*;

    #[test]
    fn test_method_into() {
        let m: Method = "GET".into(); // 由于Method实现了from，所以可以使用into语法将字符串转为Method
        assert_eq!(m, Method::Get);
    }

    #[test]
    fn test_version_into() {
        let m: Version = "HTTP/1.1".into();
        assert_eq!(m, Version::V1_1);
    }

    #[test]
    fn test_read_http() {
        let s: String = String::from("GET /greeting HTTP/1.1\r\nHost: localhost:3000\r\nUser-Agent: curl/7.71.1\r\nAccept: */*\r\n\r\n");
        let mut header_expected = HashMap::new();
        header_expected.insert("Host".into(), " localhost".into());
        header_expected.insert("Accept".into(), " */*".into());
        header_expected.insert("User-Agent".into(), " curl/7.71.1".into());

        let req: HttpRequest = s.into();

        assert_eq!(Method::Get, req.method);
        assert_eq!(Version::V1_1, req.version);
        assert_eq!(Resource::Path("/greeting".to_string()), req.resource);
        assert_eq!(header_expected, req.headers);
    }
}
```



## 构建HTTP响应

1. HttpResponse 需要实现的trait：

| 需要实现的方法或trait | 用途                            |
| --------------------- | ------------------------------- |
| Default trait         | 指定成员的默认值                |
| new()                 | 使用默认值创建一个新的结构体    |
| send_response()       | 构建响应，将原始字节通过tcp发送 |
| getter方法            | 获取成员的值                    |
| From trait            | 能够将HttpResponse转化为String  |



2. HttpResponse.rs

```rust
// src/httpresponse.rs

use std::collections::HashMap;
use std::io::{self, Write};

// derive: 让编译器为其实现哪些trait
// Debug: 打印调试消息
// PartialEq: 让成员可以进行比较
// Clone: 让其本身可以进行克隆
// 结构体如果含有成员引用类型, 就需要指明生命周期
#[derive(Debug, PartialEq, Clone)]
pub struct HttpResponse<'a> {
    version: &'a str,
    status_code: &'a str,
    status_text: &'a str,
    headers: Option<HashMap<&'a str, &'a str>>,
    body: Option<String>,
}

impl<'a> Default for HttpResponse<'a> {
    fn default() -> Self {
        return Self {
            version: "HTTP/1.1".into(),
            status_code: "200".into(),
            status_text: "OK".into(),
            headers: None,
            body: None,
        };
    }
}

impl<'a> From<HttpResponse<'a>> for String {
    fn from(res: HttpResponse) -> String {
        let res1 = res.clone();
        format!(
            "{} {} {}\r\n{}Content-Length: {}\r\n\r\n{}",
            &res1.version(),
            &res1.status_code(),
            &res1.statue_text(),
            &res1.headers(),
            &res.body.unwrap().len(),
            &res1.body(),
        )
    }
}

impl <'a> HttpResponse<'a> {
    pub fn new (
        status_code: &'a str,
        headers: Option<HashMap<&'a str, &'a str>>,
        body: Option<String>
    ) -> HttpResponse<'a> {
        let mut response: HttpResponse<'a> = HttpResponse::default();
        if status_code != "200" {
            response.status_code = status_code.into(); 
        }
        response.headers = match &headers {
            Some(_h) => headers,
            None => {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            }
        };
        response.status_text = match response.status_code {
            "200" => "OK".into(),
            "400" => "Bad Request".into(),
            "404" => "Not Found".into(),
            "500" => "Internal Server Error".into(),
            _ => "Not Found".into(),
        };

        response.body = body;
        return response;
    }

    pub fn send_response(&self, write_stream: &mut impl Write) -> Result<(), io::Error> {
        let res = self.clone();
        let response_string = String::from(res);
        let _ = write!(write_stream, "{}", response_string);
        return Ok(());
    }

    fn version(&self) -> &str {
        return self.version;
    }
    
    fn status_code(&self) -> &str {
        return self.status_code;
    }

    fn statue_text(&self) -> &str {
        return &self.status_text;
    }

    fn headers(&self) -> String {
        let map: HashMap<&str, &str> = self.headers.clone().unwrap();
        let mut headers_string: String = "".into();
        for (k, v) in map.iter() {
            headers_string = format!("{}{}:{}\r\n", headers_string, k, v);
        }
        return headers_string;
    }

    pub fn body(&self) -> &str {
        match &self.body {
            Some(b) => b.as_str(),
            None => "",
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_response_struct_creation_200() {
        let response_actual = HttpResponse::new(
            "200", 
            None, 
            Some("xxxx".into()),
        );
        let response_expected = HttpResponse {
            version: "HTTP/1.1",
            status_code: "200",
            status_text: "OK",
            headers: {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            },
            body: Some("xxxx".into()),
        };
        assert_eq!(response_actual, response_expected);
    }

    #[test]
    fn test_response_struct_creation_404() {
        let response_actual = HttpResponse::new(
            "404", 
            None, 
            Some("xxxx".into()),
        );
        let response_expected = HttpResponse {
            version: "HTTP/1.1",
            status_code: "404",
            status_text: "Not Found",
            headers: {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            },
            body: Some("xxxx".into()),
        };
        assert_eq!(response_actual, response_expected);
    }

    #[test]
    fn test_http_response_creation() {
        let response_expected = HttpResponse {
            version: "HTTP/1.1",
            status_code: "404",
            status_text: "Not Found",
            headers: {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            },
            body: Some("xxxx".into()),
        };
        let http_string: String = response_expected.into();
        let actual_string = "HTTP/1.1 404 Not Found\r\nContent-Type:text/html\r\nContent-Length: 4\r\n\r\nxxxx";
        assert_eq!(http_string, actual_string);
    }
}
```





# 工作空间

1. 创建工作空间s1并新建项目

```sh
cargo new s1

cd s1

# 创建两个项目
cargo new tcpserver
cargo new tcpclient
```



2. 修改s1的Cargo.toml文件

```toml
[workspace]

members = ["tcpserver", "tcpclient"]
```



3. 运行指定的包

```sh
cargo run -p tcpserver
```



















