## Infiniband core ib_uverbs
__________________________________
### ib_uverbs_init函数
流程：
1. 注册字符设备 
register_chrdev_region()注册字符设备
alloc_chrdev_region()动态分配字符设备号
2. 创建sys fs class 目录
class_create()
class_create_file()
3. 调用device.c 注册ib_client
ib_register_client(&uverbs_client);
```c
static struct ib_client uverbs_client = {
	.name   = "uverbs",
	.no_kverbs_req = true,
	.add    = ib_uverbs_add_one,
	.remove = ib_uverbs_remove_one,
	.get_nl_info = ib_uverbs_get_nl_info,
};
```
IB驱动程序的高级用户可以使用ib_register_client（）注册用于IB设备添加和移除的回调。添加IB设备时，将调用每个注册客户的添加方法。
  （以注册客户端的顺序），以及设备何时删除后，将调用每个客户端的remove方法（以注册客户端的相反顺序）。另外，什么时候调用ib_register_client（）时，客户机将收到所有已注册设备的添加回调。
### ib_client
#### ib_client数据结构
```c
struct ib_client {
	const char *name;
	void (*add)   (struct ib_device *);
	void (*remove)(struct ib_device *, void *client_data);
	void (*rename)(struct ib_device *dev, void *client_data);
	int (*get_nl_info)(struct ib_device *ibdev, void *client_data,
			   struct ib_client_nl_info *res);
	int (*get_global_nl_info)(struct ib_client_nl_info *res);

	/* Returns the net_dev belonging to this ib_client and matching the
	 * given parameters.
	 * @dev:	 An RDMA device that the net_dev use for communication.
	 * @port:	 A physical port number on the RDMA device.
	 * @pkey:	 P_Key that the net_dev uses if applicable.
	 * @gid:	 A GID that the net_dev uses to communicate.
	 * @addr:	 An IP address the net_dev is configured with.
	 * @client_data: The device's client data set by ib_set_client_data().
	 *
	 * An ib_client that implements a net_dev on top of RDMA devices
	 * (such as IP over IB) should implement this callback, allowing the
	 * rdma_cm module to find the right net_dev for a given request.
	 *
	 * The caller is responsible for calling dev_put on the returned
	 * netdev. */
	struct net_device *(*get_net_dev_by_params)(
			struct ib_device *dev,
			u8 port,
			u16 pkey,
			const union ib_gid *gid,
			const struct sockaddr *addr,
			void *client_data);

	refcount_t uses;
	struct completion uses_zero;
	u32 client_id;

	/* kverbs are not required by the client */
	u8 no_kverbs_req:1;
};
/*
*返回属于该ib_client并与给定参数匹配的net_dev。
*@dev：net_dev用于通信的RDMA设备。
*@port：RDMA设备上的物理端口号。
*@pkey：net_dev使用的P_Key（如果适用）。
*@gid：net_dev用于通信的GID。
*@addr：配置net_dev的IP地址。
*@client_data：通过ib_set_client_data（）设置的设备的客户端数据。
*在RDMA设备之上实现net_dev的ib_client（例如IB上的IP）应实现此回调，从而允许rdma_cm模块为给定请求找到正确的net_dev。
*调用方负责在返回的netdev上调用dev_put。
*/
```
### ib_register_client函数

    


