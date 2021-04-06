调用字符串解密函数

```
extern "C" char* sub_1f120(int a1);

void adc_main_thread() {
    printf("2531 %s\n",sub_1f120(2531));
}

```



dump so文件脚本

```
start1 = 0xd79df000
end1 = 0xd79f8000

start2 = 0xd79f9000
end2 = 0xd79fc000


def dump(start,end):
    page_bytes = b''
    for i in range((end-start) // 0x1000 ):
        print(i)
        page_bytes = page_bytes + readMemory(start+i*0x1000, 0x1000)
    return page_bytes

so_bytes = b''
so_bytes = so_bytes + dump(start1,end1)
so_bytes = so_bytes + dump(start2,end2)
print("%x"%len(so_bytes))

with open("D:\\so_dump1.so",'wb') as f:
    f.write(so_bytes)
    f.close()
```

