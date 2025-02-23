---
layout: post
title: "알고리즘 - DFS, BFS 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
---

<details>
<summary><strong>N-Queen</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/9663)

<br>
### 난이도 : ⭐⭐⭐

### 코드

N*N 체스판에 퀸이 N개 들어가야하기 때문에 각 행마다 퀸은 반드시 들어간다.
index를 행 값을 열로 나타내는 1차원 배열을 생성했다.
arr[0] = 3일 때 (0+1)행 3열에 퀸이 존재함을 의미
```java
import java.util.*;
import java.io.*;

public class Main {
    static int[] arr;
    static int n;
    static int answer = 0;
    public static void main(String []args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        n = Integer.parseInt(stz.nextToken());

        arr = new int[n];
        dfs(0);
        System.out.println(answer);
    }
    public static void dfs(int cnt){
        if(cnt == n) {
            answer ++;
            return;
        }

        for(int i=0;i<n;i++){
            arr[cnt] = i;
            if(check(cnt)){
                dfs(cnt+1);                  
            }
        }
    }
    public static boolean check(int cnt){
        for(int i=0; i<cnt; i++){
            if(arr[i] == arr[cnt]) return false;
            if(Math.abs(cnt-i) == Math.abs(arr[cnt]-arr[i])) {
			return false;
		}
        }     
        return true;
    }

}
```
</div>
</details>

<br>

<details>
<summary><strong>이모티콘</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/14226)

<br>
### 난이도 : ⭐⭐

### 코드
시간의 최솟값을 구해야하 하기 때문에 DFS를 사용했으며
이모티콘 개수 , 버퍼 개수를 조합한 문자열을 Map 키로 저장해 중복 실행되지 않도록함
```java
import java.io.*;
import java.util.*;

public class Main {
    static int[] arr;
    static int s;
    static Queue<int[]> que = new LinkedList<>();
    static int answer = 0;
    static Map<String, String> map = new HashMap<>();
    public static void main(String[] args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        s = Integer.parseInt(stz.nextToken());

        que.add(new int[]{1,0,0});
        bfs();
        System.out.println(answer);

    }
    public static void bfs(){
        while(!que.isEmpty()){
            int []temp =que.poll();
            int num = temp[0];
            int count = temp[1];
            int buffer = temp[2];

            if(!checkKey(num,buffer)) continue;

            if(num == s) {
                answer = count;
                return;
            }

            for(int i=0; i<3; i++){
                if(i==0){
                    if(num > 0){
                        que.add(new int[]{num-1,count+1,buffer});
                    }
                } else if(i==1){
                    if(buffer > 0){
                        que.add(new int[]{num+buffer, count+1,buffer});
                    }
                } else if(i==2){
                    if(num > 0){
                        que.add(new int[]{num, count+1,num});
                    }
                }
            }
        }
    }
    public static boolean checkKey(int num, int buffer){
        String key = num+"/"+buffer;
        if(map.containsKey(key)) return false;
        map.put(key,"");
        return true;
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>아기상어2</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/17086)

<br>
### 난이도 : ⭐⭐

배열 크기가 최대 50*50이기 때문에 시간 문제는 없을 것같아 BFS 사용
배열 요소가 0인 경우 1이 발견될 때까지 BFS로 최소 거리 구함

### 코드
```java
import java.io.*;
import java.util.*;

public class Main {
    static int[] arr;
    static int s;
    static Queue<int[]> que = new LinkedList<>();
    static int answer = 0;
    static Map<String, String> map = new HashMap<>();
    public static void main(String[] args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        s = Integer.parseInt(stz.nextToken());

        que.add(new int[]{1,0,0});
        bfs();
        System.out.println(answer);

    }
    public static void bfs(){
        while(!que.isEmpty()){
            int []temp =que.poll();
            int num = temp[0];
            int count = temp[1];
            int buffer = temp[2];

            if(!checkKey(num,buffer)) continue;

            if(num == s) {
                answer = count;
                return;
            }

            for(int i=0; i<3; i++){
                if(i==0){
                    if(num > 0){
                        que.add(new int[]{num-1,count+1,buffer});
                    }
                } else if(i==1){
                    if(buffer > 0){
                        que.add(new int[]{num+buffer, count+1,buffer});
                    }
                } else if(i==2){
                    if(num > 0){
                        que.add(new int[]{num, count+1,num});
                    }
                }
            }
        }
    }
    public static boolean checkKey(int num, int buffer){
        String key = num+"/"+buffer;
        if(map.containsKey(key)) return false;
        map.put(key,"");
        return true;
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>숨바꼭질4</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/13913)

<br>
### 난이도 : ⭐⭐⭐

위치는 0이상 100,000이하 이며 이미 방문한 지점은 다시 방문할 필요가 없음.
history 배열을 생성해 history[i] i번째 이동하기 전 위치를 저장함.

**주의**
- String을 `+` 연산으로 이어 붙이게 되면 새로운 객체가 생성돼 메모리 사용 증가 , 성능 저하가 발생함을 간과  
- StringBuilder를 사용해 추가 객체 생성안되도록 `append` 사용
  - sb.append(s + " ")  (x)
  - sb.append(s).append(" ") (o) 

### 코드
```java
import java.io.*;
import java.util.*;

public class Main {
    static int s;
    static int e;
    static Queue<Integer> que = new LinkedList<>();
    static boolean [] visited = new boolean[100001];
    static int [] history = new int[100001];
    public static void main(String[] args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());
        Deque<Integer> deque = new ArrayDeque<>();
        s = Integer.parseInt(stz.nextToken());
        e = Integer.parseInt(stz.nextToken());

        Arrays.fill(history,-1);
        que.add(s);
        visited[s] = true;
        bfs();

        int count = 0;
        int key = e;
        deque.add(e);

        while(true){
            int value = history[key];
            if(value == -1) {
                count = deque.size() -1;
                break;
            }
            key = value;
            deque.add(value);
        }

        StringBuilder sb = new StringBuilder();
        while(!deque.isEmpty()){
            sb.append(deque.pollLast()).append(' ');
        }

        System.out.println(count);
        System.out.println(sb.toString());
    }
    public static void bfs(){
        while(!que.isEmpty()){
            int cur = que.poll();

            if(cur == e){
                return;
            }

            for(int i=0; i<3; i++){
                int newCur = 0;
                if(i==0){
                    newCur = cur+1;
                } else if(i==1){
                    newCur = cur-1;
                } else {
                    newCur = cur*2;
                }
                if(newCur >= 0 && newCur <=100000 && !visited[newCur]){
                    visited[newCur] = true;
                    history[newCur] = cur;
                    que.add(newCur);
                }
            }
        }
    }
}
```
</div>
</details>
