# msg分析

堆喷要用到，貌似没人写这块的分析，那我来

## 首先通信全要用mach_msg函数

```
mach_msg_return_t
mach_msg(msg, option, send_size, rcv_size, rcv_name, timeout, notify)
	mach_msg_header_t *msg;
	mach_msg_option_t option;
	mach_msg_size_t send_size;
	mach_msg_size_t rcv_size;
	mach_port_t rcv_name;
	mach_msg_timeout_t timeout;
	mach_port_t notify;
{
	mach_msg_return_t mr;

    /*
     * Consider the following cases:
     *	1) Errors in pseudo-receive (eg, MACH_SEND_INTERRUPTED
     *	plus special bits).
     *	2) Use of MACH_SEND_INTERRUPT/MACH_RCV_INTERRUPT options.
     *	3) RPC calls with interruptions in one/both halves.
     *
     * We refrain from passing the option bits that we implement
     * to the kernel.  This prevents their presence from inhibiting
     * the kernel's fast paths (when it checks the option value).
     */
    
    mr = mach_msg_overwrite_trap(msg, option &~ LIBMACH_OPTIONS,
    		   send_size, rcv_size, rcv_name,
    		   timeout, notify, MACH_MSG_NULL, 0);
    if (mr == MACH_MSG_SUCCESS)
    	return MACH_MSG_SUCCESS;
    
    if ((option & MACH_SEND_INTERRUPT) == 0)
    	while (mr == MACH_SEND_INTERRUPTED)
    		mr = mach_msg_overwrite_trap(msg,
    			option &~ LIBMACH_OPTIONS,
    			send_size, rcv_size, rcv_name,
    			timeout, notify, MACH_MSG_NULL, 0);
    
    if ((option & MACH_RCV_INTERRUPT) == 0)
    	while (mr == MACH_RCV_INTERRUPTED)
    		mr = mach_msg_overwrite_trap(msg,
    			option &~ (LIBMACH_OPTIONS|MACH_SEND_MSG),
    			0, rcv_size, rcv_name,
    			timeout, notify, MACH_MSG_NULL, 0);
    
    return mr;

}

```

可以看到程序最终调用了

mach_msg_overwrite_trap

## mach_msg_overwrite_trap


mach_msg_overwrite_trap在这里
https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/osfmk/ipc/mach_msg.c

发送消息如代码所示
​	

	if (option & MACH_SEND_MSG) {
			ipc_space_t space = current_space();
			ipc_kmsg_t kmsg;
	KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_START);
	
		mr = ipc_kmsg_get(msg_addr, send_size, &kmsg);
	
		if (mr != MACH_MSG_SUCCESS) {
			KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
			return mr;
		}
	
		KERNEL_DEBUG_CONSTANT(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_LINK) | DBG_FUNC_NONE,
				      (uintptr_t)msg_addr,
				      VM_KERNEL_ADDRPERM((uintptr_t)kmsg),
				      0, 0,
				      0);
	
		mr = ipc_kmsg_copyin(kmsg, space, map, override, &option);
	
		if (mr != MACH_MSG_SUCCESS) {
			ipc_kmsg_free(kmsg);
			KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
			return mr;
		}
	
		mr = ipc_kmsg_send(kmsg, option, msg_timeout);
	
		if (mr != MACH_MSG_SUCCESS) {
			mr |= ipc_kmsg_copyout_pseudo(kmsg, space, map, MACH_MSG_BODY_NULL);
			(void) ipc_kmsg_put(kmsg, option, msg_addr, send_size, 0, NULL);
			KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
			return mr;
		}
	
	}



## ipc_kmsg_get用来处理消息头

https://github.com/apple/darwin-xnu/blob/a449c6a3b8014d9406c2ddbdc81795da24aa7443/osfmk/ipc/ipc_kmsg.c

在ipc_kmsg_copyin的
ipc_kmsg_copyin_body用来接收body和ool_descriptor

### 处理oolport的分支

```
case MACH_MSG_OOL_VOLATILE_DESCRIPTOR:分支是这次漏洞利用的类型

  case MACH_MSG_OOL_PORTS_DESCRIPTOR: 
                user_addr = ipc_kmsg_copyin_ool_ports_descriptor((mach_msg_ool_ports_descriptor_t *)kern_addr, 
				            user_addr, is_task_64bit, map, space, dest, kmsg, optionp, &mr);
                kern_addr++;
                complex = TRUE;
                break;

```



可以看到最终调用了ipc_kmsg_copyin_ool_ports_descriptor





#### ipc_kmsg_copyin_ool_ports_descriptor

 

      data = kalloc(ports_length);
    if (data == NULL) {
        *mr = MACH_SEND_NO_BUFFER;
        return NULL;
    }
    #ifdef LP64
        mach_port_name_t *names = &((mach_port_name_t *)data)[count];
    #else
        mach_port_name_t *names = ((mach_port_name_t *)data);
    #endif
    if (copyinmap(map, addr, names, names_length) != KERN_SUCCESS) {
        kfree(data, ports_length);
        *mr = MACH_SEND_INVALID_MEMORY;
        return NULL;
    }
    
    if (deallocate) {
        (void) mach_vm_deallocate(map, addr, (mach_vm_size_t)ports_length);
    }
    
    objects = (ipc_object_t *) data;
    dsc->address = data;






可以看到最终申请了data来里面存着一堆mach_port_t指向一个object 并将object挂在entry上



## 释放

释放的时候调用了ipc_kmsg_add_trailer
将结果拷贝到字符串，原理大体一样。