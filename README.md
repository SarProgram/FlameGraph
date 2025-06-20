本文源自：https://github.com/brendangregg/FlameGraph

# 火焰图可视化分析代码性能

官方网站： http://www.brendangregg.com/flamegraphs.html


示例（点击放大）：

[![示例](http://www.brendangregg.com/FlameGraphs/cpu-bash-flamegraph.svg)](http://www.brendangregg.com/FlameGraphs/cpu-bash-flamegraph.svg)

点击某个框以将火焰图缩放至该调用栈帧。
要搜索并高亮所有匹配正则表达式的调用栈帧，请点击右上角的 _search_ 按钮或按 Ctrl-F。
默认情况下，搜索是区分大小写的，但可以通过按 Ctrl-I 或点击右上角的 _ic_ 按钮来切换。

其他站点：
- ACMQ 和 CACM 上的火焰图文章：http://queue.acm.org/detail.cfm?id=2927301 http://cacm.acm.org/magazines/2016/6/202665-the-flame-graph/abstract
- 使用 Linux perf\_events、DTrace、SystemTap 或 ktap 进行 CPU 分析：http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html
- 使用 XCode Instruments 进行 CPU 分析：http://schani.wordpress.com/2012/11/16/flame-graphs-for-instruments/  
- 使用 Xperf.exe 进行 CPU 分析：http://randomascii.wordpress.com/2013/03/26/summarizing-xperf-cpu-usage-with-flame-graphs/  
- 内存分析：http://www.brendangregg.com/FlameGraphs/memoryflamegraphs.html  
- 更多示例、更新和新闻：http://www.brendangregg.com/flamegraphs.html#Updates

火焰图的生成分为三步：

1. 采集调用栈
2. 折叠调用栈
3. 执行 flamegraph.pl

1\. 采集调用栈
=================
调用栈样本可使用 Linux perf\_events、FreeBSD pmcstat（hwpmc）、DTrace、SystemTap 及其他分析器采集。请查看 stackcollapse-\* 转换器。

### Linux perf\_events

使用 Linux perf\_events（即 “perf”）以 99 Hz 频率采集 60 秒的调用栈样本（包含用户态和内核态，所有进程）：

```
# perf record -F 99 -a -g -- sleep 60
# perf script > out.perf
```

仅采集 PID 为 181 的进程：

```
# perf record -F 99 -p 181 -g -- sleep 60
# perf script > out.perf
```

### DTrace

使用 DTrace 以 997 Hz 频率采集 60 秒的内核调用栈：

```
# dtrace -x stackframes=100 -n 'profile-997 /arg0/ { @[stack()] = count(); } tick-60s { exit(0); }' -o out.kern_stacks
```

使用 DTrace 以 97 Hz 频率采集 PID 为 12345 的进程的用户态调用栈：

```
# dtrace -x ustackframes=100 -n 'profile-97 /pid == 12345 && arg1/ { @[ustack()] = count(); } tick-60s { exit(0); }' -o out.user_stacks
```

包含内核时间的用户态调用栈（60 秒，97 Hz）：

```
# dtrace -x ustackframes=100 -n 'profile-97 /pid == 12345/ { @[ustack()] = count(); } tick-60s { exit(0); }' -o out.user_stacks
```

若应用程序提供 ustack 助手，可使用 `jstack()` 代替 `ustack()` 以包含翻译后的调用帧（如 node.js；参见：http://dtrace.org/blogs/dap/2012/01/05/where-does-your-node-program-spend-its-time/）。用户态采样频率有意设为低于内核，尤其是使用 `jstack()` 时，以避免其在帧翻译上的额外开销。

2\. 折叠调用栈
==============
使用 stackcollapse 系列程序将调用栈样本折叠为单行。可用的程序包括：

- `stackcollapse.pl`：用于 DTrace 栈
- `stackcollapse-perf.pl`：用于 Linux perf_events “perf script” 输出
- `stackcollapse-pmc.pl`：用于 FreeBSD pmcstat -G
- `stackcollapse-stap.pl`：用于 SystemTap 栈
- `stackcollapse-instruments.pl`：用于 XCode Instruments
- `stackcollapse-vtune.pl`：用于 Intel VTune
- `stackcollapse-ljp.awk`：用于 Lightweight Java Profiler
- `stackcollapse-jstack.pl`：用于 Java jstack(1) 输出
- `stackcollapse-gdb.pl`：用于 gdb(1) 栈
- `stackcollapse-go.pl`：用于 Golang pprof 栈
- `stackcollapse-vsprof.pl`：用于 Microsoft Visual Studio
- `stackcollapse-wcp.pl`：用于 wallClockProfiler 输出

示例用法：


```
For perf_events:
$ ./stackcollapse-perf.pl out.perf > out.folded

For DTrace:
$ ./stackcollapse.pl out.kern_stacks > out.kern_folded
```

输出类似如下格式：

```
unix`_sys_sysenter_post_swapgs 1401
unix`_sys_sysenter_post_swapgs;genunix`close 5
unix`_sys_sysenter_post_swapgs;genunix`close;genunix`closeandsetf 85
unix`_sys_sysenter_post_swapgs;genunix`close;genunix`closeandsetf;c2audit`audit_closef 26
unix`_sys_sysenter_post_swapgs;genunix`close;genunix`closeandsetf;c2audit`audit_setf 5
unix`_sys_sysenter_post_swapgs;genunix`close;genunix`closeandsetf;genunix`audit_getstate 6
unix`_sys_sysenter_post_swapgs;genunix`close;genunix`closeandsetf;genunix`audit_unfalloc 2
unix`_sys_sysenter_post_swapgs;genunix`close;genunix`closeandsetf;genunix`closef 48
[...]
```

3\. flamegraph.pl
================
使用 flamegraph.pl 渲染为 SVG：

```
$ ./flamegraph.pl out.kern_folded > kernel.svg
```

拥有已折叠的输入文件的好处是可以用 grep 提取关注函数，例如：

```
$ grep cpuid out.kern_folded | ./flamegraph.pl > cpuid.svg
```

提供的示例
=================

### Linux perf\_events

包含了来自 Linux “perf script” 的 gzip 示例：example-perf-stacks.txt.gz，生成的火焰图为 example-perf.svg：

[![示例](http://www.brendangregg.com/FlameGraphs/example-perf.svg)](http://www.brendangregg.com/FlameGraphs/example-perf.svg)

您可以通过以下命令生成：

```
$ gunzip -c example-perf-stacks.txt.gz | ./stackcollapse-perf.pl --all | ./flamegraph.pl --color=java --hash > example-perf.svg
```

这展示了我的典型工作流：在目标机 gzip 压缩分析数据，再拷贝至本地分析。由于有数百个 profile，均保持压缩状态。

由于包含了 Java，使用了 flamegraph.pl 的 --color=java 颜色方案。同时使用了 stackcollapse-perf.pl 的 --all 选项，以便 flamegraph.pl 能区分内核与用户态代码。图中颜色含义：绿色 = Java，黄色 = C++，红色 = 用户态 native，橙色 = 内核。

该 profile 来源于 vert.x 性能分析，基准客户端 wrk 也出现在图中。

### DTrace

也提供了来自 DTrace 的示例输出 example-dtrace-stacks.txt，生成火焰图为 example-dtrace.svg：

[![示例](http://www.brendangregg.com/FlameGraphs/example-dtrace.svg)](http://www.brendangregg.com/FlameGraphs/example-dtrace.svg)

生成命令如下：

```
$ ./stackcollapse.pl example-stacks.txt | ./flamegraph.pl > example.svg
```

此示例来自一次性能调查，火焰图指出 CPU 时间花费在 lofs 模块，并量化了该时间。

选项
=======
使用 `--help` 查看 flamegraph.pl 可用选项：

用法：./flamegraph.pl [options] infile > outfile.svg

	--title TEXT     # change title text
	--subtitle TEXT  # second level title (optional)
	--width NUM      # width of image (default 1200)
	--height NUM     # height of each frame (default 16)
	--minwidth NUM   # omit smaller functions. In pixels or use "%" for 
	                 # percentage of time (default 0.1 pixels)
	--fonttype FONT  # font type (default "Verdana")
	--fontsize NUM   # font size (default 12)
	--countname TEXT # count type label (default "samples")
	--nametype TEXT  # name type label (default "Function:")
	--colors PALETTE # set color palette. choices are: hot (default), mem,
	                 # io, wakeup, chain, java, js, perl, red, green, blue,
	                 # aqua, yellow, purple, orange
	--bgcolors COLOR # set background colors. gradient choices are yellow
	                 # (default), blue, green, grey; flat colors use "#rrggbb"
	--hash           # colors are keyed by function name hash
	--cp             # use consistent palette (palette.map)
	--reverse        # generate stack-reversed flame graph
	--inverted       # icicle graph
	--flamechart     # produce a flame chart (sort by time, do not merge stacks)
	--negate         # switch differential hues (blue<->red)
	--notes TEXT     # add notes comment in SVG (for debugging)
	--help           # this message

	eg,
	./flamegraph.pl --title="Flame Graph: malloc()" trace.txt > graph.svg

如上所示，火焰图可用于处理任何事件的调用栈，例如 malloc()，前提是采集了调用栈。

一致调色板
==================
如果使用 `--cp` 选项，将使用 $colors 并生成随机调色板。
后续所有使用 `--cp` 的火焰图将复用相同 palette.map 文件。
新出现的符号将按相同配色逻辑分配新颜色。

若不喜欢当前调色板，可删除 palette.map 文件。

这有助于在对比火焰图时高亮差异。

示例：

假设有两个采样数据，一个出错，一个正常：

```
cat working.folded | ./flamegraph.pl --cp > working.svg
# this generates a palette.map, as per the normal random generated look.

cat broken.folded | ./flamegraph.pl --cp --colors mem > broken.svg
# this svg will use the same palette.map for the same events, but a very
# different colorscheme for any new events.
```

参见 demo 目录示例：

palette-example-working.svg  
palette-example-broken.svg
