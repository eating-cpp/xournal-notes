# xournal-notes


# Xournal++ 多端同步（GitHub + Git）操作指南
## 概述
本文档记录 Xournal++ 笔记通过 **Git + GitHub 仓库**实现多端（Ubuntu/Windows/macOS）同步的完整操作流程，包含主设备初始化、其他设备同步配置、日常使用规范及常见问题解决，便于后续查阅复用。

## 一、通用前期准备（所有设备必做）
### 1. 安装 Git 并配置全局身份
Git 是版本控制工具，用于追踪笔记文件变更；配置身份是 Git 提交的必要条件，关联 GitHub 账号。
```bash
# 1. 安装 Git（不同系统方式）
# Ubuntu：终端执行
sudo apt install git -y
# Windows/macOS：从 Git 官网下载安装包，默认下一步安装即可

# 2. 配置全局用户名（替换为你的 GitHub 用户名）
git config --global user.name "你的GitHub用户名"
# 3. 配置全局邮箱（替换为你的 GitHub 绑定邮箱）
git config --global user.email "你的GitHub绑定邮箱"

# 4. 验证配置是否生效
git config --global --list
```

### 2. 配置 SSH 密钥（可选，推荐，避免重复输密码）
SSH 协议用于免密连接 GitHub 仓库，若不想配置，后续可改用 HTTPS 协议（需个人访问令牌）。
```bash
# 1. 生成 ED25519 类型 SSH 密钥（关联 GitHub 邮箱）
ssh-keygen -t ed25519 -C "你的GitHub绑定邮箱"
# 执行后一路回车（默认路径保存、空密码，无需额外配置）

# 2. 复制公钥内容（用于粘贴到 GitHub）
# Ubuntu/macOS：终端执行
cat ~/.ssh/id_ed25519.pub
# Windows：打开 C:\Users\你的用户名\.ssh\id_ed25519.pub，记事本复制所有内容

# 3. GitHub 绑定公钥（网页操作）
# ① 登录 GitHub → 点击右上角头像 → Settings → SSH and GPG keys → New SSH key
# ② Title：自定义（如「主设备-笔记本」「副设备-台式机」）
# ③ Key type：选择 Authentication key
# ④ Key：粘贴上述复制的公钥完整内容（开头 ssh-ed25519，结尾你的邮箱）
# ⑤ 点击 Add SSH key 保存

# 4. 测试 SSH 连通性（验证配置成功）
ssh -T git@github.com
# 若输出「Hi 你的用户名! You've successfully authenticated...」即为成功
```

## 二、主设备操作（首次初始化，仅需执行1次）
主设备：用于首次创建 GitHub 仓库并推送初始笔记，后续作为笔记同步的核心来源。
### 1. 新建 GitHub 远程仓库（网页操作）
1.  登录 GitHub → 点击右上角「+」→ New repository
2.  仓库名称：自定义（如 `xournal-notes`，建议英文）
3.  仓库类型：Private（私有仓库，避免笔记泄露）
4.  勾选「Add a README file」（初始化仓库结构）
5.  其他默认配置 → 点击「Create repository」创建
6.  复制仓库地址：仓库页面 → Code → SSH → 复制（格式：`git@github.com:你的用户名/xournal-notes.git`）

### 2. 本地笔记目录配置与 Git 初始化
```bash
# 1. 创建本地笔记目录（自定义路径，示例如下）
mkdir -p ~/xournal-notes  # Ubuntu/macOS（用户主目录下）
# mkdir D:\xournal-notes  # Windows（Git Bash 中执行，或手动创建文件夹）

# 2. 进入本地笔记目录
cd ~/xournal-notes  # Ubuntu/macOS
# cd D:\xournal-notes  # Windows

# 3. 初始化 Git 仓库
git init

# 4. 关联远程 GitHub 仓库（替换为你的仓库 SSH 地址）
git remote add origin git@github.com:你的用户名/xournal-notes.git

# 5. 拉取远程仓库的 README 文件（建立本地与远程的初始关联）
git pull origin main
```

### 3. 关联 Xournal++ 软件（指定笔记保存路径）
1.  打开主设备的 Xournal++
2.  依次点击：Edit（编辑）→ Preferences（首选项）→ Save & Export（保存和导出）
3.  在「Default Save Directory」（默认保存目录）中，选择上述创建的 `xournal-notes` 文件夹
4.  点击「OK」保存配置（后续新建/编辑的笔记会自动保存到该目录）

### 4. 首次提交并推送笔记到 GitHub
```bash
# 1. 进入本地笔记目录（若已在目录中，可跳过）
cd ~/xournal-notes

# 2. 添加所有笔记文件（. 表示所有文件，含.xopp笔记、README等）
git add .

# 3. 提交修改（备注清晰，便于追溯）
git commit -m "Initial commit: Xournal++ 初始笔记同步"

# 4. 推送本地分支到远程仓库（建立长期关联）
# 若提示分支名称不匹配（本地master，远程main），执行以下两种方案之一：
# 方案1：直接推送（快速生效，无需修改分支名）
git push -u origin master:main
# 方案2：重命名本地分支为main（规范统一，推荐）
git branch -m master main
git push -u origin main
```

### 5. 优化配置：添加 .gitignore 文件（忽略无关临时文件）
在 `xournal-notes` 目录下创建 `.gitignore` 文件，避免 Git 追踪临时文件/配置文件：
```bash
# 1. 创建并编辑 .gitignore 文件
nano ~/xournal-notes/.gitignore  # Ubuntu/macOS
# notepad D:\xournal-notes\.gitignore  # Windows

# 2. 粘贴以下内容（保存退出即可）
*.swp
*.tmp
.xournalpp/
.DS_Store
```
```bash
# 3. 提交 .gitignore 文件到远程仓库
git add .gitignore
git commit -m "Add .gitignore: 忽略无关临时文件"
git push origin main
```

