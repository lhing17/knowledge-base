tmux可以实现断开SSH连接后，继续在服务器上工作

## 创建tmux会话
tmux new -s XXX

## 进入tmux会话
tmux attach -t XXX

## 断开tmux会话
Ctrl+b, d

## 查看tmux会话
tmux ls

## 删除tmux会话
tmux kill-session -t XXX