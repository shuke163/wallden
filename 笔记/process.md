# process
-------------
>进程管理脚本
-------------
```
#!/bin/bash 
# ------------------------------------------------------------------------------------
# Copyright (C) 2015 Archosaur Studio, Zulong Entertainment Inc. All Rights Reserved.
# Version: 0.9.3
# ------------------------------------------------------------------------------------

usage()
{ 
  echo "Usage: $0 {start|stop|restart|status} [lineid]" 
} 

suffix=_stable
LINELIST="1 2 3 4 5 6"
LINK_NUM=3
UNAMED_ACTIVE=0
PROGRAME_ACTIVE=0

start_line()
{
	IDLIST=$LINELIST
	if [ ! -z $1 ]; then
		IDLIST=$1
	fi
	SELFID=`id -u`
	cd gamed
	for LN in $IDLIST
	do
		PIDS=`pgrep -u $SELFID -f "./gs$suffix gs.conf gmserver.conf config/gsalias$LN.conf"`
		if [ -z "$PIDS" ]; then
			echo start line $LN
			nohup setsid ./gs$suffix gs.conf gmserver.conf config/gsalias$LN.conf &>../logs/game$LN.log &
			#nohup setsid ./gs_asan$suffix gs.conf gmserver.conf config/gsalias$LN.conf  &>../logs/game$LN.log &
		else
			echo line $LN is already running
		fi
		sleep 1
	done
	cd ..
}



status() 
{
	PROGRAME_ACTIVE=0;
	for LN in $LINELIST
	do
		check_line $LN
		PROGRAME_ACTIVE=`expr $PROGRAME_ACTIVE + $?`;
	done
	check_process zlogd 
	PROGRAME_ACTIVE=`expr $PROGRAME_ACTIVE + $?`;
	check_process gdeliveryd
	PROGRAME_ACTIVE=`expr $PROGRAME_ACTIVE + $?`;
	check_process gamedbd
	PROGRAME_ACTIVE=`expr $PROGRAME_ACTIVE + $?`;
	# check_process gonlineinfod
	# PROGRAME_ACTIVE=`expr $PROGRAME_ACTIVE + $?`;
	if [ $UNAMED_ACTIVE -eq 1 ]; then
		check_process unamed
		PROGRAME_ACTIVE=`expr $PROGRAME_ACTIVE + $?`;
	fi
	check_process glinkd
	PROGRAME_ACTIVE=`expr $PROGRAME_ACTIVE + $?`;
	return $PROGRAME_ACTIVE;
}

check_process()
{
	SELFID=`id -u`
	PIDS=`pgrep -u $SELFID -f "./$1$suffix.*$1.conf"`
	if [ ! -z "$PIDS" ]; then
		echo service $1 is running, pid $PIDS
		return 1;
	else
		echo service $1 is stopped.
		return 0;
	fi
}

exists()
{
	SELFID=`id -u`
	PIDS=`pgrep -u $SELFID -f "./$1$suffix.*$1.conf"`
	if [ ! -z "$PIDS" ]; then
		echo service $1 is running, pid $PIDS
		return 1
	fi
	return 0
}

kill_process()
{
	NAME=$1
	SIG=$2
	SELFID=`id -u`
	PIDS=`pgrep -u $SELFID -f "./$NAME$suffix.*$NAME.conf"`
	if [ -n "$PIDS" ]; then
		for PID in $PIDS
		do
			kill -$SIG $PID
			echo service $NAME pid $PID is killed.
		done
	else
		echo service $NAME is not running.
	fi
}

check_line()
{
	SELFID=`id -u`
	LN=$1
	PIDS=`pgrep -u $SELFID -f "./gs$suffix gs.conf gmserver.conf config/gsalias$LN.conf"`
	if [ ! -z "$PIDS" ]; then
		echo line $LN is running, pid $PIDS
		return 1;
	else
		echo line $LN is stopped.
		return 0;
	fi
}

kill_line()
{
	SELFID=`id -u`
	LN=$1
	PIDS=`pgrep -u $SELFID -f "./gs$suffix gs.conf gmserver.conf config/gsalias$LN.conf"`
	if [ -n "$PIDS" ]; then
		for PID in $PIDS
		do
			kill -9 $PID
			echo line $LN pid $PID is killed.
		done
	else
		echo line $LN is not running.
	fi
}

kill_grc()
{
	kill -9  $(ps -ef|grep grc.jar$suffix |grep -v grep|awk '{print $2}')
}

start_grc()
{
	cd grc
	rm -rf ./lib
	unzip lib.zip
	rm -rf grc.jar$suffix
	cp grc.jar grc.jar$suffix
	nohup java -jar ./grc.jar$suffix  -Xms1G -Xmx2G >>/dev/null  2>>/tmp/grc.err$suffix &
	cd ..
}

start()
{
	mkdir -p logs
        export LD_LIBRARY_PATH=.:../../lib
	ulimit -c unlimited
	ulimit -n 4096

	if [ ! -z $1 ]; then
                start_line $1
		return
	fi

	exists zlogd
	if [ $? -eq 0 ]; then
		cd zlogd
		echo 'start zlogd service'
		nohup setsid ./zlogd$suffix zlogd.conf &>/dev/null &
		sleep 2
		cd ..
	fi

	exists gdeliveryd
	if [ $? -eq 0 ]; then
		cd gdeliveryd
		echo 'start DS service'
		nohup setsid ./gdeliveryd$suffix gdeliveryd.conf &>../logs/gdelivery.log  &
		cd ..
	fi

	exists glinkd
	if [ $? -eq 0 ]; then
		cd glinkd
		echo 'start LS service'
		nohup setsid ./glinkd$suffix --ccs glinkd.conf &>../logs/glinkd.ccs.log  &
		for ((i = 1; i <= $LINK_NUM; i++))
		do
			nohup setsid ./glinkd$suffix --cls -i $i glinkd.conf &>../logs/glinkd.cls$i.log  &
		done
		cd ..
	fi

	exists gamedbd
	if [ $? -eq 0 ]; then
		cd gamedbd
		echo 'start DB service'
		nohup setsid ./gamedbd$suffix gamedbd.conf &>../logs/gamedbd.log &
		cd ..
	fi

	if [ $UNAMED_ACTIVE -eq 1 ]; then
		exists unamed
		if [ $? -eq 0 ]; then
			cd unamed
			echo 'start unamed service'
			nohup setsid ./unamed$suffix unamed.conf &>../logs/unamed.log &
			cd ..
		fi
	fi

#	exists gonlineinfod
#	if [ $? -eq 0 ]; then
#		cd gonlineinfod
#		echo 'start OL service'
#		nohup setsid ./gonlineinfod$suffix gonlineinfod.conf &>../logs/gonlineinfod.log  &
#		cd ..
#	fi

	#start_grc

	start_line
}

stop()
{
	if [ ! -z $1 ]; then
                kill_line $1
		return
	fi
	#kill_grc
	kill_process glinkd KILL
	kill_process gdeliveryd KILL
	kill_process gamedbd USR1
	#kill_process gonlineinfod KILL
	if [ $UNAMED_ACTIVE -eq 1 ]; then
		kill_process unamed USR1
	fi
        for LN in $LINELIST
        do
                kill_line $LN
        done
	kill_process zlogd KILL
} 

wait()
{
	for (( i=0; i<5; i++))
	do
		echo -n "."
		sleep 0.2
	done
	echo "."
}

restart()
{
        stop $1
	if [ ! -z $1 ]; then
		wait
	else
		PROGRAME_ACTIVE=1;
		while [ "$PROGRAME_ACTIVE" != "0" ]
		do
			wait
			status
			echo $PROGRAME_ACTIVE program is running
		done
	fi
        start $1
	wait
	echo 'task accomplished'
} 

case $1 in 
start) 
        start $2
        ;; 
stop)
        stop $2
        ;; 
restart)
        restart $2
        ;; 
status)
        status 
        ;; 
*) 
        usage 
        ;;
esac 
```