+++
title = "Linux kernel i2c subsystem"
date = "2018-01-24T09:00:00"
tags = ["linux", "kernel", "i2c"]
+++

## introduction

在linux kernel中i2c的设备驱动, 主要有下面几个软件抽象 :

- i2c_bus_type: 软件对i2c总线的抽象, 类型为 bus_type , kernel在初始化时会向设备驱动模型中注册该bus , 之后所有的i2c设备以及驱动都会被挂载在该总线下.
- i2c_driver: i2c驱动, 包括i2c控制器 , i2c主/从设备的驱动.
- i2c_adapter: i2c总线控制器.
- i2c_client: i2c从设备, 一般挂载在对应的总线控制器下面.

## i2c_bus_type

系统在初始化过程中 , 首先会注册i2c的bus:

```
static int __init i2c_init(void)
{
	...
	retval = bus_register(&i2c_bus_type);
	...
}
postcore_initcall(i2c_init);
```

首先我们来看下它的定义 :

```
struct bus_type i2c_bus_type = {
	.name		= "i2c",
	.match		= i2c_device_match,
	.probe		= i2c_device_probe,
	.remove		= i2c_device_remove,
	.shutdown	= i2c_device_shutdown,
};
```

由驱动模型可知 , 当总线上的驱动和设备是否匹配 , 会调用该总线的 `match` 的方法 :

```
static int i2c_device_match(struct device *dev, struct device_driver *drv)
{
	...

	/* Attempt an OF style match */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	driver = to_i2c_driver(drv);
	/* match on an id table if there is one */
	if (driver->id_table)
		return i2c_match_id(driver->id_table, client) != NULL;

	return 0;
}
```

可以看出 , 这里首先尝试通过dts进行匹配 , 如果匹配失败再尝试acpi方式 ,
最后才会进行 `id_table` 的方式.

如果匹配成功 , 下面就会进行probe :

```
static int i2c_device_probe(struct device *dev)
{
	...

	if (!device_can_wakeup(&client->dev))
		device_init_wakeup(&client->dev,
					client->flags & I2C_CLIENT_WAKE);
	dev_dbg(dev, "probe\n");

	status = of_clk_set_defaults(dev->of_node, false);
	if (status < 0)
		return status;

	status = dev_pm_domain_attach(&client->dev, true);
	if (status != -EPROBE_DEFER) {
		status = driver->probe(client, i2c_match_id(driver->id_table,
					client));
		if (status)
			dev_pm_domain_detach(&client->dev, true);
	}

	return status;
}
```

这里主要是上电对应的i2c设备, 如果设备成功上电 , 则调用driver的probe方法

而remove和shutdown方法则都是对driver中相关方法的封装.

## i2c_driver

在i2c核心层有一个简单的dummy driver, 该驱动不会对对应的设备做任何操作 :

```
static const struct i2c_device_id dummy_id[] = {
	{ "dummy", 0 },
	{ },
};

static int dummy_probe(struct i2c_client *client,
			   const struct i2c_device_id *id)
{
	return 0;
}

static int dummy_remove(struct i2c_client *client)
{
	return 0;
}

static struct i2c_driver dummy_driver = {
	.driver.name	= "dummy",
	.probe		= dummy_probe,
	.remove		= dummy_remove,
	.id_table	= dummy_id,
};
```

同时 , 这里还提供了helper function , 用户指定的i2c从设备地址 , 该设备会自动
匹配上述的dummy driver , 关于i2c从设备的创建后面会具体分析 .

```
struct i2c_client *i2c_new_dummy(struct i2c_adapter *adapter, u16 address)
{
	struct i2c_board_info info = {
		I2C_BOARD_INFO("dummy", address),
	};

	return i2c_new_device(adapter, &info);
}
EXPORT_SYMBOL_GPL(i2c_new_dummy);
```

## i2c_adapter

i2c_adapter是对i2c总线上控制器的抽象, 首先来看它的定义 :

```
struct i2c_adapter {
	...
	const struct i2c_algorithm *algo; /* the algorithm to access the bus */
	void *algo_data;

	...
	int timeout;			/* in jiffies */
	int retries;
	struct device dev;		/* the adapter device */

	int nr;
	...
};
```

这里省略了非关键字段, 

- algo, algo_data: 这里提供了i2c bus的操作函数，以及相关特性
- timeout, retries: 定义bus操作的超时时间和重试次数
- dev: i2c适配器同样也是i2c总线上的一个设备
- nr: 每个i2c适配器都有一个全局唯一的id , 其实你也可以把它认为i2c bus number , 只不过这里是软件层面上的

驱动程序一般会初始化好上述字段 , 之后便是将该adapter注册到kernel中 ,
主要的接口有 `i2c_add_numbered_adapter` 和 `i2c_add_adapter`
,两者的区别主要是是否使用设置的bus
number作为id还是让kernel动态生成一个未使用的id.

这两个接口最终都会调用 `i2c_register_adapter` ,
其中会向device driver model中注册该设备 :

```
dev_set_name(&adap->dev, "i2c-%d", adap->nr);
adap->dev.bus = &i2c_bus_type;
adap->dev.type = &i2c_adapter_type;
res = device_register(&adap->dev);
```

