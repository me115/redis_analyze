

1.柔性数组的使用；
::

    // sdshdr 结构
    struct sdshdr {

        // buf 已占用长度
        int len;

        // buf 剩余可用长度
        int free;

        // 实际保存字符串数据的地方
        // 利用 flexible array member,通过buf来引用sdshdr后面的地址，
        // 详情google "flexible array member"
        char buf[];
    };


::

    sds sdsnew(const char *init) {
        size_t initlen = (init == NULL) ? 0 : strlen(init);
        struct sdshdr *sh;
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
        sh->len = initlen;
        sh->free = 0;
        memcpy(sh->buf, init, initlen);
        return (char*)sh->buf;
    }


为什么要返回buf(sh->buf)，而不返回sds的结构指针？
使用buf的可以更简单，更多的时候是在操作字符串，可以将sds结构记录信息都不再考虑，直接当作char*字符串来使用；
比如转字符串转大写的过程，sds拿过来直接引用下标进行转换，完全屏蔽sds结构信息的影子：
::

    void sdstoupper(sds s) {
        int len = sdslen(s), j;

        for (j = 0; j < len; j++) s[j] = toupper(s[j]);
    }




由于sds保存的是buf的指针，要获得整个buf的指针就需要减去相应的结构偏移量（柔性数组的使用）：
struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));

对于删除，并非调用系统的释放内存的函数，而是将其放入到程序的内存管理空间中，
::

    void sdsclear(sds s) {

        struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));

        sh->free += sh->len;
        sh->len = 0;
        sh->buf[0] = '\0';
    }



可变参数列表：va_list,va_arg,va_copy,va_start,va_end
va_list arg_ptr：定义一个指向个数可变的参数列表指针；
va_copy(dest, src)：dest，src的类型都是va_list，va_copy()用于复制参数列表指针，将dest初始化为src
关系va_list详情，参考这里\ [#]_\ , 
如果我们不需要一一详解每个参数，只需要将可变列表拷贝至某个缓冲，可用vsprintf/vsnsprintf函数；

::

    sds sdscatvprintf(sds s, const char *fmt, va_list ap) {
        va_list cpy;
        char *buf, *t;
        size_t buflen = 16;

        while(1) {
            buf = zmalloc(buflen);
            if (buf == NULL) return NULL;
            buf[buflen-2] = '\0';
            va_copy(cpy,ap);
            vsnprintf(buf, buflen, fmt, cpy);
            if (buf[buflen-2] != '\0') {
                zfree(buf);
                buflen *= 2;
                continue;
            }
            break;
        }
        t = sdscat(s, buf);
        zfree(buf);
        return t;
    }

总结
====================
柔性数组
可变参数列表
新生成的结构直接返回buf指针

.. [#] va_arg、va_copy、va_end、va_start: http://technet.microsoft.com/zh-cn/office/kb57fad8
