

---

蒙特卡洛 NUMA 高负载 P99 测试落地步骤

假设环境

· 服务器：双路 NUMA 节点（节点0：CPU 0-15，节点1：CPU 16-31）
· OS：CentOS 7 / Ubuntu 20.04+
· 已安装：stress-ng, numactl, jmeter, jHiccup（或直接使用 MyPerf4J / Micrometer，但本例选择零侵入的 jHiccup 便于快速展示）
· Java：JDK 11+

---

第一阶段：环境准备（预计 10 分钟）

1. 确认 NUMA 拓扑

```bash
numactl --hardware
```

记录下节点数量和每个节点绑定的 CPU 范围。例如：

```
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
node 1 cpus: 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
```

2. 关闭 NUMA 自动平衡（重要！）

```bash
echo 0 | sudo tee /proc/sys/kernel/numa_balancing
```

测试完成后恢复：

```bash
echo 1 | sudo tee /proc/sys/kernel/numa_balancing
```

3. 准备测试目录

```bash
mkdir -p ~/montecarlo-test && cd ~/montecarlo-test
```

4. 下载 jHiccup（如果需要）

```bash
wget https://github.com/giltene/jHiccup/releases/download/2.0.10/jHiccup-2.0.10.jar
```

---

第二阶段：编写蒙特卡洛 Java 应用（预计 5 分钟）

创建文件 MonteCarloServer.java：

```java
import com.sun.net.httpserver.HttpServer;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpExchange;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.util.concurrent.Executors;
import java.util.Random;

public class MonteCarloServer {
    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
        server.setExecutor(Executors.newFixedThreadPool(30));
        server.createContext("/work", new WorkHandler());
        server.start();
        System.out.println("MonteCarloServer running on 8080");
    }

    static class WorkHandler implements HttpHandler {
        private final Random rand = new Random();

        @Override
        public void handle(HttpExchange exchange) throws java.io.IOException {
            long start = System.nanoTime();

            double p = rand.nextDouble();
            int n;
            if (p < 0.70) {
                n = 25;                  // 70% 轻请求
            } else if (p < 0.95) {
                n = 35;                  // 25% 中请求
            } else {
                n = 42;                  // 5% 重请求，拉高 P99
            }

            fibonacci(n);                // CPU 计算

            long duration = (System.nanoTime() - start) / 1_000_000;
            String resp = "{\"type\":\"" + (n<=25?"light":n<=35?"medium":"heavy") + "\",\"ms\":" + duration + "}";
            exchange.getResponseHeaders().set("Content-Type", "application/json");
            exchange.sendResponseHeaders(200, resp.getBytes().length);
            try (OutputStream os = exchange.getResponseBody()) {
                os.write(resp.getBytes());
            }
        }

        static long fibonacci(int n) {
            if (n <= 1) return n;
            return fibonacci(n - 1) + fibonacci(n - 2);
        }
    }
}
```

---

第三阶段：准备 JMeter 测试计划（预计 10 分钟）

最简单方式是通过 GUI 生成，但这里给出直接可用的纯文本 JMX 文件（无需 GUI），保存为 montecarlo-test.jmx：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testname="MonteCarlo NUMA P99 Test">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments"/>
      </elementProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testname="Steady Load">
        <intProp name="ThreadGroup.num_threads">50</intProp>   <!-- 并发线程数，可根据需求调整 -->
        <intProp name="ThreadGroup.ramp_time">10</intProp>
        <longProp name="ThreadGroup.duration">300</longProp>   <!-- 压测持续 5 分钟 -->
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testname="Work Request">
          <stringProp name="HTTPSampler.domain">localhost</stringProp>
          <stringProp name="HTTPSampler.port">8080</stringProp>
          <stringProp name="HTTPSampler.path">/work</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
        <hashTree/>
      </hashTree>
      <ResultCollector guiclass="StatVisualizer" testname="Aggregate Report">
        <boolProp name="ResultCollector.error_logging">false</boolProp>
        <stringProp name="filename">/tmp/montecarlo-result.jtl</stringProp>
      </ResultCollector>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

可按需修改 num_threads 和 duration。

---

第四阶段：一键执行测试脚本

创建并赋予执行权限文件 run_montecarlo_test.sh：