可以看出 , 这里设置的设备名中包含了i2c总线号 , 
所以之后我们可以sys文件系统看到看到诸如 `i2c-0` , `i2c-1` 这样的设备目录 , 
这些设备就是i2c adapter .

同时 , 该设备所属的bus为之前注册的 `i2c_bus_type` ,
而设备类型 , 则是 `i2c_adapter_type` :

```
static struct attribute *i2c_adapter_attrs[] = {
	&dev_attr_name.attr,
	&dev_attr_new_device.attr,
	&dev_attr_delete_device.attr,
	NULL
};
ATTRIBUTE_GROUPS(i2c_adapter);

struct device_type i2c_adapter_type = {
	.groups		= i2c_adapter_groups,
	.release	= i2c_adapter_dev_release,
};
```

该设备类型有3个属性: name, new_device, delete_device.
其中的new_device和delete_device
主要用于通过用户空间向kernel添加和删除对应总线下的i2c设备

## i2c_client

i2c_client对应于位于某个i2c总线下的一个i2c从设备 , 首先来看它的定义 :

```
struct i2c_client {
	unsigned short flags;		/* div., see below		*/
	unsigned short addr;		/* chip address - NOTE: 7bit	*/
	char name[I2C_NAME_SIZE];
	struct i2c_adapter *adapter;	/* the adapter we sit on	*/
	struct device dev;		/* the device structure		*/
	int irq;			/* irq issued by device		*/
	...
};
```

其中 , 

- flag: 用于表示该设备的一些特性 , 比如设备地址是10位还是7位 , 是否启动SCCB等
- addr: 该设备在总线上的地址
- name: 用户可以指定该i2c设备的名字
- adapter: 指向该设备所在总线的i2c控制器
- dev: 用于向device driver model中注册
- irq: 该设备产生中断的中断号

i2c从设备的添加 , 主要有以下三种方式 : 

- 驱动手动注册
- 在对应的总线下进行探测 , 从而进行动态注册
- 通过用户空间添加

### 驱动手动注册

这里主要通过 `i2c_new_device` 接口实现 :

```
struct i2c_client *
i2c_new_device(struct i2c_adapter *adap, struct i2c_board_info const *info)
```

可以看出这里的参数有两个 , 首先是对应的总线控制器 , 然后是 `i2c_board_info`
信息 : 

```
struct i2c_board_info {
	char		type[I2C_NAME_SIZE];
	unsigned short	flags;
	unsigned short	addr;
	void		*platform_data;
	struct dev_archdata	*archdata;
	struct device_node *of_node;
	struct fwnode_handle *fwnode;
	int		irq;
};
```

可以看出驱动程序可以指定对应设备的地址 , 中断号 , flag, type等等,
其中 , 这里的type其实就是设备的名字

在 `i2c_new_device` 函数中 , 首先会坚持该设备地址的合法性 ,
包括该地址是否正在被其他设备使用 , 之后便会向device driver model中注册该设备.

```
client->dev.parent = &client->adapter->dev;
client->dev.bus = &i2c_bus_type;
client->dev.type = &i2c_client_type;
client->dev.of_node = info->of_node;
client->dev.fwnode = info->fwnode;

i2c_dev_set_name(adap, client);
status = device_register(&client->dev);
```

可以看出该设备所属的总线为 `i2c_bus_type` , 而且他的parent是对应的总线控制器,
同时它也有属于其自己的device type `i2c_client_type` , 用户和控制器设备做区分.

这里 , 让我们来看下该设备的命名 :

```
dev_set_name(&client->dev, "%d-%04x", i2c_adapter_id(adap),
		 client->addr | ((client->flags & I2C_CLIENT_TEN)
				 ? 0xa000 : 0));
```

可以看出该设备的名称是 `总线号-设备地址`.

### 总线地址探测

这里驱动可以指定需要探测的地址列表, 其格式为:

```
static const unsigned short scan_addresses[] = {
	addr1, addr2, addr3,
	I2C_CLIENT_END
};
```

之后调用 `i2c_new_probed_device` :

```
struct i2c_client *
i2c_new_probed_device(struct i2c_adapter *adap,
			  struct i2c_board_info *info,
			  unsigned short const *addr_list,
			  int (*probe)(struct i2c_adapter *, unsigned short addr))
```

这样kernel会使用你提供的probe方法(会系统默认的探测方法 , 如果probe为NULL)
去探测指定的那些地址 , 如果发现其中一个地址上存在设备 ,
则使用该地址添加一个i2c从设备.

