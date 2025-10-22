---
title: 数据清洗、迁移（基于go）
date: 2025-10-22 21:04:11
categories: Essay
---

[源码地址](https://github.com/Breeze1203/grpc_study/tree/main/migrate)
```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"runtime"
	"sort"
	"sync/atomic"

	"strings"
	"sync"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

// DSN (Data Source Name) 占位符
const (
	SourceDB_DSN = "root:bwzn!@#123@tcp(127.0.0.1:3306)/jproduct_share" // 库 A
	TargetDB_DSN = "root:bwzn!@#123@tcp(127.0.0.1:3306)/ruoyi-vue-pro"  // 库 B
)

// RowData 是我们在流水线中传递的数据类型 使用 map[string]interface{} 来保持抽象，键是列名
type RowData map[string]interface{}

// TransformFunc 是转换函数的类型 它接收一行数据，返回转换后的数据。如果返回 nil, 则该行被跳过。
type TransformFunc func(row RowData) (RowData, error)

// Migrator 是迁移任务的配置器
type Migrator struct {
	sourceDB             *sql.DB
	targetDB             *sql.DB
	sourceTable          string
	targetTable          string
	batchSize            int               // 批量插入的大小
	transformer          TransformFunc     // 数据转换函数
	transformConcurrency int               // 转换阶段的并发数
	columnMapping        map[string]string // {"sourceCol": "targetCol"}
	sourceColumns        []string          // ORDERED list of cols to SELECT
	targetColumns        []string
}

func NewMigrator(sourceDSN, targetDSN, sourceTable, targetTable string, batchSize int, transformConcurrency int, transformer TransformFunc, mapping map[string]string) (*Migrator, error) {
	sdb, err := sql.Open("mysql", sourceDSN)
	if err != nil {
		return nil, fmt.Errorf("连接源数据库失败: %w", err)
	}
	if err := sdb.Ping(); err != nil {
		return nil, fmt.Errorf("Ping 源数据库失败: %w", err)
	}
	sdb.SetConnMaxLifetime(time.Minute * 3)
	sdb.SetMaxOpenConns(10)
	sdb.SetMaxIdleConns(10)
	tdb, err := sql.Open("mysql", targetDSN)
	if err != nil {
		return nil, fmt.Errorf("连接目标数据库失败: %w", err)
	}
	if err := tdb.Ping(); err != nil {
		return nil, fmt.Errorf("Ping 目标数据库失败: %w", err)
	}
	tdb.SetConnMaxLifetime(time.Minute * 3)
	tdb.SetMaxOpenConns(10)
	tdb.SetMaxIdleConns(10)
	sourceCols := make([]string, 0, len(mapping))
	targetCols := make([]string, 0, len(mapping))

	// 我们必须对 key (源列) 进行排序，以保证 SELECT 和 INSERT 的顺序一致
	keys := make([]string, 0, len(mapping))
	for k := range mapping {
		keys = append(keys, k)
	}
	sort.Strings(keys) // 排序
	// 按照排好序的 key 来构建 source 和 target 列表
	for _, srcCol := range keys {
		sourceCols = append(sourceCols, srcCol)
		targetCols = append(targetCols, mapping[srcCol])
	}
	return &Migrator{
		sourceDB:             sdb,
		targetDB:             tdb,
		sourceTable:          sourceTable,
		targetTable:          targetTable,
		batchSize:            batchSize,
		transformer:          transformer,
		transformConcurrency: transformConcurrency,
		// 赋值新字段
		columnMapping: mapping,
		sourceColumns: sourceCols,
		targetColumns: targetCols,
	}, nil
}

// Close 关闭数据库连接
func (m *Migrator) Close() {
	m.sourceDB.Close()
	m.targetDB.Close()
}

// getColumns 是一个辅助函数，用于从指定数据库和表获取列名
func (m *Migrator) getColumns(db *sql.DB, tableName string) ([]string, error) {
	query := fmt.Sprintf("SELECT * FROM %s LIMIT 0", tableName)
	rows, err := db.Query(query)
	if err != nil {
		return nil, fmt.Errorf("getColumns (for table: %s) 查询失败: %w", tableName, err)
	}
	defer rows.Close()

	cols, err := rows.Columns()
	if err != nil {
		return nil, fmt.Errorf("getColumns (for table: %s) rows.Columns() 失败: %w", tableName, err)
	}
	return cols, nil
}

func (m *Migrator) verifyColumnMapping() error {
	log.Println("正在验证列映射关系...")
	// 验证源表
	sourceDBCols, err := m.getColumns(m.sourceDB, m.sourceTable)
	if err != nil {
		return err
	}
	sourceDBSet := make(map[string]struct{}, len(sourceDBCols))
	for _, col := range sourceDBCols {
		sourceDBSet[col] = struct{}{}
	}
	// 检查 mapping 中定义的 sourceColumns 是否都存在于源表中
	for _, col := range m.sourceColumns {
		if _, ok := sourceDBSet[col]; !ok {
			return fmt.Errorf("映射错误: 源列 '%s' 在源表 '%s' 中未找到", col, m.sourceTable)
		}
	}
	// 验证目标表
	targetDBCols, err := m.getColumns(m.targetDB, m.targetTable)
	if err != nil {
		return err
	}
	targetDBSet := make(map[string]struct{}, len(targetDBCols))
	for _, col := range targetDBCols {
		targetDBSet[col] = struct{}{}
	}
	// 检查 mapping 中定义的 targetColumns 是否都存在于目标表中
	for _, col := range m.targetColumns {
		if _, ok := targetDBSet[col]; !ok {
			return fmt.Errorf("映射错误: 目标列 '%s' 在目标表 '%s' 中未找到", col, m.targetTable)
		}
	}
	log.Println("列映射关系验证成功.")
	log.Printf("将从源表 SELECT (A库): %v", m.sourceColumns)
	log.Printf("将 INSERT 进目标表 (B库): %v", m.targetColumns)
	return nil
}

// Run 启动迁移流水线
func (m *Migrator) Run() error {
	startTime := time.Now()
	// 获取列名
	if err := m.verifyColumnMapping(); err != nil {
		return err
	}
	// 创建 channels
	extractChan := make(chan RowData, m.batchSize*2) // 提取 channel
	loadChan := make(chan RowData, m.batchSize*2)    // 加载 channel
	errChan := make(chan error, 3)                   // 错误 channel
	var wg sync.WaitGroup
	// 启动流水线
	wg.Add(3)
	go m.extract(&wg, extractChan, errChan)    // 启动 Extract
	go m.transform(&wg, extractChan, loadChan) // 启动 Transform
	go m.load(&wg, loadChan, errChan)          // 启动 Load
	// 等待所有 goroutines 完成
	wg.Wait()
	close(errChan)
	// 检查错误
	for err := range errChan {
		if err != nil {
			return err // 返回第一个遇到的错误
		}
	}
	log.Printf("迁移完成! 总耗时: %v\n", time.Since(startTime))
	return nil
}

// 从源表读取数据，并发送到 extractChan
func (m *Migrator) extract(wg *sync.WaitGroup, extractChan chan<- RowData, errChan chan<- error) {
	defer wg.Done()
	defer close(extractChan)
	query := fmt.Sprintf("SELECT %s FROM %s", strings.Join(m.sourceColumns, ", "), m.sourceTable)

	log.Println("[Extract] 正在向源数据库发送查询...")
	rows, err := m.sourceDB.Query(query)
	if err != nil {
		errChan <- fmt.Errorf("[Extract] 查询失败: %w", err)
		return
	}
	defer rows.Close()
	log.Println("[Extract] 查询已发送, 正在等待第一行数据...")
	values := make([]interface{}, len(m.sourceColumns))
	scanArgs := make([]interface{}, len(m.sourceColumns))
	for i := range values {
		scanArgs[i] = &values[i]
	}
	totalCount := 0
	for rows.Next() {
		if totalCount == 0 {
			log.Println("[Extract] 已收到第一行, 开始流式读取...")
		}
		if err := rows.Scan(scanArgs...); err != nil {
			errChan <- fmt.Errorf("[Extract] 扫描行失败: %w", err)
			return
		}
		rowData := make(RowData, len(m.sourceColumns))
		for i, colName := range m.sourceColumns {
			if b, ok := values[i].([]byte); ok {
				rowData[colName] = string(b)
			} else {
				rowData[colName] = values[i]
			}
		}
		extractChan <- rowData
		totalCount++

		if totalCount%10000 == 0 {
			log.Printf("[Extract] 已读取 %d 行\n", totalCount)
		}
	}

	if err := rows.Err(); err != nil {
		errChan <- fmt.Errorf("[Extract] rows.Err: %w", err)
	}

	log.Printf("[Extract] 读取完成, 总计 %d 行.\n", totalCount)
}

// 从 extractChan 读取数据，应用转换函数，发送到 loadChan
func (m *Migrator) transform(wg *sync.WaitGroup, extractChan <-chan RowData, loadChan chan<- RowData) {
	defer wg.Done()
	// 创建一个新的 WaitGroup 来管理 workers
	var workerWG sync.WaitGroup
	// 确定并发数
	concurrency := m.transformConcurrency
	var processedCount atomic.Uint64
	if concurrency <= 0 {
		concurrency = runtime.NumCPU() // 如果没设置，默认为 CPU 核心数
	}
	log.Printf("[Transform] 启动 %d 个并发转换 worker...", concurrency)
	// 启动 'concurrency' 个 worker
	for i := 0; i < concurrency; i++ {
		workerWG.Add(1)
		workerID := i + 1
		go func(id int) {
			defer workerWG.Done()
			log.Printf("[Transform-Worker %d] 已启动, 正在等待数据...", id)
			// 每个 worker 都从同一个 extractChan 竞争数据
			for row := range extractChan {
				transformedRow, err := m.transformer(row)
				if err != nil {
					log.Printf("[Transform] 转换失败，跳过该行: %v (Row: %v)\n", err, row)
					continue
				}
				if transformedRow == nil {
					// transform 函数返回 nil，表示跳过该行
					continue
				}
				// 将处理完的数据放入 loadChan
				loadChan <- transformedRow
				count := processedCount.Add(1)
				if count%10000 == 0 {
					log.Printf("[Transform] 已转换 %d 行\n", count)
				}
			}
		}(workerID)
	}
	// 等待 *所有* worker goroutine 执行完毕
	// (当 extractChan 关闭且所有数据被处理完, for range 循环会退出)
	workerWG.Wait()
	// 此时，所有数据都已转换并放入 loadChan， 安全地关闭 loadChan，通知 Load 阶段没有更多数据了。
	close(loadChan)
	log.Println("[Transform] 转换完成.")
}

// 从 loadChan 读取数据，批量写入目标数据库
func (m *Migrator) load(wg *sync.WaitGroup, loadChan <-chan RowData, errChan chan<- error) {
	defer wg.Done()
	colNames := strings.Join(m.targetColumns, ", ")
	placeholders := fmt.Sprintf("(%s)", strings.Repeat("?,", len(m.targetColumns)-1)+"?")
	valuesBuffer := make([]interface{}, 0, m.batchSize*len(m.targetColumns))
	rowsInBatch := 0
	totalLoaded := 0
	log.Println("[Load] 已启动, 正在等待转换后的数据...")
	for row := range loadChan {
		for _, colName := range m.targetColumns {
			valuesBuffer = append(valuesBuffer, row[colName])
		}
		rowsInBatch++
		if rowsInBatch >= m.batchSize {
			if err := m.executeBatch(colNames, placeholders, valuesBuffer, rowsInBatch); err != nil {
				errChan <- fmt.Errorf("[Load] 执行批量插入失败: %w", err)
				return
			}
			totalLoaded += rowsInBatch
			log.Printf("[Load] 成功插入 %d 行 (总计 %d)\n", rowsInBatch, totalLoaded)
			// 重置缓冲区
			valuesBuffer = valuesBuffer[:0]
			rowsInBatch = 0
		}
	}
	log.Println("[Load] loadChan 已关闭. 正在处理最后一批数据...")
	// 处理最后一批不足 batchSize 的数据
	if rowsInBatch > 0 {
		if err := m.executeBatch(colNames, placeholders, valuesBuffer, rowsInBatch); err != nil {
			errChan <- fmt.Errorf("[Load] 执行最后批量插入失败: %w", err)
			return
		}
		totalLoaded += rowsInBatch
		log.Printf("[Load] 成功插入最后 %d 行 (总计 %d)\n", rowsInBatch, totalLoaded)
	}
	log.Printf("[Load] 加载完成, 总计 %d 行.\n", totalLoaded)
}

// executeBatch 是 load 的辅助函数，用于执行批量插入
func (m *Migrator) executeBatch(colNames, placeholderTemplate string, values []interface{}, rowCount int) error {
	if rowCount == 0 {
		return nil
	}
	query := fmt.Sprintf("INSERT INTO %s (%s) VALUES %s",
		m.targetTable,
		colNames,
		strings.Repeat(placeholderTemplate+",", rowCount-1)+placeholderTemplate,
	)
	log.Println("[Load-Batch] 正在开启事务 (Begin)...")
	tx, err := m.targetDB.Begin()
	if err != nil {
		return err
	}
	log.Printf("[Load-Batch] 正在执行 %d 行的 INSERT... (如果卡在这里, 就是 B 库被锁了!)", rowCount)
	_, err = tx.Exec(query, values...)
	if err != nil {
		log.Printf("[Load-Batch] 批量 INSERT 失败! 错误: %v\n", err)
		log.Println("[Load-Batch] 正在回滚 (Rollback)...")
		tx.Rollback() // 尝试回滚
		return err    // 返回执行错误
	}

	log.Println("[Load-Batch] 正在提交事务 (Commit)...")
	if err := tx.Commit(); err != nil {
		log.Println("[Load-Batch] 事务 Commit 失败...")
		return err // 返回提交错误
	}
	return nil
}

// =================================================================
// MAIN - 任务配置
// =================================================================
func main() {
	log.Println("启动迁移任务...")
	// --- 定义转化规则 ---
	// Key: 源表 (A) 列名, Value: 目标表 (B) 列名
	columnMapping := map[string]string{
		"customer_id":             "customer_id",
		"customer_name":           "name",
		"customer_unifiedcredit":  "customer_unifiedcredit",
		"customer_addr":           "detail_address",
		"customer_code":           "code",
		"customer_business":       "customer_business", //新增
		"customer_linkman":        "customer_linkman",  //新增
		"customer_phone":          "telephone",
		"customer_wecat":          "wechat",
		"status":                  "transform_status",
		"salesman_id":             "owner_user_id",
		"salesman_name":           "salesman_name",         //新增
		"salesman_departmentid":   "salesman_departmentid", //新增
		"salesman_department":     "salesman_department",   // 新增
		"assign_userid":           "assign_user_id",        // 新增
		"assign_username":         "assign_username",       //新增
		"assign_time":             "assign_time",           //新增
		"remark":                  "remark",
		"is_del":                  "deleted",
		"customer_email":          "email",
		"customer_type":           "type",
		"customer_source":         "source",
		"requirement_description": "requirement_description",
		"customer_position":       "position",
		"customer_purpose":        "purpose",     //新增
		"participant":             "participant", //新增
		"follow _time_new":        "follow _time_new",
		"link_info":               "link_info", // 新增
		"create_time":             "create_time",
		"follow_status":           "follow_up_status",
		"count_down":              "count_down",        // 新增
		"invoce":                  "invoce",            //新增
		"customer_industry":       "customer_industry", //新增
	}
	// --- 定义转换函数 ---
	mappingTransform := func(row RowData) (RowData, error) {
		newRow := make(RowData, len(row))
		for sourceKey, value := range row {
			// 从 mapping 中查找对应的目标 key
			targetKey, ok := columnMapping[sourceKey]
			if !ok {
				return nil, fmt.Errorf("Transformer: key '%s' 未在 columnMapping 中定义", sourceKey)
			}
			newRow[targetKey] = value
		}
		return newRow, nil
	}
	log.Println("--- 正在执行任务 : tb_customer_saleslead -> crm_clue ---")
	migrator1, err := NewMigrator(
		SourceDB_DSN,
		TargetDB_DSN,
		"tb_customer_saleslead", // 源表
		"crm_clue",              // 目标表
		1000,                    // 批量大小
		8,                       // 并发数
		mappingTransform,        // “映射”转换器
		columnMapping,           // 传入映射表
	)
	if err != nil {
		log.Fatalln("创建迁移器 1 失败:", err)
	}
	println(migrator1.transformer)
	if err := migrator1.Run(); err != nil {
		log.Fatalln("迁移任务 1 失败:", err)
	}
	migrator1.Close()
	log.Println("--- 任务 1 完成 ---")
}
```
