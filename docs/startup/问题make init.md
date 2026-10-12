# 问题make init

---

### 问题make init

replace为什么不符合要求？

1. 目标模块 `github.com/tx7do/go-wind-toolkit/protoc-gen-typescript-http` 的 `go.mod` 含有指向本地相对路径的 `replace` 指令，
2. `Go 1.25+` 不允许这种模块被 `go install module@version` 方式安装

### 解决方案

#### 方案1：临时目录手动build

因为 `go install@version` 有这个限制，但是 `go build` 不受到影响。终端执行：

```shell
$tmpDir = "$env:TEMP\protoc-gen-build"
mkdir $tmpDir -ErrorAction SilentlyContinue
cd $tmpDir
go mod init protoc-build
go get github.com/tx7do/go-wind-toolkit/protoc-gen-typescript-http@latest
go build -o "$(go env GOPATH)\bin\protoc-gen-typescript-http.exe" github.com/tx7do/go-wind-toolkit/protoc-gen-typescript-http
```

#### 方案2：clone仓库本地安装

```shell
git clone https://github.com/tx7do/go-wind-toolkit.git
cd go-wind-toolkit\protoc-gen-typescript-http
go install .
```

#### 方案3：长期修复

1. 创建私人仓库

2. 获取上游代码，并初始化仓库

3. 仓库内修改：

   - 批量替换 `module path` 和 `import path`

   - 去掉所有的 `replace` 指令

   - 为每个子模块补充正确的版本依赖：原来被 `replace` 的那个依赖需要变成一个真实可拉取的版本（两种处理方式）

     - 方式1：进入每个有问题的子模块，执行`go mod tidy`自动处理

       ```shell
       cd protoc-gen-common
       go mod init github.com/<你的用户名>/my-wind-toolkit/protoc-gen-common
       go mod tidy
       ```

     - 方式2：更可控直接使用 `go mod edit` 修改版本号

       ```shell
       # 假设刚 push 上去的 commit hash 是 abc123def456，时间戳 20260930120000
       go mod edit -require=github.com/Hzn8Hub/teamtalk-tools/protoc-gen-common@v0.0.0-20260930120000-abc123def456
       go mod edit -dropreplace=github.com/Hzn8Hub/teamtalk-tools/protoc-gen-common
       ```

4. 推送仓库所有修改：

   ```shell
   cd D:\code\teamtalk-tools  # 回到仓库根目录
   git add -A
   git commit -m "chore: rename module paths and remove replace directives"
   git push origin main
   ```

5. 修改文件 `makefile`：

   ```makefile
   plugin:
       go install github.com/Hzn8Hub/teamtalk-tools/protoc-gen-go-redact@latest
       go install github.com/Hzn8Hub/teamtalk-tools/protoc-gen-typescript-http@latest
   cli:
       go install github.com/Hzn8Hub/teamtalk-tools/gowind/cmd/gow@latest
   ```

原理：Go multi-module monorepo 里，tag 必须是 `<子目录>/v<语义版本号>`的格式，Go 会根据 tag 前缀匹配到对应的子模块。比如

- `gowind/v0.0.1` 就对应目录 `gowind/go.mod`，
- `protoc-gen-common/v0.0.1` 对应 `protoc-gen-common/go.mod`，

1. 每个子模块（独立 go.mod）对应一个 `<子目录>/v<语义版本号>` 格式的 git tag。
2. 之前没 tag，Go 在远程找不到 `gowind/v0.0.0` 这个 revision。
3. 现在 7 个 tag 都打了，go mod tidy 应该能自动拉到对应的 pseudo-version 了。

```
cd D:\codeRepo\github\teamtalk\teamtalk-tools

git tag protoc-gen-common/v0.0.1
git tag protoc-gen-dart-http/v0.0.1
git tag protoc-gen-go-http/v0.0.1
git tag protoc-gen-go-redact/v0.0.1
git tag protoc-gen-typescript-http/v0.0.1
git tag gowind/v0.0.1
git tag gowind-uiapp/v0.0.1

git push origin --tags
```





