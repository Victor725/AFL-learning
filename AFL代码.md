# AFL代码

## afl-gcc





## afl-fuzz

1. 从main函数开始看，首先进入while循环，读取在控制台输入的命令中的选项

```c
  while ((opt = getopt(argc, argv, "+i:o:f:m:b:t:T:dnCB:S:M:x:QV")) > 0)

    switch (opt) {
            ...
    }
```

2. 设置信号处理函数

```c
setup_signal_handlers();
```

3. 检查ASAN选项（ASAN用于检测内存错误）

```c
check_asan_opts();
```

4. 存储命令行参数

```c
save_cmdline(argc, argv);
```

5. 获取环境相关的信息等
6. 建立共享内存

```
setup_shm();//初始化了virgin_bits   trace_bits 两者分别用于存储未被覆盖的边和各个边的覆盖率（通过哈希得到下标）
```

进入查看：

* 调用memset，为virgin_bits分配空间
* 将virgin_bits   virgin_tmout   virgin_crash 初始化为全1，表示此时没有任何区域是已经被触及的、已经超时过的和已经触发崩溃的
* 分配共享内存，并将其连接到当前进程的地址空间，这样，便生成了trace_bits（有检测位图的共享内存 大小64kb）

7. 加载目录，读取测试用例等

8. 开始Fuzz

```c
perform_dry_run(use_argv); 
```

进入此函数查看：

* 主体是一个大的while循环

* 分配use_mem，将测试用例读入其中

* 对队列中的每一个测试用例，进行校准，之后针对不同的校准错误进行处理。校准时调用：

  ```
  res = calibrate_case(argv, q, use_mem, 0, 1);
  ```

  进入此函数查看：

  * 首先进行一系列初始化工作

  * 确定运行次数  若设定了快速校准，则运行3次，否则8次

  * 若没有forkserver，则建立forkserver

    ```c
    init_forkserver(argv);
    ```

    * 进入查看

      * 进行fork

      * 子进程就是forkserver，它会执行被测程序

        ```
        execv(target_path, argv);
        ```

      * 父进程（Fuzzer）进行等待（`setitimer()`），之后检查管道，得到子进程（forkserver）"hello"的回复后返回，否则处理错误

      * 这里要注意，到这里，子进程（forkserver）去运行了被测程序，同时，应当注意，之前对其插了桩，插入的代码在afl-as.h中。首先运行`__afl_maybe_log()`，这个函数会将"hello"（即上一步中父进程（fuzzer）要等的回复）写入管道，告诉父进程（fuzzer）一切正常，然后进入`__afl_fork_wait_loop`，读取管道，直到fuzzer通知其进行fork，fork后，子进程（真正运行测试用例的进程）跳到`__afl_fork_resume`，关闭无用的管道，继续执行，父进程（forkserver）则继续作为forkserver，这里注意！！！有个坑，AT&T汇编中，

        ```
        "  call fork\n"
        "\n"
        "  cmpl $0, %eax\n"
        "  jl   __afl_die\n"
        "  je   __afl_fork_resume\n"
        ```

        这一段的意思是，%eax<0时（fork失败），转向die，=0时（子进程），转向resume，这里cmp的含义和x86中的不一样

      * 父进程（forkserver）继续进入`__afl_fork_wait_loop`

  * 若校验和不为0，则拷贝trace_bits到first_trace,并检查virgin_bits中是否有新情况

  * 进入for循环，循环次数为3或8次（由前面确定）

    * 调用write_to_testcase，将修改后的use_mem写入测试用例

    * 调用run_target，通知forkserver准备fork并fuzz（将trace_bits设为0，将信息写入管道，读取状态管道）  

      * 注意！！！这里要结合插桩的代码进行理解，一些fuzz的逻辑隐含在插入被测程序的代码中（如改变trace_bits等）？？？？？？

    * 看是否出现了新情况
    
      ```
      cksum = hash32(trace_bits, MAP_SIZE, HASH_CONST);
      ```
    
    * 跟之前的校验和（q->exec_cksum）作比较，若不等，则是出现了新情况
    
      * q->exec_cksum若为0，说明是第一次执行，继续进行初始化操作，即令其等于cksum，并将trace_bits复制到first_trace中
      * 若不为0，如果first_trace的某一位和trace_bits的不一样，就说明该位发生了变化，将var_bytes的相应位置置1，并让这个用例多运行几次，即将次数调成40
    
  * 之后，进行一定的处理工作（计算运行时间等）

  * 更新位图的得分（update_bitmap_score）目的是找到top_rated，即最小的、消耗最短时间到达该边的测试用例，进而找到这些“最优”用例组成的集合——favorables，通过这个集合中的用例，就能通过最小的空间、时间代价来到达所有的边

    * ​	对于每个trace_bits，若被置了值
      * 若相应位置的top_rated不为空，说明有竞争者，比较运行时间x用例大小的值，此值较小的为更优，若top_rated不是较小的，则将他的引用数减一，减一后为0，就将他释放
      * 若top_rated为空，则将这个用例作为top_rated，将其引用数加一

* 如果刚才校准测试用例没有出错，就会返回FAULT_NONE，此时，会执行check_map_coverage()，检查覆盖范围

* 处理错误并返回

9. 调用 cull_queue()

   它遍历 top_rated[] 条目，然后顺序地获取先前未见过的字节 (temp_v) 的获胜者并将它们标记为喜欢的条目，至少直到下一次运行。

   在所有模糊测试步骤中，喜欢的条目会获得更多的运行时间。

进入查看：

* 将temp_v置为全1
* 将队列所有queue_entry的favored的属性置为零
* 对于每一条边，若它有top_rated，且没有在temp_v中（说明此边已经被到达过），删除top_rated中属于当前条目的所有位，同时，将当前queue_entry设置为favored
* 进行遍历，若当前queue_entry不是favored，就将他标注为冗余的，这样，就达到了筛选queue的目标

10. 保存一系列信息，显示一系列信息
11. 进入主循环

* 挑选用例（cull_queue()）

* 若当前用例的指针为空，说明已经遍历了一遍，进行一些处理，从头开始
* 用fuzz_one对用例进行fuzz，具体来讲，就是先校准用例（`calibrate_case`，在这个函数内真正运行用例），修剪用例（防止过大），对用例进行评分，对用例进行变异

12. 写入bitmap，做一些收尾工作，结束
