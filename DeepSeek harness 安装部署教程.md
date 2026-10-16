安装完node.js 之后再windows power shell
```
Set-ExecutionPolicy Restricted -Scope CurrentUser
```
npm config set registry https://registry.npmmirror.com  


一次性拉起
```
npx -y --registry=https://registry.npmmirror.com @deepseek-ai/dsh web
```
```
永久
npm install -g @deepseek-ai/dsh --registry=https://registry.npmmirror.com
```
npm cache clean --force  清除缓存命令

harness启动命令
```
dsh web
```