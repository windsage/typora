# How to cross compile fio and hdparm



## Fio
https://blog.csdn.net/u011649400/article/details/113759374
https://blog.csdn.net/aa787282301/article/details/109230094
https://fio.readthedocs.io/en/latest/fio_doc.html#building

```bash
#需要remove llog选项，编译static的话没有办法-llog
UNAME=Android && ./configure --cpu=aarch64 --prefix=./tmp/fio --disable-shm --build-static
make -j$(nproc) LDFLAGS="-static"  

# dynamic
UNAME=Android CROSS_COMPILE=/home/UserName/Android/Sdk/ndk/22.0.7026061/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21- CC=/home/UserName/Android/Sdk/ndk/22.0.7026061/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang ./configure
```




## hdparm
https://linkscue.com/posts/2018-04-27-how-to-cross-build-hdparm/