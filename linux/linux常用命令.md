# 查看端口占用进程

    1、lsof -i:port
        lsof 是 list of open file 的简称

        COMMAND   PID USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
        uwsgi   10170  xx    xx  IPv4 397113343      0t0  TCP *:port (LISTEN)
        uwsgi   10175  xx    xx  IPv4 397113343      0t0  TCP *:port (LISTEN)

    2、 netstat -tunlp|grep port
        tcp  0 0 0.0.0.0:port   0.0.0.0:*    LISTEN    pid/xx uW 

