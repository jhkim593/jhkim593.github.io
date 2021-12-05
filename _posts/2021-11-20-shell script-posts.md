---
layout: post
title: "shell script"
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

if [ ${string1} == ${string2}]; then
	#실행 문장 없으면 오류
	echo "hello1"

elif [${string1} == ${string3}] then
	echo "hello2"

fi

#조건절 작성시 띄어쓰기 주의 [ ${..} ]
if [ ${string1} == ${string2} ]|| [ ${string3} == ${string4} ] && [ ${string5} == ${string6} ] ;then
   echo "11"
fi
~~~

- if -z '문자열' : 문자열의 길이가 0이면 참  ex ) [-z string]
- if -n '문자열' : 문자열의 길이가 0이 아니면 참 [-n string]




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

nohup java -jar snapshot.jar

~~~


### Example2

~~~shell

~~~
