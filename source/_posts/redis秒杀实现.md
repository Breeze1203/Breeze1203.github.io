---
title: redis秒杀实现
date: 2025-12-04 20:47:08
categories: Essay
---
秒杀（Seckill）的核心思想，可以用一句话概括：**“挡在外面，漏斗过滤，逐层削峰。”**

如果把普通的电商购物比作“在超市排队结账”，那么秒杀就是**“丧尸围城”**。几百万人瞬间冲向只有 100 个商品的柜台，如果你开门让他们全冲进去，超市（数据库）瞬间会被踩平。

秒杀架构设计的**最高准则**不是“如何让所有人都抢到”，而是**“如何保证系统不被巨大的流量冲垮，活着把这 100 个商品卖出去”**。

这套思想主要体现在一个经典的**“漏斗模型”**上：

我们将流量一层层拦截，直到最后打到数据库时，只剩下极少数的请求。具体思想如下：

#### 1. 动静分离：让浏览器抗住 90% 的流量
在秒杀开始前，用户会疯狂刷新页面。
* **思想：** 凡是能不请求服务器的，坚决不请求。
* **手段：**
    * **CDN 加速：** 把秒杀页面的 HTML、CSS、JS、图片全部扔到 CDN 上。用户刷新页面时，请求的是离他最近的 CDN 节点，根本没打到你的服务器。
    * **按钮控制：** 秒杀还没开始时，前端按钮置灰。点击后，按钮强制置灰 3 秒，防止用户用“单身三十年的手速”疯狂点击（前端拦截无效请求）。

#### 2. 网关限流：把“凑热闹”的挡回去
请求发出来了，到了你的服务器大门口（Nginx / Gateway）。
* **思想：** 既然只能卖 100 个，放 100 万人进去也没用，放 500 人进去就够了。
* **手段：**
    * **IP 限流：** 同一个 IP 一秒钟请求 100 次？肯定是脚本机器人，直接封锁。
    * **令牌桶/漏桶算法：** 设置一个阀值，比如每秒只允许 1000 个请求通过，剩下的 99 万请求直接返回“哎呀，太拥挤了”，连业务代码都不跑，直接在门口劝退。

#### 3. 缓存抗压：Redis 才是主战场
通过了网关的请求，进入了业务层。
* **思想：** 数据库（MySQL）是娇贵的，千万不能让大量请求直接碰它。所有的扣减库存操作，必须在内存（Redis）里完成。
* **手段：**
    * **预热库存：** 活动开始前，把 `stock = 100` 写进 Redis。
    * **原子扣减：** 使用 Redis 的 `decr` 命令或者 Lua 脚本扣减库存。
        * 谁扣减成功（返回值 >= 0），谁就有资格进入下一步。
        * 谁扣减失败（返回值 < 0），直接返回“抢光了”，**不需要去查数据库**。
    * **这一步最为关键：** 这里挡住了绝大多数并发，真正能走到下一步的，只有那 100 个成功扣减了 Redis 库存的幸运儿。

#### 4. 削峰填谷：消息队列（MQ）做缓冲
那 100 个在 Redis 里抢到库存的人，需要生成订单、扣钱、写数据库。
* **思想：** 虽然只剩 100 个请求，但如果瞬间同时写数据库，数据库压力还是大。既然已经抢到了，稍微晚 1 秒生成订单也没关系。
* **手段：**
    * **异步处理：** 把这 100 个“中奖”的请求扔进 RabbitMQ / Kafka。
    * **慢慢消费：** 后台起几个消费者线程，匀速地从 MQ 里取消息，慢慢地写数据库、创建订单。
    * **用户体验：** 用户点了“抢购”后，界面显示“排队中...”，其实后台正在慢慢处理。

#### 5. 数据库兜底：事务的一致性
最后，只有极少数请求（几乎就是 1:1 的库存数）真正执行 SQL。
* **思想：** 这是最后一道防线，保证数据绝对正确，不错乱。
* **手段：**
    * 利用数据库的事务（Transaction）和行锁，做最终的订单落库。因为流量已经被前面几层漏斗削得微乎其微了，数据库完全扛得住。

