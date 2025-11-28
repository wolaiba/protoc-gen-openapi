# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

protoc-gen-openapi 是一个 protoc 插件，用于从 Protocol Buffer 定义生成 OpenAPI v3 文档。该项目 fork 自 google/gnostic，并添加了一些额外功能。

## Build & Test Commands

```bash
# 安装插件
go install

# 运行测试（测试会自动先执行 go install）
go clean -testcache && go test

# 使用插件生成 OpenAPI 文档
protoc sample.proto -I. --openapi_out=version=1.2.3:.

# 启用 validate 选项
protoc sample.proto -I. --openapi_out=version=1.2.3,validate=true:.

# 使用 build_tag 过滤
protoc sample.proto -I. --openapi_out=version=1.2.3,build_tag=postman:.
```

## Architecture

### 核心组件

- **main.go**: 插件入口，解析命令行参数并调用生成器
- **generator/openapi-v3.go**: 主要生成逻辑，包含 `OpenAPIv3Generator` 结构体
- **generator/reflector.go**: 处理 protobuf 类型到 OpenAPI schema 的映射
- **generator/validate.go**: 处理 protoc-gen-validate 注解转换为 OpenAPI 验证规则
- **generator/enum.go**: 枚举类型处理
- **openapi/annotations.pb.go**: 自定义 OpenAPI 扩展注解定义

### 配置参数

通过 `--openapi_out=` 传递的参数：
- `version`: API 版本号
- `title`: API 标题
- `description`: API 描述
- `naming`: 命名约定 (`json` 或 `proto`)
- `fq_schema_naming`: 是否使用完全限定的 schema 名称
- `enum_type`: 枚举序列化类型 (`integer` 或 `string`)
- `depth`: 循环消息的递归深度
- `default_response`: 是否添加默认错误响应
- `validate`: 是否解析 protoc-gen-validate 注解
- `build_tag`: 构建标签过滤

### 测试结构

测试位于 `plugin_test.go`，使用 `examples/tests/` 目录下的 proto 文件：
- 每个测试用例有一个目录，包含 `message.proto`、`openapi.yaml`（proto naming）和 `openapi_json.yaml`（json naming）
- 测试通过运行 protoc 生成 openapi.yaml 并与预期文件进行 diff 比较

### 扩展功能

1. **Summary Field**: 注释中使用 `|` 分隔 summary 和 description
2. **Validation**: 支持部分 protoc-gen-validate 注解
3. **Field Behaviors**: 支持 google.api.field_behavior (REQUIRED, OUTPUT_ONLY, INPUT_ONLY)
4. **Custom Headers**: 通过 `openapi.service_params` 和 `openapi.method_params` 添加自定义头
5. **Build Tags**: 用于条件生成方法或参数
