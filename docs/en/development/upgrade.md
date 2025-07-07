---
created: "2025-2-11"
title: 升级 gitlab 版本
---

## 前提条件

1.  已经构建出新版本的镜像
2.  已经完成 chart 的升级，并通过了集成测试

## 升级步骤

1. 替换工具版本：将工具上一个版本替换为当前版本
2. 重新生成 submodule:

```bash
make update-submodule
```

3. 提交代码构建 operator 镜像测试