---

#### 总结：秒杀的哲学

1.  **稳大于快：** 即使 1000 万人来抢，由于层层拦截，最后打到数据库的可能只有 100 个 QPS，系统稳如泰山。
2.  **读多写少：** 99.9% 的操作是“读”（看页面、查库存），只有 0.1% 是“写”（下单）。让缓存抗读，让 DB 抗写。
3.  **快速失败 (Fail Fast)：** 这是对你上一个问题的最好回答。对于抢不到的人，**直接、立刻、马上**告诉他“没抢到”，不要让他等待，不要消耗服务器资源去计算他的请求。

#### 思考题

在秒杀系统中，有一个非常可怕的风险叫做**“超链接暴露”**。如果懂技术的黄牛抓包拿到了你的秒杀 URL（比如 `/seckill/buy/1001`），他在活动开始前就用脚本疯狂请求这个 URL，怎么办？

**想知道如何通过“动态 URL”或“隐藏入口”来防御这种黄牛脚本吗？**
这是一个非常硬核且必须解决的安全问题。

如果你的秒杀接口是固定的，比如 `POST /api/seckill/buy/1001`，那么黄牛根本不需要打开你的网页。他们会在活动开始前 5 分钟写好脚本，死循环请求这个接口。结果就是：**你的活动还没开始，黄牛的请求已经在服务器门口排队了。**

为了防御这种“抢跑”和“脚本轰炸”，我们必须把 URL 变成动态的，让黄牛**“猜不到、拿不着”**。

这套防御体系通常分为 **3 个步骤** 来实现：

---

#### 1. 核心战术：接口地址动态化 (URL Randomization)

我们不能让秒杀接口是一个静态的字符串，它必须包含一个**“只有在活动开始那一刻才能拿到，且每个人都不一样”**的随机加密串（Token）。



#### 以前的流程（不安全）：
1.  用户点击“抢购”。
2.  前端直接请求：`POST /seckill/buy/{productId}`

#### 现在的流程（安全 - 两段式请求）：

**第一步：获取“秒杀令牌” (Get Token)**
只有到了秒杀时间（比如 12:00:00），前端才允许发送这个请求。
* 请求：`GET /seckill/path/{productId}`
* 服务器逻辑：
    1.  判断时间：到了 12:00 吗？没到就报错。
    2.  判断资格：用户登录了吗？有黑名单吗？
    3.  **生成随机串：** 比如生成一个 UUID `a1b2-c3d4...`，或者用 MD5 加密一下用户信息。
    4.  **存入 Redis：** Key=`seckill_path_{userId}_{productId}`, Value=`a1b2-c3d4...` (设置 60秒过期)。
    5.  **返回给前端：** 把这个随机串 `a1b2-c3d4...` 给前端。

**第二步：携带令牌去秒杀 (Do Seckill)**
前端拿到随机串后，拼接出真正的秒杀地址，立马发起第二次请求。
* 请求：`POST /seckill/buy/{productId}/{randomPath}`
    * 实际 URL 变成了：`/seckill/buy/1001/a1b2-c3d4...`
* 服务器逻辑：
    1.  去 Redis 查：`seckill_path_{userId}_{productId}` 是否存在？
    2.  比对：URL 里的随机串，和 Redis 里存的一样吗？
    3.  **如果一样：** 说明这是合法流程，放行，去扣库存。
    4.  **如果不一样/不存在：** 报错“非法请求”。

**防御效果：** 黄牛即使知道秒杀 ID 是 1001，但他不知道那个随机生成的 `a1b2-c3d4...` 是什么，所以他拼不出合法的 URL。

---

#### 2. 辅助战术：隐藏入口 (Hidden Entrance)

除了后端加密，前端页面也要配合，不能让黄牛直接从 HTML 源码里看到按钮的逻辑。

* **按钮置灰 + 倒计时：** 在 11:59:59 之前，按钮是 `disabled` 的。
* **JS 动态加载：** 秒杀的 JavaScript 逻辑不要直接写在页面里。
    * 到了 12:00，前端通过一个定时器去请求一个只有几 KB 的 js 文件（或者请求服务器获取当前状态）。
    * 只有这个请求成功了，才会把“抢购”按钮真正渲染出来，或者把点击事件绑定上去。
