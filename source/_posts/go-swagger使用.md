---
title: go-swagger使用
date: 2021-12-13 14:31:25
tags:
---

## go swagger 自动生成文档

### 前言

- 在写完代码或者正在写代码的时候，如果要与前端进行同步开发文档是必须的，但是如果在写代码的时候发现接口要改，这个时候为了确保前端能够理解又要修改开发文档，万一忘记改了，可能无法面对前端的质问
- 这个时候一个实时同步的开发文档就很重要了
<!--more-->
## 学习使用

### 1.安装

```
COPY# 安装 swag 如果不行直接上二进制文件 (国内的连接问题 -- 直接用二进制文件)
go get -u github.com/swaggo/swag/cmd/swag

## 安装 gin-swagger 
go get github.com/swaggo/gin-swagger

## 安装  swaggerFiles
go get github.com/swaggo/gin-swagger/swaggerFiles
```

### 2. 编写注释

- 编写 main.go 中的注释

```
COPY// main.go 中添加注释


// @title 标题
// @version 1.0 (版本)
// @description 声明（可不写）
// @termsOfService https://www.test.com

// @contact.name www.test.com
// @contact.url https://www.test.com
// @contact.email me@test.me

// @license.name Apache 2.0 (必填)
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host 127.0.0.1:8080
// @BasePath 
```

- 注意
  @host 直接调试 API地址
  @BasePath 基础前缀路径
- 使用命令

```
COPY
swag init
# 或者
swag init --parseDependency --parseInternal --parseDepth 1 
```

- 在需要编写文档的 func 上

```
COPY type LoginReq{
	Code string `json:"code"` // 编码
 }
 
 // @Summary 获取 code
// @title 后台接口
// @Tags 登录
// @Router /wxapp/login [post]
// @param param body LoginReq true "用户请求参数"
// @Success 200 {object} JsonMsg
func Login(c *gin.Context) {
	// ...
}
```

- 然后再次运行 `swag init`

### 3. 运行程序 `go run main.go`

- 打开 http://localhost:8080/swagger/index.html

![go swagger 使用](https://cdn.learnku.com/uploads/images/202112/13/81167/hpUvBPbrCC.png!large)

- 补充
  - 如果需要查看 json 内容可以打开 http://localhost:8080/swagger/doc.json

## yapi + swagger 文档管理

### 1.打开 yapi 的一个项目至 数据管理

![go swagger 使用](https://cdn.learnku.com/uploads/images/202112/13/81167/Xsl2HiVfFa.png!large)

### 2.选择 项目中 docs/swagger.json

![go swagger 使用](https://cdn.learnku.com/uploads/images/202112/13/81167/7IZoIJCggK.png!large)

### 3. 将 swagger.json 放入 yapi Swagger数据导入则会自动导入构建好的文档中