## 三、其他设备操作（同步主设备笔记，每台副设备执行1次）
其他设备：用于拉取主设备推送的笔记，并将本地编辑的笔记同步到 GitHub，实现多端一致。
### 1. 克隆远程 GitHub 仓库到本地
```bash
# 克隆仓库（两种协议二选一，替换为你的仓库地址）
# 选项1：SSH 协议（已配置 SSH 密钥，推荐）
git clone git@github.com:你的用户名/xournal-notes.git ~/xournal-notes  # Ubuntu/macOS
# git clone git@github.com:你的用户名/xournal-notes.git D:\xournal-notes  # Windows

# 选项2：HTTPS 协议（未配置 SSH 密钥，备用，需个人访问令牌）
git clone https://github.com:你的用户名/xournal-notes.git ~/xournal-notes  # Ubuntu/macOS
# git clone https://github.com:你的用户名/xournal-notes.git D:\xournal-notes  # Windows
```

### 2. 关联 Xournal++ 软件（指定笔记保存路径）
操作与主设备一致，仅需将 Xournal++ 的默认保存路径指向克隆后的 `xournal-notes` 文件夹：
1.  打开当前设备的 Xournal++
2.  Edit → Preferences → Save & Export
3.  选择克隆后的 `xournal-notes` 文件夹 → OK 保存

### 3. 验证同步（查看主设备笔记是否存在）
克隆完成后，打开 `xournal-notes` 文件夹，若能看到主设备推送的 `.xopp` 笔记和 README 文件，说明克隆成功。

## 四、所有设备通用：日常同步流程（核心！每次使用必做）
为避免笔记冲突、确保多端一致，所有设备（主设备+其他设备）每次编辑笔记都需遵循以下流程：
### 1. 开始编辑前：拉取最新笔记（必做！）
打开终端/Git Bash，先拉取 GitHub 上的最新笔记（其他设备推送的更新）：
```bash
# 1. 进入笔记目录（替换为你的本地路径）
cd ~/xournal-notes  # Ubuntu/macOS
# cd D:\xournal-notes  # Windows

# 2. 拉取远程 main 分支的最新内容
git pull origin main
```

### 2. 编辑完成后：提交并推送更新（必做！）
笔记编辑并保存后，将本地修改同步到 GitHub，供其他设备拉取：
```bash
# 1. 进入笔记目录（若已在目录中，可跳过）
cd ~/xournal-notes

# 2. 添加所有修改/新增的文件（. 表示所有文件，也可指定单个文件）
git add .  # 全部添加
# git add 我的笔记.xopp  # 单个文件添加（按需使用）

# 3. 提交修改（备注清晰，建议包含日期和笔记内容）
git commit -m "Update: 2025-12-28 工作笔记补充 / 学习笔记修改"

# 4. 推送到 GitHub 远程仓库
git push origin main
```

### 3. 冲突处理（少见，按此操作）
若多台设备同时编辑同一个笔记并推送，拉取时会提示冲突，处理步骤：
1.  打开冲突的 `.xopp` 笔记文件，Xournal++ 会标记冲突位置，手动保留需要的内容（删除无用内容）
2.  保存文件后，执行以下命令完成同步：
```bash
cd ~/xournal-notes
git add .
git commit -m "Fix: 合并笔记冲突，保留最新有效内容"
git push origin main
```

## 五、常见问题与解决方案
### 1. SSH 认证失败：git@github.com: Permission denied (publickey)
-  原因：SSH 密钥未加载、公钥与 GitHub 不匹配、仓库地址错误
-  解决：
   1.  启动 SSH 代理并加载私钥：`eval "$(ssh-agent -s)"` → `ssh-add ~/.ssh/id_ed25519`
   2.  重新复制本地公钥（完整复制，无多余空格），重新绑定到 GitHub
   3.  验证仓库地址是否正确（用户名/仓库名无拼写错误）

### 2. 推送失败：error: 源引用规格 main 没有匹配
-  原因：本地分支名与远程分支名不匹配（本地master，远程main）、本地无有效提交
-  解决：
   1.  查看本地分支：`git branch`
   2.  分支重命名：`git branch -m master main` → 重新推送：`git push -u origin main`
   3.  若无分支：先执行 `git add .` → `git commit -m "备注"` 创建本地分支

### 3. 提交失败：作者身份未知
-  原因：未配置 Git 用户名/邮箱
-  解决：重新执行全局配置命令（见「通用前期准备」第1步）

### 4. 笔记不同步：本地编辑后 GitHub 无更新
-  原因：未执行 `git add + commit + push` 流程、Xournal++ 保存路径错误
-  解决：
   1.  确认 Xournal++ 保存路径指向 `xournal-notes` 目录
   2.  按「日常同步流程」执行提交和推送操作

## 六、补充说明
1.  若设备本地笔记目录丢失/损坏：直接重新执行「其他设备操作」中的克隆命令，即可恢复所有历史笔记
2.  移动端同步：Android 可安装 Termux + Git，克隆仓库后用 Xournal++ Mobile 打开 `.xopp` 文件
3.  自动同步（Ubuntu）：可编写 bash 脚本绑定快捷键，一键执行 pull + add + commit + push（按需使用）