* **防止本地时间作弊：** 千万别用 `new Date()` (用户本地时间) 来判断是否开始。用户把电脑时间调快 1 分钟，按钮就亮了？
    * **必须以服务器时间为准。** 页面加载时获取一次服务器时间，然后客户端进行倒计时修正。

---

#### 3. 终极防御：图形验证码 (Captcha)

如果黄牛写了超级复杂的脚本，真的模拟了“获取 Token -> 拼接 URL”的全过程，怎么办？

**加一道验证码！** 这招虽然损用户体验，但防刷效果极好。

* **时机：** 在点击“抢购”瞬间，先弹出一个验证码（比如：简单的数学题 `12 + 5 = ?`，或者滑块验证）。
* **流程：**
    1.  用户填对验证码。
    2.  前端把 `验证码结果` 发给后端。
    3.  后端验证通过后，才给你下发那个 **动态 URL 的随机串**。
* **作用：**
    * **防脚本：** OCR 识别图片需要时间，这一耽误，几百毫秒就过去了，普通用户就有了机会。
    * **削峰：** 100 万人同时点按钮，大家都卡在填验证码上。有人填得快（2秒），有人填得慢（5秒）。**流量被自然地拉平了**，服务器压力瞬间减小。

---

#### 总结：一套组合拳

| 攻击手段 | 防御手段 | 原理 |
| :--- | :--- | :--- |
| **提前请求** | **服务器时间校验** | 时间没到，所有请求直接拒之门外。 |
| **暴力猜测 URL** | **动态 URL (Token)** | 只有先领了“号”（随机串），才能去排队。 |
| **脚本自动抢** | **图形验证码** | 强行拉低脚本的速度，把机器拉到和人同一水平线。 |
| **跳过前端 UI** | **隐藏 JS / 动态加载** | 让脚本找不到点击入口。 |


```go
// main.go 
package main

import (
	"context"
	"crypto/md5"
	"fmt"
	"log"
	"math/rand"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
)

var (
	ctx           = context.Background()
	rdb           *redis.Client
	seckillLuaSHA string
)

// ==================== 1. 核心终极无锁秒杀 Lua ====================
const SeckillLuaScript = `
local sold_key = KEYS[1]           -- seckill:sold:10001
local stock    = tonumber(ARGV[1])

local count = redis.call("INCR", sold_key)
if count <= stock then
    return 1
else
    redis.call("DECR", soldKey)
    return 0
