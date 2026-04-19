# Requirement: vv 支持 "-p <prompt>" 单步执行命令

vv 支持 "-p <prompt>" 单步执行命令, 执行完退出程序。

即用户可以通过命令行传入 `-p` 参数指定一个 prompt，vv 直接执行该 prompt 对应的任务，执行完成后自动退出程序，而不是进入交互式终端模式。
