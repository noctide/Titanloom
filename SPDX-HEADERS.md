# 源文件头（SPDX 标记）

在每个源文件顶部加一行，便于工具识别，不必粘贴完整许可证文本。

Python / Shell / YAML：

```
# SPDX-License-Identifier: Apache-2.0
# Copyright 2026 Titanloom contributors
```

TypeScript / JavaScript：

```
// SPDX-License-Identifier: Apache-2.0
// Copyright 2026 Titanloom contributors
```

SQL：

```
-- SPDX-License-Identifier: Apache-2.0
-- Copyright 2026 Titanloom contributors
```

说明：
- LICENSE 文件保持官方文本原样，不要往里填写版权人；版权人写在 NOTICE 与文件头。
- 生成代码、第三方代码保留其原有头与许可证，不要改写。
- SPDX 标记不能代替权利与兼容性审查。
