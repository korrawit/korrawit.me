---
title: How to test client timeout in Go net/http
date: "2020-03-02T22:40:32.169Z"
template: "post"
draft: false
slug: "how-to-test-client-timeout-in-go-net-http"
category: "programming"
tags:
  - "go"
  - "programming"
description: "ทดสอบเคส client timeout ง่ายนิดเดียว"
---

การทำซอฟแวร์ในแต่ละครั้งนั้นคงหนีไม่พ้น การทำ scenario ที่มีทั้ง success case และ exception case
ซึ่ง exception case ก็มีต่างๆมากมายเช่น 
	
1. Client timeout โดยที่ service ปลายทาง ***ได้รับ*** request

![timeout](/media/timeout_1.png)

2. Client timeout โดยที่ service ปลายทาง ***ไม่ได้รับ*** request

![timeout](/media/timeout_2.png)

Client HTTP Timeout แบ่งออกตามรูป

![golang client timeout](https://blog.cloudflare.com/content/images/2016/06/Timeouts-002.png)
source: https://blog.cloudflare.com/content/images/2016/06/Timeouts-002.png

สิ่งที่ต้องทำคือ สร้าง http client ที่สามารถ config dial timeout และ response header timeout

1. กรณี Client timeout โดยที่ service ปลายทาง ***ได้รับ*** request
   - คือสามารถ client dial ได้แต่ server ไม่สามารถ response ได้
   - ต้อง config ให้ `ResponseHeaderTimeout` น้อยๆ
2. กรณี Client timeout โดยที่ service ปลายทาง ***ไม่ได้รับ*** request
   - คือสามารถ client ไม่สามารถ dial ไปได้
   - ต้อง config ให้ `DialTimeout` น้อยๆ

**ตัวอย่าง code custom http client**
```go
c := &http.Client{
	Transport: &http.Transport{
		Dial: (&net.Dialer{
			Timeout:   dialTimeout,
			KeepAlive: 30 * time.Second,
		}).Dial,
		ResponseHeaderTimeout: respHeaderTimeout,
	},
}
```
Source code: https://github.com/korrawit/go-timeout-experiment


ตัวอย่างการ run case1
```bash
# Case 1 Client
$ RESP_HEADER_TIMEOUT=1ns MODE=CUSTOM_CLIENT go run cmd/client/main.go 
2020/03/15 12:34:11 Running on: CUSTOM_CLIENT
2020/03/15 12:34:11 DIAL_TIMEOUT: 3s
2020/03/15 12:34:11 RESP_HEADER_TIMEOUT: 1ns
2020/03/15 12:34:11 Get http://localhost:8080/api/hello: net/http: timeout awaiting response headers
exit status 1

# Case 1 Server
2020/03/15 12:34:11 Server received request
```

ตัวอย่างการ run case2
```bash
# Case 2 Client
$ DIAL_TIMEOUT=1ns MODE=CUSTOM_CLIENT go run cmd/client/main.go 
2020/03/15 13:04:59 Running on: CUSTOM_CLIENT
2020/03/15 13:04:59 DIAL_TIMEOUT: 1ns
2020/03/15 13:04:59 RESP_HEADER_TIMEOUT: 3s
2020/03/15 13:04:59 Get http://localhost:8080/api/hello: dial tcp: i/o timeout
exit status 1

# Case 2 Server
```

