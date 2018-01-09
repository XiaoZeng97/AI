- log into the server:
```
ssh parameter@202.120.38.60 -p 21443
```
password:******

---


- enter the virtual environment:
```
source ~/TEST/bin/activate
```
- create a new virtual environment:
```
virtualenv ENV_NAME
```
- leave the current virtual environment:
```
deactivate
```

---
these commands should be run on your **own** terminals(local)
- upload local files into the server:
```
scp -P 21443 path/filename parameter@202.120.38.60:~/target/
```

- download files from the server:
```
scp -P 21443 parameter@202.120.38.60:~/target/targetfile path/filename
```

---

- install with Luarocks:
```
luarocks install xxx --local
```
- if the installation is failed, check if it is already installed:
```
luarocks list
```

---

- deal with out of memory:
```
CUDA_VISIBLE_DEVICES=GPU_ID
```

eg. the GPU_ID can be set to 2

---

- analyze using gogui
```
gogui-twogtp -analyze AvsB.dat
w3m AvsB.html
```

change "AvsB" to the sgf filename that you set for the match
use w3m here to view the html page in the terminal
can press Q then y to exit w3m

---

- 用服务器上的gtp通信比赛(服务器)
```
gogui-server -port 40002（随便选） -timeout 50（随便填） -verbos -loop "gtp运行文件路径"
```

- 用服务器上的gtp通信比赛(本地)
```
BLACK="gogui-client -timeout 5（随便填） 202.120.38.60 40002（对应端口）
WHITE="gogui-client -timeout 5（随便填） 202.120.38.60 40003
在gogui的bin的目录下 gogui -program "gogui-twogtp -black \"$BLACK\" -white \"$WHITE\" -time 5"
```
