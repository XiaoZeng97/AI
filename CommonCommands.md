- log into the server:
```
ssh parameter@202.120.38.60 -p 21443
```
password:sjtuwin

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
