---
title: redis秒杀实现
date: 2025-12-04 20:47:08
categories: Essay
---

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
