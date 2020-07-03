---
categories:
- work
date: 2017-02-22T11:18:00
tags:
- android
title: Getaddrinfo bug on Android 64
---

## background

这几天在android x86_64上尝试运行一个游戏 , 但是程序无法启动 ,
从logcat中 , 游戏主线程抛出如下异常 :

```
I/Unity   ( 2451): LuaInterface.LuaException: No such host is known 
I/Unity   ( 2451): stack traceback: 
I/Unity   ( 2451):      [C]: in function 'ConnectUpdateServer' 
I/Unity   ( 2451):      [string "_Logic/UpdateManager.lua"]: in function 'StartUpdateVersion' 
I/Unity   ( 2451):      [string "Main.lua"]: in function <[string "Main.lua"]:0> 
I/Unity   ( 2451): System.Net.Dns:hostent_to_IPHostEntry(String, String[], String[]) 
I/Unity   ( 2451): System.Net.Dns:GetHostByName(String) 
I/Unity   ( 2451): System.Net.Dns:GetHostEntry(String) 
I/Unity   ( 2451): System.Net.Dns:GetHostAddresses(String) 
I/Unity   ( 2451): ConnectUpdate:ConnectUpdateServer(String, Int32, Int32, Int32) 
I/Unity   ( 2451): ConnectUpdateWrap:ConnectUpdateServer(IntPtr) 
I/Unity   ( 2451): LuaInterface.LuaDLL:lua_pcall(IntPtr, Int32, Int32, Int32) 
I/Unity   ( 2451): LuaInterface.LuaState:PCall(Int32, Int32) 
I/Unity   ( 2451): LuaInterface.LuaFunction:PCall() 
I/Unity   ( 2451): LuaInterface.LuaFunction:Call() 
I/Unity   ( 2451): LuaFramework.LuaManager:StartMain() 
I/Unity   ( 2451): LuaFramework.LuaManager:InitStart() 
I/Unity   ( 2451): LuaFramework.GameManager:StartInGame() 
I/Unity   ( 2451): LuaFramework.<OnExtractResource>c__Iterator0:MoveNext() 
I/Unity   ( 2451): UnityEngine.SetupCoroutine:InvokeMoveNext(IEnumerator, IntPtr) 
```

可以看出 , 问题出在dns解析失败 .

## getaddrinfo

由于android中使用的是自己的bionic libc ,
而dns解析最终是通过getaddrinfo实现 ,
所以首先从它开始分析 .

android dns解析整体是c/s 架构的 , 
当程序中调用getaddrinfo时 ,
都是通过将dns请求发送给netd服务 , 
由它统一处理 , 然后将结果返回给客户端 ,
而服务端和客户端之间是通过socket进行通信的 .

首先来看下服务端 , 也就是netd处理的相关代码 : 

```
struct addrinfo* result = NULL;
uint32_t rv = android_getaddrinfofornet(mHost, mService, mHints, mNetId, mMark, &result);
if (rv) {
	// getaddrinfo failed
	mClient->sendBinaryMsg(ResponseCode::DnsProxyOperationFailed, &rv, sizeof(rv));
} else {
	bool success = !mClient->sendCode(ResponseCode::DnsProxyQueryResult);
	struct addrinfo* ai = result;
	while (ai && success) {
		success = sendLenAndData(mClient, sizeof(struct addrinfo), ai)
			&& sendLenAndData(mClient, ai->ai_addrlen, ai->ai_addr)
			&& sendLenAndData(mClient,
							  ai->ai_canonname ? strlen(ai->ai_canonname) + 1 : 0,
							  ai->ai_canonname);
		ai = ai->ai_next;
	}
	success = success && sendLenAndData(mClient, 0, "");
	if (!success) {
		ALOGW("Error writing DNS result to client");
	}
}
if (result) {
	freeaddrinfo(result);
}
mClient->decRef();
```

可以看出 , 当解析成功时 , 首先返回一个状态码 (ResponseCode::DnsProxyQueryResult)
紧接着 , 将每个struct addrinfo返回给客户端 , 
由于struct addrinfo中包含其他字符串指针 , 
所以这里还需要将字符串通过值拷贝传给客户端 . 

客户端同样有相对应的接收流程 :

```
...
char buf[4];
// read result code for gethostbyaddr
if (fread(buf, 1, sizeof(buf), proxy) != sizeof(buf)) {
	goto exit;
}

int result_code = (int)strtol(buf, NULL, 10);
// verify the code itself
if (result_code != DnsProxyQueryResult ) {
	fread(buf, 1, sizeof(buf), proxy);
	goto exit;
}

struct addrinfo* ai = NULL;
struct addrinfo** nextres = res;
while (1) {
	uint32_t addrinfo_len;
	if (fread(&addrinfo_len, sizeof(addrinfo_len),
		  1, proxy) != 1) {
		break;
	}
	addrinfo_len = ntohl(addrinfo_len);
	if (addrinfo_len == 0) {
		success = 1;
		break;
	}

	if (addrinfo_len < sizeof(struct addrinfo)) {
		break;
	}
	struct addrinfo* ai = calloc(1, addrinfo_len +
				     sizeof(struct sockaddr_storage));
	if (ai == NULL) {
		break;
	}

	if (fread(ai, addrinfo_len, 1, proxy) != 1) {
		// Error; fall through.
		break;
	}

	// Zero out the pointer fields we copied which aren't
	// valid in this address space.
	ai->ai_addr = NULL;
	ai->ai_canonname = NULL;
	ai->ai_next = NULL;

	// struct sockaddr
	uint32_t addr_len;
	if (fread(&addr_len, sizeof(addr_len), 1, proxy) != 1) {
		break;
	}
	addr_len = ntohl(addr_len);
	if (addr_len != 0) {
		if (addr_len > sizeof(struct sockaddr_storage)) {
			// Bogus; too big.
			break;
		}
		struct sockaddr* addr = (struct sockaddr*)(ai + 1);
		if (fread(addr, addr_len, 1, proxy) != 1) {
			break;
		}
		ai->ai_addr = addr;
	}

	// cannonname
	uint32_t name_len;
	if (fread(&name_len, sizeof(name_len), 1, proxy) != 1) {
		break;
	}
	name_len = ntohl(name_len);
	if (name_len != 0) {
		ai->ai_canonname = (char*) malloc(name_len);
		if (fread(ai->ai_canonname, name_len, 1, proxy) != 1) {
			break;
		}
		if (ai->ai_canonname[name_len - 1] != '\0') {
			// The proxy should be returning this
			// NULL-terminated.
			break;
		}
	}

	*nextres = ai;
	nextres = &ai->ai_next;
	ai = NULL;
}
...
```

这里 , 所有的数据都是先传输一个4字节的长度 ,
再将具体的数据以字节流的方式传输 , 
当然 , 这里使用的都是网络字节序的方式 . 

这里其实是存在一个问题 , 
在传输struct addrinfo时 , 需要先传输其大小 ,
所以 , 如果客户端是32位 , 而服务端是64位时 , 
那么两端的大小就不同 . 
下面我们来看下在32位时， 它的大小 : 

```
(gdb) p sizeof(struct addrinfo)
$1 = 32
```

在64位的情况下 : 

```
(gdb) p sizeof(struct addrinfo)
$1 = 48
```

果然 , 所以该问题出现在客户端和服务端不匹配的情况下 . 

问题既然已经清楚了 , 解决思路就是避免传输 struct addrinfo的大小 . 
同时 ,
google官方已经提供一个[Fix](https://android-review.googlesource.com/#/q/Iee5ef802ebbf2e000b2593643de4eec46f296c04).

FIN.
