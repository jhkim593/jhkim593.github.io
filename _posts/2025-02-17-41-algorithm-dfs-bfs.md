---
layout: post
title: "알고리즘 - DFS, BFS 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
hidden: true
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


<br>

<details>
<summary><strong>벽 부수고 이동하기</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/2206)

<br>
### 난이도 : ⭐⭐⭐

방문 배열을 **3차원 배열**로 생성해서 벽을 뚫었을 떄와 안뚫었을 때를 각각 체크한다.
boolean[i][j][k] visit 배열을 생성했으며 k가 0일 때는 벽을 뚫지 않은 상태 , 1일 때는 뚫은 상태를 의미한다.

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    static int[] tx = {0 , 0,1,-1};
    static int[] ty = {1, -1,0,0};
    static Queue<int[]>que;
    static boolean[][][] visit;
    static char[][] arr;
    static int n,m;
    static int answer = -1;
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        n = Integer.parseInt(stz.nextToken());
        m = Integer.parseInt(stz.nextToken());

        que = new ArrayDeque<>();
        visit = new boolean[2][n][m];
        arr = new char[n][m];
        for(int i=0; i<n; i++){
            arr[i] = br.readLine().toCharArray();
        }

        que.add(new int[]{0,0,0,1});
        visit[0][0][0] = true;

        bfs();
        System.out.println(answer);
    }
    public static void bfs(){
        while(!que.isEmpty()){
            int[] temp = que.poll();
            int x = temp[0];
            int y = temp[1];
            int crush = temp[2];
            int count = temp[3];

            if(x == n-1 && y == m-1){
                answer = count;
                break;
            }

            for(int i=0; i<4; i++){
                int rx = x+tx[i];
                int ry = y+ty[i];

                if(rx>=0 && ry>=0 && rx<n && ry <m){
                    if(!visit[crush][rx][ry]){
                        if(arr[rx][ry]=='0') {
                            visit[crush][rx][ry] = true;
                            que.add(new int[]{rx,ry,crush,count+1});
                        }

                        if(arr[rx][ry]=='1' && crush == 0){
                            visit[1][rx][ry] = true;
                            que.add(new int[]{rx,ry,crush+1,count+1});
                        }
                    }
                }
            }
        }
    }
}
```
</div>
</details>

<br>

<details>
<summary>상범빌딩</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/6593)

<br>
### 난이도 : ⭐⭐

### 코드
```java
import java.io.*;
import java.util.*;


public class Main {
    static Queue<int[]> que;
    static char[][][] arr;
    static int [][][] check;
    static int[] tx = {1,0,0,-1,0,0};
    static int[] ty = {0,1,0,0,-1,0};
    static int[] tz = {0,0,1,0,0,-1};
    static int row,col,hei;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz;
        StringBuilder sb = new StringBuilder();
        while(true){
            stz = new StringTokenizer(br.readLine());
            hei = Integer.parseInt(stz.nextToken());
            row = Integer.parseInt(stz.nextToken());
            col = Integer.parseInt(stz.nextToken());

            if(hei == 0 && row == 0 && col ==0) break;

            arr = new char[row][col][hei];
            que = new ArrayDeque<>();
            check = new int[row][col][hei];

            for(int i =0; i<hei; i++){
                for(int j=0; j<row;j++){
                    stz = new StringTokenizer(br.readLine());

                    char[] temp = stz.nextToken().toCharArray();
                    for(int k=0; k<temp.length;k++){
                        arr[j][k][i] = temp[k];
                        if(temp[k]=='S') {
                            que.add(new int[]{j,k,i,0});
                            check[j][k][i] = 1;
                        }

                    }
                }
                stz = new StringTokenizer(br.readLine());
            }

            int answer = bfs();
            if(answer ==-1) {
                sb.append("Trapped!\n");
            } else{
                sb.append("Escaped in ");
                sb.append(answer);
                sb.append(" minute(s).\n");
            }
        }
        System.out.println(sb);
    }
    public static int bfs(){
        while(!que.isEmpty()){
            int[] temp = que.poll();
            int x = temp[0];
            int y = temp[1];
            int z = temp[2];
            int count = temp[3];

            if(arr[x][y][z] == 'E'){
                return count;
            }

            for(int i=0; i<6; i++){
                int rx = x+tx[i];
                int ry = y+ty[i];
                int rz = z+tz[i];

                if(rx>=0 && ry>=0 && rz>=0 && rx<row && ry<col && rz<hei){
                    if(check[rx][ry][rz] == 1) continue;
                    if(arr[rx][ry][rz] == '#') continue;
                    check[rx][ry][rz] = 1;
                    que.add(new int[]{rx,ry,rz,count+1});

                }
            }
        }
        return -1;
    }
}
```
</div>
</details>

<br>

<details>
<summary>신기한 소수</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/2023)

<br>
### 난이도 : ⭐


### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static Queue<Integer> que = new ArrayDeque<>();
    static int length;
    static StringBuilder sb;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        length = (int)(Math.pow(10,Integer.parseInt(stz.nextToken())-1));

        sb = new StringBuilder();

        que.add(2);
        que.add(3);
        que.add(5);
        que.add(7);

        while(!que.isEmpty()){
            int num = que.poll();

            if(num/length > 1 && num/length < 10){
                sb.append(num).append("\n");
                continue;
            }

            for(int i =0;i<=9;i++){
                int newNum = num*10+i;
                if(!check(newNum)) continue;
                que.add(newNum);
            }
        }
        System.out.println(sb);
    }

    public static boolean check(int num){
        for(int i=2; i*i<=num; i++){
            if(num % i == 0) return false;
        }
        return true;
    }
}
```
</div>
</details>


