# Requirement: Edit Tool Improvement

## Reference
参考 `doc/claude-code-prd/tools/file-edit.md` 中的规则对 `vage/tool/edit/edit_tool.go` 进行改进。

## Reference Document Key Rules

Based on `doc/claude-code-prd/tools/file-edit.md`:

1. **编辑前必须先用 Read 工具读取过文件** — 安全特性
2. **old_string !== new_string 验证** — 已实现
3. **拒绝规则检查** — 需要增强
4. **文件大小限制** — MAX_EDIT_FILE_SIZE: 1 GiB (当前仅 1MB)
5. **权限检查** — 需要增强
6. **UNC 路径安全阻止** — 需要增加
7. **如果 old_string 在文件中不唯一,编辑会失败** — 已实现
8. **replace_all 可在文件中重命名变量等** — 已实现

## Target File
`vage/tool/edit/edit_tool.go`

## Improvement Areas
- Increase default max file size constant to align with reference (1 GiB)
- Add UNC path blocking for security
- Enhance error messages to be more informative
- Ensure all safety features from the reference are implemented