```bash
#!/bin/bash
set -e

# 配置
JAVA_APP="java -javaagent:./jHiccup-2.0.10.jar -l /tmp/jhiccup.hlog MonteCarloServer.java"
STRESS_DURATION=300          # 压力持续时间
COOLDOWN=60
JMETER_HOME="/opt/apache-jmeter-5.5"   # 根据实际路径修改
JMETER_BIN="$JMETER_HOME/bin/jmeter"

echo "=============================="
echo "场景1: 基线（无 stress-ng）"
echo "=============================="
numactl --cpunodebind=1 --membind=1 $JAVA_APP &
APP_PID=$!
sleep 20   # 等待服务完全启动
$JMETER_BIN -n -t montecarlo-test.jmx -l /tmp/result_baseline.jtl
kill $APP_PID
sleep $COOLDOWN

echo "=============================="
echo "场景2: 节点0满负载 + Java在节点1（分离）"
echo "=============================="
numactl --cpunodebind=0 --membind=0 stress-ng --cpu 0 --cpu-load 100 --timeout ${STRESS_DURATION}s &
STRESS_PID=$!
numactl --cpunodebind=1 --membind=1 $JAVA_APP &
APP_PID=$!
sleep 20
$JMETER_BIN -n -t montecarlo-test.jmx -l /tmp/result_separate.jtl
kill $APP_PID $STRESS_PID
sleep $COOLDOWN

echo "=============================="
echo "场景3: 节点0满负载 + Java也在节点0且远端内存（最差）"
echo "=============================="
numactl --cpunodebind=0 --membind=0 stress-ng --cpu 0 --cpu-load 100 --timeout ${STRESS_DURATION}s &
STRESS_PID=$!
numactl --cpunodebind=0 --membind=1 $JAVA_APP &   # CPU在0，内存在1
APP_PID=$!
sleep 20
$JMETER_BIN -n -t montecarlo-test.jmx -l /tmp/result_remote_mem.jtl
kill $APP_PID $STRESS_PID

echo "所有场景完成。结果在 /tmp/result_*.jtl，停顿日志在 /tmp/jhiccup.hlog"
```

执行：

```bash
chmod +x run_montecarlo_test.sh
./run_montecarlo_test.sh
```

---

第五阶段：结果分析

1. 提取 JMeter 聚合报告（P99 延迟）

JMeter 的 .jtl 文件是 CSV，可直接用脚本计算百分位，或使用 JMeter 插件。如果安装了 jmeter-plugins-manager，可以生成报告：

```bash
$JMETER_HOME/bin/JMeterPluginsCMD.sh --generate-csv /tmp/p99_report.csv \
  --input-jtl /tmp/result_baseline.jtl --plugin-type AggregateReport
```

若无插件，用以下 awk 一行命令提取 P99：

```bash
awk -F, 'NR>1{print $2}' /tmp/result_baseline.jtl | sort -n | awk '{all[NR]=$0} END{print all[int(NR*0.99)]}'
```

依次计算三个场景的 P99 延迟，填入表格：

场景 P99 延迟 (ms) 描述
基线 (节点1独立) ? 无压力，NUMA 本地
分离 (节点0压力+节点1) ? 外部压力仅在远端，无 CPU 争抢
远端内存 (CPU0+MEM1) ? 直接争抢 CPU + 所有内存访问跨节点

正常情况下你会看到：基线 < 分离 < 远端内存，且因为蒙特卡洛的 5% 重请求存在，P99 的绝对值会比纯轻请求高很多，更能体现真实风险。

2. 解读 jHiccup 停顿日志

```bash
java -jar jHiccupLogProcessor-2.0.10.jar -i /tmp/jhiccup.hlog -o /tmp/jhiccup_results
```

查看输出目录下的 percentiles.csv，重点关注 99%ile 列在不同场景下的增加量，这反映了系统底层停顿（如 GC、调度、内存迁移）对业务延迟的贡献。

3. 验证内存分配（测试过程中另开终端）

```bash
watch -n 1 "numastat -p $(pgrep -f MonteCarloServer)"
```

确保场景 3 中 other_node 分配不断增加，证实远程内存压力。

---

第六阶段：进阶——不同负载混合的蒙特卡洛分布

如果想实验更多种分布（例如泊松到达的激增流量、对数正态的服务时间），可将 Java 服务中的随机逻辑替换：

```java
// 对数正态分布示例（需导入 java.util.concurrent.ThreadLocalRandom）
double serviceTime = Math.exp(ThreadLocalRandom.current().nextGaussian() * 0.5 + 2.0) * 10; // 单位 ms
// 通过循环或 sleep 模拟该时长
long end = System.currentTimeMillis() + (long)serviceTime;
while (System.currentTimeMillis() < end) { /* busy wait */ }
```

JMeter 端可以通过 __Random(1,100,)} 配合 If Controller 来实现更复杂的路由规则。

---

最终交付物清单

· 蒙特卡洛 Java 应用源码
· JMeter 测试计划 JMX 文件
· 一键执行脚本
· 三个场景的 P99 延迟对比表
· jHiccup 解析出的 GC/停顿百分位报告