end
`

// ==================== 2. 动态秒杀令牌 ====================
// 只有拿到令牌的请求才能进入秒杀接口
func genSeckillToken(goodsId int64, userId string) string {
	today := time.Now().Format("20060102")
	salt := "xAI2025_grok_seckill_!" // 服务端盐，绝对保密
	raw := fmt.Sprintf("%d_%s_%s_%d", goodsId, userId, today, rand.Int63())
	return fmt.Sprintf("%x", md5.Sum([]byte(raw+salt)))
}

func checkToken(goodsId int64, userId, token string) bool {
	expect := genSeckillToken(goodsId, userId)
	return token == expect
}

// ==================== 单 IP 每秒最多 200 次，超了直接拒绝 ====================
func rateLimit(ip string) bool {
	key := "rate:limit:" + ip
	count, _ := rdb.Incr(ctx, key).Result()
	if count == 1 {
		rdb.Expire(ctx, key, 1*time.Second)
	}
	return count <= 200 
}

// ==================== 4. 启动时加载 Lua ====================
func init() {
	rdb = redis.NewClient(&redis.Options{Addr: "127.0.0.1:6379"})
	sha, err := rdb.ScriptLoad(ctx, SeckillLuaScript).Result()
	if err != nil {
		panic("加载秒杀脚本失败: " + err.Error())
	}
	seckillLuaSHA = sha
	log.Println("终极秒杀系统启动成功 SHA =", seckillLuaSHA)
}

// ==================== 5. 秒杀接口 ====================
func Seckill(c *gin.Context) {
	goodsId := int64(10001)
	userId := c.GetHeader("X-User-Id")
	if userId == "" {
		userId = c.ClientIP()
	}

	// 1. 动态令牌校验（黄牛第一道门都进不来）
	token := c.Query("token")
	if !checkToken(goodsId, userId, token) {
		c.JSON(403, gin.H{"code": 1001, "msg": "无效的请求"})
		return
	}

	// 2. 全局 + IP 限流（第二道防线）
	if !rateLimit(c.ClientIP()) {
		c.JSON(403, gin.H{"code": 1002, "msg": "请求太快了，请稍后再试"})
		return
	}

	// 3. 设备/用户频率风控（1秒内同用户最多3次）
	userRateKey := fmt.Sprintf("rate:user:%d:%s", goodsId, userId)
	if rdb.Incr(ctx, userRateKey).Val() > 3 {
		rdb.Expire(ctx, userRateKey, 1*time.Second)
		c.JSON(403, gin.H{"code": 1003, "msg": "操作频繁，请稍后"})
		return
	}

	// 4. 获取实时库存
	stock := int64(100000) // 实际从 DB 或本地缓存拿
	if stock <= 0 {
		c.JSON(200, gin.H{"code": 1, "msg": "已售罄"})
		return
	}

	// 5. 终极原子秒杀（无锁计数器）
	soldKey := fmt.Sprintf("seckill:sold:%d", goodsId)
	result, _ := rdb.EvalSha(ctx, seckillLuaSHA,
		[]string{soldKey},
		stock,
	).Int()

	if result == 1 {
		// 防重复提交标记
		flagKey := fmt.Sprintf("seckill:flag:%d:%s", goodsId, userId)
		rdb.SetEx(ctx, flagKey, "1", 30*time.Minute)

		// 异步生成订单
		go asyncCreateOrder(goodsId, userId)

		c.JSON(200, gin.H{
			"code": 0,
			"msg":  "抢购成功！",
			"token_used": token[:8] + "...",
		})
		return
	}

	c.JSON(200, gin.H{"code": 1, "msg": "手慢了，已被抢光"})
}

func asyncCreateOrder(goodsId int64, userId string) {
	time.Sleep(50 * time.Millisecond)
	log.Printf("订单生成 → goods=%d user=%s", goodsId, userId)
	// 真实减 MySQL 库存 + 插入订单
}

func main() {
	r := gin.Default()

	// 获取动态令牌接口（前端提前调用）
	r.GET("/token/:goods_id", func(c *gin.Context) {
		goodsId, _ := strconv.ParseInt(c.Param("goods_id"), 10, 64)
		userId := c.GetHeader("X-User-Id")
		if userId == "" {
			userId = "guest"
		}
		token := genSeckillToken(goodsId, userId)
		c.JSON(200, gin.H{"token": token, "expire": "当天有效"})
	})

	r.GET("/seckill/:id", Seckill)

	fmt.Println("终极防黄牛秒杀系统已启动")
	fmt.Println("先访问 → http://localhost:8080/token/10001 获取 token")
	fmt.Println("再带 token 访问 → http://localhost:8080/seckill/10001")
	r.Run(":8080")
}
```

### 使用方法（完全模拟真实用户）

```bash
# 1. 先获取动态令牌（每天不同，用户不同）
curl http://localhost:8080/token/10001 -H "X-User-Id: xiaoming"

# 返回：
# {"token":"8a9f3c2d1e5b7f9...", "expire":"当天有效"}

# 2. 带 token 抢购
curl "http://localhost:8080/seckill/10001?token=8a9f3c2d1e5b7f9..."
```

### 这套系统黄牛 99.99% 死法

| 防御层          | 黄牛怎么死                                      |
|-----------------|------------------------------------------------|
| 动态令牌        | 抓不到算法，token 每天每人不一样的，死             |
| 全局 + IP 限流  | 几万个请求一秒打进来，直接 403                  |
| 用户频率限制    | 同个用户1秒点1000次，直接封                     |
| Lua 原子计数器  | 再快的脚本也超卖不了                            |
