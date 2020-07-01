+++
title = "How to add vcs information into binary on Android"
date = "2017-05-25T11:00:00"
tags = ["android"]
categories = ["work"]
+++

## background

这几天为了定位一个android问题 , 需要知道某个binary是在哪个git reversion ,
于是便决定将git reverion 编译进binary .

## show version

为了支持显示版本信息 , 这里修改binary支持的参数 , 添加 " -v " 参数 :

```
const volatile static char version[] __attribute__((used)) = VERSION;
int main(int argc, char **argv)
{
	...
	if argv contains "-v"
		print version
	...
}
```

这里的 `version` 通过编译时指定 :

```
git_reversion := $(shell git -C $(LOCAL_PATH) rev-parse HEAD)
git_workdirty := $(shell git -C $(LOCAL_PATH) diff --quiet HEAD && echo "clean")
LOCAL_CFLAGS += -DVERSION='"$(git_reversion)-$(git_workdirty)"'
```

## force rebuild

通过上述修改 , binary便可以显示git reversion , 但是 , 该方法依赖于该文件的更新 ,
如果该文件没有修改 , make便不会重新编译该文件 , 所以需要让android 编译系统每次都
编译该文件 :

```
LOCAL_MODULE_CLASS := EXECUTABLES
LOCAL_MODULE:= module_name
# always build fbvncserver.c for version update
$(local-intermediates-dir)/xxx.o: FORCE
```

这里需要在使用 `local-intermediates-dir` 之前定义 `LOCAL_MODULE_CLASS` 和
`LOCAL_MODULE` , 否则会报错 . 其中 `xxx` 为文件名 .

FIN.
