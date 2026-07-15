---
title: jvm指标采集
date: 2026-07-14 13:36:50
categories: Essay
---

```Java
/**
     * JVM 指标采集包装器。
     *
     * 采集流程：
     * 1. 手动触发一次 GC，并等待 500ms，尽量让两次测试的起点更接近；
     * 2. 重置各堆内存池的 peakUsage，便于观察本次任务执行过程中的峰值；
     * 3. 执行业务任务；
     * 4. 输出执行耗时、堆内存、非堆内存、GC 次数和 GC 耗时。
     *
     * 注意：
     * - System.gc() 只是测试辅助，不代表生产环境行为；
     * - 性能对比建议每个方法至少跑 3 次，丢弃第一次热身结果；
     * - 两个方法要使用同一批待导出任务和同一套 JVM 参数，否则结果不可直接比较。
     */
    private void measureJvm(String name, Runnable task) {
        System.gc();
        sleep(500L);
        resetPeakMemoryUsage();
        GcSnapshot beforeGc = gcSnapshot();
        MemoryUsage beforeHeap = heapUsage();
        MemoryUsage beforeNonHeap = nonHeapUsage();
        long startNanos = System.nanoTime();
        try {
            task.run();
        } finally {
            long costNanos = System.nanoTime() - startNanos;
            MemoryUsage afterHeap = heapUsage();
            MemoryUsage afterNonHeap = nonHeapUsage();
            GcSnapshot afterGc = gcSnapshot();
            printJvmInfo();
            System.out.println(new JvmMetric(
                    name,
                    costNanos,
                    beforeHeap.getUsed(),
                    afterHeap.getUsed(),
                    peakHeapUsed(),
                    beforeNonHeap.getUsed(),
                    afterNonHeap.getUsed(),
                    afterGc.count - beforeGc.count,
                    afterGc.timeMillis - beforeGc.timeMillis
            ));
        }
    }

    /**
     * 输出本次测试 JVM 基础信息。
     *
     * java.version：当前 Java 版本。
     * jvm.args：本次测试进程的 JVM 参数，例如 -Xms、-Xmx、GC 类型等。
     * heap.init / heap.max：堆初始大小和最大大小。对比两条链路时，这两个值必须一致。
     */
    private void printJvmInfo() {
        MemoryUsage heap = heapUsage();
        System.out.println("==== JVM INFO ====");
        System.out.println("java.version=" + System.getProperty("java.version"));
        System.out.println("jvm.args=" + ManagementFactory.getRuntimeMXBean().getInputArguments());
        System.out.println("heap.init=" + toMb(heap.getInit()) + "MB, heap.max=" + toMb(heap.getMax()) + "MB");
    }

    private MemoryUsage heapUsage() {
        MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();
        return memoryMXBean.getHeapMemoryUsage();
    }

    private MemoryUsage nonHeapUsage() {
        MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();
        return memoryMXBean.getNonHeapMemoryUsage();
    }

    /**
     * 统计本次任务执行过程中所有 HEAP 内存池的峰值 used。
     *
     * 对比时优先看这个指标：
     * - 旧链路如果把文件完整放进 byte[]，峰值通常会被明显抬高；
     * - 流式链路如果生效，峰值应更接近业务数据本身，而不是额外叠加整份文件字节数组。
     */
    private long peakHeapUsed() {
        return ManagementFactory.getMemoryPoolMXBeans().stream()
                .filter(MemoryPoolMXBean::isValid)
                .filter(pool -> pool.getType() == MemoryType.HEAP)
                .map(MemoryPoolMXBean::getPeakUsage)
                .mapToLong(MemoryUsage::getUsed)
                .sum();
    }

    private void resetPeakMemoryUsage() {
        ManagementFactory.getMemoryPoolMXBeans().forEach(pool -> {
            try {
                pool.resetPeakUsage();
            } catch (UnsupportedOperationException ignored) {
                // Some JVM memory pools do not support peak reset.
            }
        });
    }

    /**
     * 汇总所有 GC 收集器的累计次数和累计耗时。
     *
     * measureJvm 会在任务前后各采一次快照，两者相减得到本次任务触发的 GC 压力。
     */
    private GcSnapshot gcSnapshot() {
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        long count = 0L;
        long timeMillis = 0L;
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            long collectionCount = gcBean.getCollectionCount();
            long collectionTime = gcBean.getCollectionTime();
            if (collectionCount > 0) {
                count += collectionCount;
            }
            if (collectionTime > 0) {
                timeMillis += collectionTime;
            }
        }
        return new GcSnapshot(count, timeMillis);
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private static long toMb(long bytes) {
        if (bytes < 0) {
            return bytes;
        }
        return bytes / 1024 / 1024;
    }

    private record GcSnapshot(long count, long timeMillis) {
    }

    /**
     * 单次任务的 JVM 指标输出。
     *
     * 字段说明：
     * - name：当前测试链路名称；
     * - cost：任务耗时；
     * - heap.before / heap.after：任务执行前后的堆已用内存；
     * - heap.delta：after - before，观察任务结束后是否留下明显堆占用；
     * - heap.peak：任务执行期间堆峰值，最适合观察大文件是否进入堆内存；
     * - nonHeap.before / nonHeap.after：元空间、Code Cache 等非堆内存变化，通常只做辅助参考；
     * - gc.count / gc.time：任务期间 GC 次数和 GC 耗时，越低通常表示内存压力越小。
     */
    private record JvmMetric(String name,
                             long costNanos,
                             long beforeHeapUsed,
                             long afterHeapUsed,
                             long peakHeapUsed,
                             long beforeNonHeapUsed,
                             long afterNonHeapUsed,
                             long gcCount,
                             long gcTimeMillis) {

        @Override
        public String toString() {
            return "==== JVM METRIC ====\n"
                    + "name=" + name + "\n"
                    + "cost=" + Duration.ofNanos(costNanos).toMillis() + "ms\n"
                    + "heap.before=" + toMb(beforeHeapUsed) + "MB\n"
                    + "heap.after=" + toMb(afterHeapUsed) + "MB\n"
                    + "heap.delta=" + toMb(afterHeapUsed - beforeHeapUsed) + "MB\n"
                    + "heap.peak=" + toMb(peakHeapUsed) + "MB\n"
                    + "nonHeap.before=" + toMb(beforeNonHeapUsed) + "MB\n"
                    + "nonHeap.after=" + toMb(afterNonHeapUsed) + "MB\n"
                    + "gc.count=" + gcCount + "\n"
                    + "gc.time=" + gcTimeMillis + "ms";
        }
    }
```