<br>

<details>
<summary>두 동전</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/16197)

<br>
### 난이도 : ⭐⭐

두 동전 위치를 한번에 저장하기 위해 4차원 배열을 생성해서 방문 체크를 수행함


### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static Queue<int[]> que = new ArrayDeque<>();
    static int row;
    static int col;
    static char[][]arr;
    static int[] tx = {1,-1,0,0};
    static int[] ty = {0,0,1,-1};
    static Map<String,Integer> map = new HashMap<>();
    static int[][][][] visit;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        row = Integer.parseInt(stz.nextToken());
        col = Integer.parseInt(stz.nextToken());

        arr = new char[row][col];

        List<int[]> list = new ArrayList<>();
        visit  = new int[row][col][row][col];
        for(int i=0; i< row; i++){
            stz = new StringTokenizer(br.readLine());
            char [] temp = stz.nextToken().toCharArray();
            for(int j=0; j< col; j++){
                arr[i][j] = temp[j];
                if(temp[j] == 'o'){
                    list.add(new int[]{i,j});
                }
            }
        }
        int sx1 = list.get(0)[0];
        int sy1 = list.get(0)[1];
        int sx2 = list.get(1)[0];
        int sy2 = list.get(1)[1];

        que.add(new int[]{sx1,sy1,sx2,sy2,0});
        visit[sx1][sy1][sx2][sy2] = 1;

        System.out.println(bfs());

    }
    public static int bfs(){
        while(!que.isEmpty()){
            int []temp = que.poll();
            int x1 = temp[0];
            int y1 = temp[1];
            int x2 = temp[2];
            int y2 = temp[3];
            int count = temp[4];

            //10번 초과면
            if(count >= 10) return -1;

            for(int i=0; i<4; i++){
                int rx1 = x1+tx[i];
                int ry1 = y1+ty[i];
                int rx2 = x2+tx[i];
                int ry2 = y2+ty[i];

                boolean check1 = outCheck(rx1,ry1);
                boolean check2 = outCheck(rx2,ry2);

                //둘다 나갔을 때
                if(check1 && check2) continue;

                //하나만 나갔을 때
                if(!check1 && !check2){
                } else{
                    return count+1;
                }

                //방문 체크
                if(visit[rx1][ry1][rx2][ry2] == 1) continue;
                visit[rx1][ry1][rx2][ry2] = 1;

                int[] move1 = move(x1,y1,rx1,ry1);
                int[] move2 = move(x2, y2, rx2, ry2);
                que.add(new int[]{move1[0], move1[1],move2[0],move2[1],count+1});
            }
        }
        return -1;
    }
    public static int[] move(int x,int y , int rx, int ry){
        if(arr[rx][ry] == '#') return new int[]{x,y};
        return new int[]{rx,ry};
    }
    public static boolean outCheck(int x, int y){
        if(x < 0 || y <0 || x>= row || y >=col){
            return true;
        }
        return false;
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>경주로 건설</strong></summary>
<div markdown="1">

> [문제 링크](https://school.programmers.co.kr/learn/courses/30/lessons/67259?language=java)

<br>
### 난이도 : ⭐⭐

방향에 따라 비용이 달라지기 때문에 check[n][m][4] 3차원배열 생성해서 비용을 저장
만약 check 배열 요소보다 비용이 크다면 더 이상 탐색할 필요가 없기 때문에 탐색 종료


### 코드
```java
import java.util.*;
public class Solution {
    int[]tx={-1,0,0,1};
    int[]ty={0,1,-1,0};
    Queue<int[]> que;
    int row = 0;
    int col = 0;
    int[][] arr;
    int[][][] check;
    public int solution(int[][] board) {
        row = board.length;
        col = board[0].length;
        arr = board;
        check = new int[row][col][4];

        for(int i=0; i<row; i++){
            for(int j=0; j<col; j++){
                for(int k=0; k<4; k++){
                    check[i][j][k] = -1;
                }
            }
        }

        que = new ArrayDeque<>();
        que.add(new int[]{0,0,0,0});
        que.add(new int[]{0,0,0,1});
        que.add(new int[]{0,0,0,2});
        que.add(new int[]{0,0,0,3});

        check[0][0][0] = 0;
        check[0][0][1] = 0;
        check[0][0][2] = 0;
        check[0][0][3] = 0;

        bfs();

        int answer = Integer.MAX_VALUE;
        for(int i=0; i<4; i++){
            System.out.println(check[row-1][col-1][i]);
            if(answer > check[row-1][col-1][i]){
                if(check[row-1][col-1][i]  == -1) continue;
                answer = check[row-1][col-1][i];
            }
        }

        return answer;
    }
    public void bfs(){
        while(!que.isEmpty()){
            int[] temp = que.poll();
            int curRow = temp[0];
            int curCol = temp[1];
            int cost = temp[2];
            int dir = temp[3];

            if(curRow == row-1 && curCol == col-1) continue;
            for(int i=0; i<4; i++){
                int newRow = tx[i] + curRow;
                int newCol = ty[i] + curCol;

                if(newRow <0 || newCol <0 || newRow>=row || newCol >= col) continue;
                if(arr[newRow][newCol] == 1) continue;

                int newCost = 0;
                if(dir == i) newCost = cost+100;
                else newCost = cost+500+100;

                if(check[newRow][newCol][dir] != -1 && check[newRow][newCol][dir] < newCost) continue;

                check[newRow][newCol][dir] = newCost;
                que.add(new int[]{newRow, newCol, newCost, i});

            }
        }
    }
}
```
</div>
</details>
