# Inspiration

## Guide

Powershell运行以下代码

```powershell
code $PROFILE
```

在profile文件中添加以下代码，并配置好ideaRepo路径

```powershell
function idea {
    param (
        [Parameter(Mandatory = $true, ValueFromRemainingArguments = $true)]
        [string[]]$ContentParts
    )

    # 合并剩余参数为单个字符串
    $content = $ContentParts -join " "

    # 使用您指定的路径
    $ideaRepo = "E:\repo\Inspiration"
    $ideaFile = Join-Path $ideaRepo "Idea.md"

    # 确保仓库目录存在
    if (-not (Test-Path -Path $ideaRepo)) {
        New-Item -ItemType Directory -Path $ideaRepo -Force | Out-Null
    }

    # 进入仓库目录
    Push-Location $ideaRepo

    $remotes = git remote 2>&1
    if ($remotes) {
        # 获取当前分支名称
        $currentBranch = git rev-parse --abbrev-ref HEAD
        
        # 先拉取最新更改（使用rebase）
        Write-Host "Pulling latest changes from remote..."
        git pull --rebase origin $currentBranch 2>&1 | Out-Null
    }

    # 添加时间戳和灵感内容到文件 (使用UTF-8无BOM编码)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "## [$timestamp]`n$content`n" | Out-File -FilePath $ideaFile -Append -Encoding utf8

    try {
        # 初始化 Git 仓库（如果尚未初始化）
        if (-not (Test-Path -Path ".git" -PathType Container)) {
            git init | Out-Null
            
            # 配置Git使用UTF-8
            git config core.quotepath false
            git config i18n.commitencoding utf-8
            git config i18n.logoutputencoding utf-8
            
            # 解决换行符问题
            git config core.autocrlf false
            git config core.safecrlf warn
            
            git add . | Out-Null
            git commit -m "Initial commit" | Out-Null
        }

        # 提交到本地仓库
        git add $ideaFile 2>&1 | Out-Null
        $commitMessage = "Add idea: $($content.Substring(0, [Math]::Min(30, $content.Length)))..."
        
        # 尝试提交
        git commit -m $commitMessage 2>&1 | Out-Null
        
        # 如果提交失败，尝试修改上次提交
        if ($LASTEXITCODE -ne 0) {
            git commit --amend --no-edit 2>&1 | Out-Null
        }
        
        # 检查远程仓库配置
        $remotes = git remote 2>&1
        if ($remotes) {
            # 获取当前分支名称（使用master分支）
            $currentBranch = git rev-parse --abbrev-ref HEAD
            
            # 推送到远程仓库（强制使用master分支）
            git push origin $currentBranch 2>&1 | Out-Null
            
            # 检查推送是否成功
            if ($LASTEXITCODE -eq 0) {
                Write-Host "[SUCCESS] Pushed to remote repository" -ForegroundColor Green
            } else {
                Write-Host "[WARNING] Failed to push to remote. Please check your remote configuration." -ForegroundColor Yellow
            }
        } else {
            Write-Host "[INFO] No remote repository configured. Changes committed locally only." -ForegroundColor Cyan
        }
    }
    catch {
        Write-Warning "Git operation error: $_"
    }
    finally {
        Pop-Location
    }

    # 使用纯ASCII字符输出结果
    Write-Host "[SUCCESS] Idea recorded" -ForegroundColor Green
    Write-Host "[FILE] Location: $ideaFile" -ForegroundColor Cyan
}
```