```
struct i2c_client *
i2c_new_probed_device(struct i2c_adapter *adap,
			  struct i2c_board_info *info,
			  unsigned short const *addr_list,
			  int (*probe)(struct i2c_adapter *, unsigned short addr))
{
	int i;

	if (!probe)
		probe = i2c_default_probe;

	for (i = 0; addr_list[i] != I2C_CLIENT_END; i++) {
		/* Check address validity */
		if (i2c_check_addr_validity(addr_list[i]) < 0) {
			dev_warn(&adap->dev, "Invalid 7-bit address "
				 "0x%02x\n", addr_list[i]);
			continue;
		}

		/* Check address availability */
		if (i2c_check_addr_busy(adap, addr_list[i])) {
			dev_dbg(&adap->dev, "Address 0x%02x already in "
				"use, not probing\n", addr_list[i]);
			continue;
		}

		/* Test address responsiveness */
		if (probe(adap, addr_list[i]))
			break;
	}

	if (addr_list[i] == I2C_CLIENT_END) {
		dev_dbg(&adap->dev, "Probing failed, no device found\n");
		return NULL;
	}

	info->addr = addr_list[i];
	return i2c_new_device(adap, info);
}
```

### 用户空间动态添加

用户空间可以通过向i2c总线控制器目录下的 `new_device`
文件写入信息来动态添加i2c设备,
写入的内容格式为: `设备名称` `设备地址\n` .

这样kernel就会根据对应的信息添加对应的从设备.

```
i2c_sysfs_new_device(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count)
{
	struct i2c_adapter *adap = to_i2c_adapter(dev);
	struct i2c_board_info info;
	struct i2c_client *client;
	char *blank, end;
	int res;

	memset(&info, 0, sizeof(struct i2c_board_info));

	blank = strchr(buf, ' ');
	...
	memcpy(info.type, buf, blank - buf);

	/* Parse remaining parameters, reject extra parameters */
	res = sscanf(++blank, "%hi%c", &info.addr, &end);
	...
	client = i2c_new_device(adap, &info);
	if (!client)
		return -EINVAL;

	/* Keep track of the added device */
	mutex_lock(&adap->userspace_clients_lock);
	list_add_tail(&client->detected, &adap->userspace_clients);
	mutex_unlock(&adap->userspace_clients_lock);
	dev_info(dev, "%s: Instantiated device %s at 0x%02hx\n", "new_device",
		 info.type, info.addr);

	return count;
}
static DEVICE_ATTR(new_device, S_IWUSR, NULL, i2c_sysfs_new_device);
```

可以看出 , 如果设备添加成功 , 对应的控制器下会有一个对应的链表用于管理这些设备.

## i2c transfer

i2c总线上的数据是以包为单位 , kernel使用 `i2c_msg` 定义每个包 :

```
struct i2c_msg {
	__u16 addr;	/* slave address			*/
	__u16 flags;
	__u16 len;		/* msg length				*/
	__u8 *buf;		/* pointer to msg data			*/
};
```

i2c系统主要有以下几个接口用于接受和发送数据包 :

- `__i2c_transfer`: 最低层接口 , 可用于任意个数的i2c数据包的读和写
- `i2c_transfer`: 对 `__i2c_transfer` 的封装 , 会使用adapter的lock进行串行化
- `i2c_master_send`: 用于i2c单个数据包的写操作 , 最终调用 `i2c_transfer` 进行传输
- `i2c_master_recv`: 用于i2c单个数据包的读操作 , 最终调用 `i2c_transfer` 进行传输

这里我们主要看一下 `__i2c_transfer` :

```
int __i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
{
	unsigned long orig_jiffies;
	int ret, try;

	...
	/* Retry automatically on arbitration loss */
	orig_jiffies = jiffies;
	for (ret = 0, try = 0; try <= adap->retries; try++) {
		ret = adap->algo->master_xfer(adap, msgs, num);
		if (ret != -EAGAIN)
			break;
		if (time_after(jiffies, orig_jiffies + adap->timeout))
			break;
	}
	...
	return ret;
}
```

这里会尝试控制器指定的尝试次数 , 同时也会考虑操作是否超时 , 
而最终的操作会调用控制器的 `master_xfer` 方法.

这里的超时时间 , 如果在控制器注册时驱动没有指定 , 默认时间是1s.

```
static int i2c_register_adapter(struct i2c_adapter *adap)
{
	...
	/* Set default timeout to 1 second if not already set */
	if (adap->timeout == 0)
		adap->timeout = HZ;
	...
}
```

当时 , 为了方便调试i2c数据的交互流程 , `__i2c_transfer` 中还提供了
event trace , 在实际调试过程中我们可以打开对应的event ,
其中包括 `i2c_read` , `i2c_write`, `i2c_reply`, `i2c_result`

下面是打开对应event trace开关看到的trace :

```
xxx-619   [000] ...1    47.566322: i2c_write: i2c-0 #0 a=051 f=0000 l=2 [7f-02]
xxx-619   [000] ...1    47.567551: i2c_result: i2c-0 n=1 ret=1
xxx-619   [000] ...1    47.577685: i2c_write: i2c-0 #0 a=051 f=0000 l=1 [7f]
xxx-619   [000] ...1    47.577691: i2c_read: i2c-0 #1 a=051 f=0001 l=1
xxx-619   [000] ...1    47.584811: i2c_reply: i2c-0 #1 a=051 f=0001 l=1 [02]
xxx-619   [000] ...1    47.584814: i2c_result: i2c-0 n=2 ret=2
```

可以看到这里会把i2c的总线号 , 从设备地址 , 数据包的flag和内容, 
以及操作相应的结果都dump出来 , 这对于调试是很方便的.

FIN