#### 使用方法

```java
@Autowired
    private ExportTaskJob exportTaskJob;

    /**
     * 旧导出链路性能观察入口。
     *
     * 使用方式：
     * 1. 单独运行本测试方法；
     * 2. 记录控制台中 JVM INFO / JVM METRIC 的输出。
     *
     * 对比方式：
     * - 再单独运行 exportTaskWithStream()；
     * - 对比两次输出中的 cost、heap.delta、heap.peak、gc.count、gc.time。
     * - uploadFile 链路如果存在大 byte[] / MockMultipartFile，通常 heap.delta 和 heap.peak 会更高。
     */
    @Test
    public void exportTask() {
        //measureJvm("exportTask(uploadFile)", () -> exportTaskJob.exportTask());
    }

    /**
     * 流式导出链路性能观察入口。
     *
     * 使用方式同 exportTask()，但本方法用于观察 exportTaskWithStream / uploadLargeFile 链路。
     *
     * 重点看：
     * - heap.delta 是否比旧链路更低：表示执行后堆内存净增长更少；
     * - heap.peak 是否比旧链路更低：表示执行过程中的堆内存峰值更低；
     * - gc.count / gc.time 是否下降：表示导出过程中 GC 压力降低；
     * - cost 只能作为参考，流式写磁盘/网络后可能更稳，但不一定每次都更快。
     */
    @Test
    public void exportTaskWithStream() {
        //measureJvm("exportTaskWithStream(uploadLargeFile)", () -> exportTaskJob.exportTaskWithStream());
    }
```
