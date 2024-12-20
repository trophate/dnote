（redhat）设置kdump保留内存：

```sh
grubby --update-kernel=ALL --args="crashkernel=0"
```

