SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)

call ksys_read

​	call fdget_pos // convert long fd to struct fd

​	call vfs_read

​		call rw_verify_area

​		call __vfs_read

​			call file->f_op->read(file, buf, count, pos);

假设我们使用的ext4文件系统

```c
const struct file_operations ext4_file_operations = {
	.llseek		= ext4_llseek,
	.read_iter	= ext4_file_read_iter,
	.write_iter	= ext4_file_write_iter,
	.unlocked_ioctl = ext4_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= ext4_compat_ioctl,
#endif
	.mmap		= ext4_file_mmap,
	.mmap_supported_flags = MAP_SYNC,
	.open		= ext4_file_open,
	.release	= ext4_release_file,
	.fsync		= ext4_sync_file,
	.get_unmapped_area = thp_get_unmapped_area,
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
	.fallocate	= ext4_fallocate,
};
```



call ext4_file_read_iter

```c
	if (iocb->ki_flags & IOCB_DIRECT)
		return ext4_dio_read_iter(iocb, to);

	return generic_file_read_iter(iocb, to);
```

call generic_file_read_iter

call generic_file_buffered_read

check page cache

call error = mapping->a_ops->readpage(filp, page);



(1) 页缓存（page cache）针对以页为单位的所有操作，并考虑了特定体系结构上的页长度。一个

主要的例子是许多章讨论过的内存映射技术。因为其他类型的文件访问也是基于内核中的这一技术实

现的，所以页缓存实际上负责了块设备的大部分缓存工作。

(2) 块缓存（buffer cache）以块为操作单位。在进行I/O操作时，存取的单位是设备的各个块，而

不是整个内存页。尽管页长度对所有文件系统都是相同的，但块长度取决于特定的文件系统或其设置。

因而，块缓存必须能够处理不同长度的块。