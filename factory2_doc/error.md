# DGL

## `GLIBCXX_3.4.26` not found

```bash
ImportError: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.26' not found
```

### solve:

这个是默认路径下的`libstdc++.so.6`缺少`GLIBCXX_3.4.26`，你有可能缺少其它版本的比如3.4.26

- 使用指令先看下目前都有哪些版本的，到3.4.21

```bash
strings /usr/lib/x86_64-linux-gnu/libstdc++.so.6 | grep GLIBCXX
```

![image-20240627125634284](C:\Users\DG2024\AppData\Roaming\Typora\typora-user-images\image-20240627125634284.png)

- 使用`find / -name "libstdc++.so.6*"`来查看当前系统中其它的同类型文件，找到一个版本比较高的，我这里列出如下：

![image-20240627125930814](C:\Users\DG2024\AppData\Roaming\Typora\typora-user-images\image-20240627125930814.png)

- 选了一个版本较高的使用之前的指令看看其是否包含需要的版本

```bash
strings /gemini/data-1/my_env/faiss/lib/libstdc++.so.6.0.32 | grep GLIBCXX_3.4.2
```

​		运行后得到如下结果

![image-20240627130018234](C:\Users\DG2024\AppData\Roaming\Typora\typora-user-images\image-20240627130018234.png)

可以看到有需要的版本，接下来就是建立新的链接到这个文件上

- 复制到指定目录并建立新的链接

```bash
cp /gemini/data-1/my_env/faiss/lib/libstdc++.so.6.0.32 /usr/lib/x86_64-linux-gnu
```

------

==~~删除之前链接~~==

==~~`rm /usr/lib/x86_64-linux-gnu/libstdc++.so.6`~~==

==~~创建新的链接~~==

==~~`ln -s /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.32 /usr/lib/x86_64-linux-gnu/libstdc++.so.6`~~==

==千万别乱删`lib.so`这种文件，不然你的linux系统可能会崩溃，类似于这种==

==`ls: error while loading shared libraries: libm.so.6: cannot open shared object file: no such file or directory`==

------

可直接创建软连接，强制执行：

```bash
ln -sf /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.32 /usr/lib/x86_64-linux-gnu/libstdc++.so.6
```



# faiss

问题描述：将index从CPU转移到GPU卡死，无响应

卸载faiss-gpu，conda重新安装：

`conda install conda-forge::faiss-gpu`

![image-20240705144112639](C:\Users\DG2024\AppData\Roaming\Typora\typora-user-images\image-20240705144112639.png)



