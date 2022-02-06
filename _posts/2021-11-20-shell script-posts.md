---
layout: post
title: "shell script(1)"
author: "jhkim593"
tags: Linux

---


### 함수 (function)
- function 생략가능
- 호출 코드가 함수 코드보다 반드시 뒤에 있어야함.

~~~shell
string_test(){
	echo "String test"
}

function string_test2() {
	echo "string test2"
	echo "인자값:"${@}
}

string_test "hello" "world"
~~~

결과로 인자값: hello world가 출력됩니다.

### 변수 (Variable)
- 변수 사용시에는 "="기호 앞뒤로 공백 없이 입력하면 대입연산자가 됨.
- 선언된 변수는 기본적으로 전역변수.
- 함수안에 지역변수 사용시 local을 붙여주면됨.
- 전역변수 export 붙여주면 자식 스크립트에서 사용가능.

~~~shell
#전역 변수 지정
string="hello"
echo ${string}

#지역 변수 테스트가
string_test(){
	#local 빼면 전역변수에 덮어 씌어짐
	local string ="local"
	echo ${string}
}

#지역 변수 테스트가
string_test

echo ${string}

~~~

<br>

### 반복문 (for ,while , untill)

~~~shell

for string in "hello1" "hello2" "..."; do
echo ${string}
done;

count=0
while [ ${count} -le 5]; do
	echo ${count}
	count=$(( ${count}+1 ))
done

#while 파일 값 읽기
#serverlist 값을 차례로 읽어서 SVR대입
while read SVR
do
        echo ${SVR}
done < serverlist


#수행 조건이 false 일때 실행됨
count2=10
until [ ${count2} -le 5 ]; do
    echo ${count2}
    count2=$(( ${count2}-1 ))
done

# 이중괄호 사용
for (( counter=10; counter>0; counter-- ))
do
echo -n "$counter "
done
~~~

이중 괄호가 산술연산 기능을 확장해주기 때문에 반복문에 사용 가능합니다.

### 산술 연산


~~~shell
a=10
b=20

sum=$(($a + $b))
echo $sum
~~~
괄호를 두번 사용하여 직접 더할 수있습니다.

<br>


~~~shell

sum=$(expr $a + $b)
~~~

산술 연산을 하는데 사용되는 expr명령을 이용할 수있습니다.




### 조건문 (if elif else fi)
- 실행 문장이 없을시 오류 발생.

~~~shell
string1="hello1"
string2="hello2"
string3="hello4"
string4="hello4"
string5="hello4"
string6="hello4"


  # 대괄호는 test 명령어를 의미하므로 공백이 필
if [ ${string1} == ${string2} ]; then
	#실행 문장 없으면 오류
	echo "hello1"

elif [${string1} == ${string3} ] then
	echo "hello2"

fi

#조건절 작성시 띄어쓰기 주의 [ ${..} ]
if [ ${string1} == ${string2} ]|| [ ${string3} == ${string4} ] && [ ${string5} == ${string6} ] ;then
   echo "11"
fi
~~~

- if -z '문자열' : 문자열의 길이가 0이면 참  ex ) [-z string]
- if -n '문자열' : 문자열의 길이가 0이 아니면 참 [-n string]
- && : 좌측 명령 /테스트의 결과가 참이면 우측 명령을 실행
- || : 좌측 명령 /테스트의 결과가 거짓이면 우측 명령 실행 (이미 참이기 때문에 실행 할 필요 없음)
- [] : test 명령어를 의미하며 괄호 다음 공백이 반드시 필요하다.

###조건문 (case)
~~~shell
CMD=$1

case "${CMD}" in

start)
        echo start;;
stop)
        echo stop;;
rerolad)
        echo reload;;

*)
        echo etc;;
esac

~~~



### 스크립트 절대 경로
쉘 스크립트를 실행할 때 스크립트 파일의 경로를 알아야 할 때가 있습니다. 리눅스 쉘의 실행 위치와 관련없이 쉘스크립트 파일의 절대경로를 얻는 방법에 대해서 알아보겠습니다. 쉘에서 sh명령어로 직접 스크립트를 실행하는 경우가 있으며, source명령어로 환경변수만 읽어오는 경우가 있습니다. 두가지 케이스에 대해서 다른 방법을 적용하여 절대경로를 얻었습니다.

<br>

- 스크립트 실행하는 쉘의 경로 얻기
쉘에서 pwd -P 명령어를 사용하면 현재 쉘의 절대경로를 알 수 있습니다. 이것을 스크립트에 적용해보면 아래처럼 사용할 수 있습니다.

~~~shell
#!/bin/sh
SHELL_PATH=`pwd -P`
echo $SHELL_PATH
~~~

이 방법으로 쉘스크립트 파일의 절대 경로를 얻을 수 없으며 스크립트를 실행하는 쉘 위치에 따라서 결과가 달라집니다.

- 스크립트 실행시 명령어를 인자로 받아 경로 얻기

~~~shell
DIR="$( cd "$( dirname "$0" )" && pwd -P )"
echo $DIR
~~~

$0은 명령어의 첫번째 인자를 뜻하고 dirname은 문자열에서 디렉토리만 추출해주는 명령어입니다.
스크립트 파일로 이동한 후 pwd를 통해 스크립트 파일의 절대 경로를 얻을 수 있습니다.


- readlink를 이용해 스크립트 절대경로 얻기
~~~shell
#/bin/bash
DIR="$(dirname $(readlink -f "$0"))"
ehco $DIR
~~~
readlink는 canonical file name을 얻는데 사용되는 커맨드 이며 Canonical Path는 어떤 파일의 경로를 나타내기 위한 유일한 심볼입니다.


### Example1
간단하게 ps, grep으로 pid를 조회하여 프로세스를 종료하는 script를 작성해 보겠습니다.

~~~shell

# ps -a 세션 리더(로그인 쉘)을 제외하고 터미널에 종속되지 않은 모든 프로세스 출력
# ps -e 커널 프로세스를 제외한 모든 프로세스 출력
# ps -f 풀 포맷으로 출력, PID와 같은 내용 추가
# grep "매칭되는 pattern이 존재하는 라인만"
# grep -v "매칭되는 Pattern이 존재하지 않는 라인만"
# awk {print $2} 두번째 필드만 출력

pid=$(ps -ef | grep snapshot.jar | grep -v "grep" | awk '{print $2}')

if [ "$pid" = "" ]; then
  echo " web application is not running."
else
  kill -9 $pid
  echo "web application proceess killed forcefully. (pid: $pid)"
fi

nohup java -jar snapshot.jar &

~~~

| (파이프라인): 파이프 라인 왼쪽의 명령결과 표준 스트림을 오른쪽 명령의 입력으로 사용한다.
; (세미 콜론) : 왼쪽의 명령이 끝난 후 이어서 세미콜론 오른쪽의 명령어를 실행한다.


### Example2

~~~shell
ds
~~~
