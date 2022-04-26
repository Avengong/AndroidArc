````

const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = compat_ptr_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};

static int __init init_binder_device(const char *name)
{
	int ret;
	struct binder_device *binder_device;

	binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
	if (!binder_device)
		return -ENOMEM;

	binder_device->miscdev.fops = &binder_fops; //这个就是驱动注册函数
	binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
	binder_device->miscdev.name = name;

	refcount_set(&binder_device->ref, 1);
	binder_device->context.binder_context_mgr_uid = INVALID_UID;
	binder_device->context.name = name;
	mutex_init(&binder_device->context.context_mgr_node_lock);
     // 调用misc_register注册binder驱动
	ret = misc_register(&binder_device->miscdev);
	if (ret < 0) {
		kfree(binder_device);
		return ret;
	}

	hlist_add_head(&binder_device->hlist, &binder_devices);

	return ret;
}

```


```

static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc, *itr; //每一个进程对应一个binder_proc结构体
	struct binder_device *binder_dev;
	struct binderfs_info *info;
	struct dentry *binder_binderfs_dir_entry_proc = NULL;
	bool existing_pid = false;

	proc = kzalloc(sizeof(*proc), GFP_KERNEL); // 分配内存空间
	if (proc == NULL)
		return -ENOMEM;
	...
	INIT_LIST_HEAD(&proc->todo); //初始化todo 链表
	init_waitqueue_head(&proc->freeze_wait); wait链表
	
	/* binderfs stashes devices in i_private */
	...
	refcount_inc(&binder_dev->ref);
	proc->context = &binder_dev->context;
	binder_alloc_init(&proc->alloc);
	binder_stats_created(BINDER_STAT_PROC);
	proc->pid = current->group_leader->pid;
	INIT_LIST_HEAD(&proc->delivered_death);
	INIT_LIST_HEAD(&proc->waiting_threads);
	filp->private_data = proc; // file中的private_data指向biner_proc对象

	mutex_lock(&binder_procs_lock);
	hlist_for_each_entry(itr, &binder_procs, proc_node) {
		if (itr->pid == proc->pid) {
			existing_pid = true;
			break;
		}
	}
	hlist_add_head(&proc->proc_node, &binder_procs); // 把binder_node节点加到binder_proc表头的队列中
	mutex_unlock(&binder_procs_lock);
    ...
	return 0;
}

```


```
static int binder_mmap(struct file *filp, struct vm_area_struct *vma) //vma 是应用进程的某块虚拟内存区域
{
    // 得到进程的proc结构体
	struct binder_proc *proc = filp->private_data;

	if (proc->tsk != current->group_leader)
		return -EINVAL;

	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
		       proc->pid, vma->vm_start, vma->vm_end, "bad vm_flags", -EPERM);
		return -EPERM;
	}
	vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
	vma->vm_flags &= ~VM_MAYWRITE;

	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;

	return binder_alloc_mmap_handler(&proc->alloc, vma);
}



int binder_alloc_mmap_handler(struct binder_alloc *alloc,
			      struct vm_area_struct *vma)
{
	int ret;
	const char *failure_string;
	struct binder_buffer *buffer; // binder驱动对应的物理内存的内核中虚拟内存描述

	mutex_lock(&binder_alloc_mmap_lock);
	if (alloc->buffer_size) { //是否已经映射？？
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}
	alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start,
				   SZ_4M);// 分配应用的虚拟内存空间，最大值为4M。
	mutex_unlock(&binder_alloc_mmap_lock);

	alloc->buffer = (void __user *)vma->vm_start; //buffer指向应用进程的虚拟内存的起始地址

	alloc->pages = kcalloc(alloc->buffer_size / PAGE_SIZE,
			       sizeof(alloc->pages[0]),
			       GFP_KERNEL); //计算偏移量offset。实际上只为进程分配了一个物理页的大小 4k
	if (alloc->pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}

	buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
	if (!buffer) {
		ret = -ENOMEM;
		failure_string = "alloc buffer struct";
		goto err_alloc_buf_struct_failed;
	}

	buffer->user_data = alloc->buffer;
	list_add(&buffer->entry, &alloc->buffers);
	buffer->free = 1;
	binder_insert_free_buffer(alloc, buffer);
	alloc->free_async_space = alloc->buffer_size / 2;
	binder_alloc_set_vma(alloc, vma);
	mmgrab(alloc->vma_vm_mm);

	return 0;

	return ret;
}




```


```
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;

	/*pr_info("binder_ioctl: %d:%d %x %lx\n",
			proc->pid, current->pid, cmd, arg);*/

	binder_selftest_alloc(&proc->alloc);

	trace_binder_ioctl(cmd, arg);

	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret)
		goto err_unlocked;

	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	case BINDER_WRITE_READ: // 接收和发送binder数据
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	...
	case BINDER_SET_CONTEXT_MGR_EXT: { // 设置serviceManager为全局静态的binder大管家
		struct flat_binder_object fbo;

		if (copy_from_user(&fbo, ubuf, sizeof(fbo))) {
			ret = -EINVAL;
			goto err;
		}
		ret = binder_ioctl_set_ctx_mgr(filp, &fbo);
		if (ret)
			goto err;
		break;
	}
	...
	default:
		ret = -EINVAL;
		goto err;
	}
	ret = 0;

	return ret;
}

```


```
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	//把数据从用户空间拷贝到内核空间
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
        
    // write_size>0，表示进程向驱动中写数据
	if (bwr.write_size > 0) {
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		if (ret < 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	// read_size>0，表示进程从驱动中读数据
	if (bwr.read_size > 0) {
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&proc->todo))
			binder_wakeup_proc_ilocked(proc);
		binder_inner_proc_unlock(proc);
		if (ret < 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}



```




