在使用的环境(比如我的环境：torch2.0)里安装jupyter，它会附带安装许许多多附包，比如`ipykernel`

```bash
pip install jupyter
```

将本地环境注入`jupyter`,执行`python -m ipykernel install --user --name 环境名字 --display-name jupyter显示的名字`

```
python -m ipykernel install --user --name torch2.0 --display-name torch2.0
```

然后就能用了