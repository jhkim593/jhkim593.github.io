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

 : ⭐⭐⭐

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

 : ⭐⭐

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

 : ⭐⭐

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

 : ⭐⭐⭐

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

 : ⭐⭐⭐

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

 : ⭐⭐

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

 : ⭐


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

 : ⭐⭐

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

 : ⭐⭐

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


<br>

<details>
<summary><strong>공주님을 구해라!</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/17836)

<br>

 : ⭐⭐

그람이 있을 때 없을 때 체크를 위해 3차원 배열 생성 이동할 때 마다 동일하게 1시간씩 늘어나기 있기때문에 공주 위치 도달 시 바로 시간을 return 했음


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


<br>

<details>
<summary><strong>움직이는 미로 탈출!</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/16954)

<br>

 : ⭐⭐⭐


### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static ArrayDeque<int[]> que = new ArrayDeque<>();
    static char[][] arr = new char[8][8];
    static int[][][] check = new int[8][8][8];
    static int[] tx = {0 ,0, 1,-1,1,1 ,-1,-1,0};
    static int[] ty = {1, -1,0,0,1,-1,1,-1,0};
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

        for(int i=0; i< 8; i++){
            StringTokenizer stz = new StringTokenizer(br.readLine());
            String str = stz.nextToken();
            for(int j=0; j<8; j++){
                arr[i][j] = str.charAt(j);
            }
        }
        que.add(new int[]{7,0,0});
        System.out.println(bfs());
    }
    public static int bfs(){
        while(!que.isEmpty()){
            int[] temp = que.poll();
            int x = temp[0];
            int y = temp[1];
            int time = temp[2];

            if(x == 0 && y==7){
                return 1;
            }

            int wall = time >= 8 ? 7 : time;

            for(int i=0; i<9 ; i++){
                int rx = x+tx[i];
                int ry = y+ty[i];

                if(i==7 && time >=8) continue;
                if(rx <0 || ry <0 || rx>=8 || ry>= 8) continue;
                if(rx -time>=0){
                    if(arr[rx-time][ry]=='#') continue;
                }
                if(rx -(time+1)>=0){
                    if(arr[rx-(time+1)][ry]=='#') continue;
                }
                if(check[rx][ry][wall] == 1) continue;
                check[rx][ry][wall] = 1;
                que.add(new int[]{rx,ry,time+1});

            }
        }
        return  0;

    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>ABCDE</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/13023)

<br>

 : ⭐⭐

방문 배열을 초기화해서 각 정점에 dfs를 실행함
### 코드
```java
import java.util.*;
import java.io.*;
class Main{

    static ArrayList<Integer>[] arr;
    static int answer = 0;
    static int[] visit;
    public static void main(String[]args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());
        int r = Integer.parseInt(stz.nextToken());

        arr = new ArrayList[n];
        visit = new int[n];
        for(int i=0; i<n; i++){
            arr[i] = new ArrayList<>();
        }

        for(int i=0; i<r; i++){
            stz = new StringTokenizer(br.readLine());
            int f1 = Integer.parseInt(stz.nextToken());
            int f2 = Integer.parseInt(stz.nextToken());

            arr[f1].add(f2);
            arr[f2].add(f1);
        }
        for(int i=0; i<n;i++){
            visit[i] = 1;
            dfs(i,1);
            visit[i] = 0;
            if(answer == 1){
                System.out.println(answer);
                return;
            }
        }
        System.out.println(answer);
    }
    public static void dfs(int idx, int count){
        if(count == 5) {
            answer = 1;
            return;
        }
        for(int f : arr[idx]){
            if(visit[f] == 1) continue;
            visit[f] = 1;
            dfs(f,count+1);
            visit[f] = 0;
        }
    }
}
```
</div>
</details>


<br>

<details>
<summary>N과M(1)</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/15649)

<br>

 : ⭐

방문 배열을 초기화해서 각 정점에 dfs를 실행함
### 코드
```java
import java.util.*;
import java.io.*;


class Main{
    static int n,m;
    static StringBuilder sb = new StringBuilder();
    static int [] arr;
    static int [] visit;
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        n = Integer.parseInt(stz.nextToken());
        m = Integer.parseInt(stz.nextToken());

        visit = new int[n+1];
        arr = new int[m];
        dfs(0);
        System.out.print(sb);
    }
    public static void dfs(int length){
        if(length == m ){
            for(int num : arr){
                sb.append(num).append(" ");
            }
            sb.append("\n");
        }
        else{
            for(int i=1; i<=n; i++){
                if(visit[i] == 1) continue;
                visit[i] = 1;
                arr[length] = i;
                dfs(length+1);
                visit[i] = 0;
            }
        }
    }

}
```
</div>
</details>


<br>

<details>
<summary>N과M(5)</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/15654)

<br>

 : ⭐

방문 배열을 초기화해서 각 정점에 dfs를 실행함
### 코드
```java
import java.util.*;
import java.io.*;


class Main{
    static int n,m;
    static StringBuilder sb = new StringBuilder();
    static int [] arr;
    static int [] visit;
    static int [] temp;
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        n = Integer.parseInt(stz.nextToken());
        m = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(br.readLine());
        visit = new int[n];
        temp = new int[m];
        arr = new int[n];
        for(int i=0; i<n; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }
        Arrays.sort(arr);
        dfs(0);
        System.out.println(sb);

    }
    public static void dfs(int length){
        if(length == m){
            for(int num : temp){
                sb.append(num).append(" ");
            }
            sb.append("\n");
        } else {
            for(int i=0; i<arr.length; i++){
                if(visit[i] == 1) continue;
                visit[i] = 1;
                temp[length] = arr[i];
                dfs(length+1);
                visit[i] = 0;
            }
        }
    }
}
```
</div>
</details>

<br>

<details>
<summary>부분수열의 합</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1182)

<br>

### 난이도 : ⭐⭐

부분집합을 구하기위해 dfs를 수행함    
만약 target이 0이면 공집합도 개수에 포함되기때문에 제외  

### 코드
```java
import java.io.*;
import java.util.*;

public class Main{
    static int count,target,answer;
    static int[] arr;
    public static void main(String[] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        count = Integer.parseInt(stz.nextToken());
        target = Integer.parseInt(stz.nextToken());

        arr = new int[count];
        stz = new StringTokenizer(br.readLine());
        for(int i=0; i<count; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }
        dfs(0,0);
        if(target == 0)answer--;
        System.out.println(answer);
    }
    public static void dfs(int idx, int sum){
        if(idx == count){
            if(sum == target) answer++;
            return;
        }
        dfs(idx+1,sum+arr[idx]);
        dfs(idx+1,sum);    
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>샘터</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/18513)

<br>

### 난이도 : ⭐⭐


### 코드
```java
import java.util.*;
import java.io.*;
public class Main {
  static Queue<int[]> que = new ArrayDeque<>();
  static long answer = 0;
  static int count = 0;
  static Map<Integer, Integer> map = new HashMap<>();
  public static void main(String[] args) throws IOException {
    BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
    StringTokenizer stz = new StringTokenizer(bf.readLine());

    int n = Integer.parseInt(stz.nextToken());
    int k = Integer.parseInt(stz.nextToken());

    stz = new StringTokenizer(bf.readLine());
    for(int i=0; i< n; i++){
      int num = Integer.parseInt(stz.nextToken());
      que.add(new int[]{num, 0});
      map.put(num,0);
    }

    loop:
    while(!que.isEmpty()){
      int []temp = que.poll();
      int cur =temp[0];
      int dis = temp[1];

      for(int i=0; i<2; i++){
        int next = cur;
        if(i==0){
          next +=1;
        } else{
          next -=1;
        }
        if(-100000000>next || next>100000000) continue;
        if(map.containsKey(next)) continue;

        map.put(next,0);
        count++;
        answer+=(dis+1);
        if(count == k) break loop;
        que.add(new int[]{next,dis+1});
      }
    }
    System.out.print(answer);
  }
}
```
</div>
</details>



<br>

<details>
<summary><strong>연산자 끼워넣기(3)</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/15659)

<br>

### 난이도 : ⭐⭐⭐
사칙연산 계산시 ArrayDeque를 활용
- * , /는 먼저 큐에 마지막으로 들어간 값과 계산
* 이후에 + , -를 계산

### 코드
```java
import java.util.*;
import java.io.*;
public class Main {
  static int []arr;
  static int []opr;
  static int max = Integer.MIN_VALUE;
  static int min = Integer.MAX_VALUE;
  static int n;
  static int []curOpr;
  public static void main(String[] args) throws IOException {
    BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
    StringTokenizer stz = new StringTokenizer(bf.readLine());

    n = Integer.parseInt(stz.nextToken());

    arr = new int[n];
    stz = new StringTokenizer(bf.readLine());
    for(int i=0; i<n; i++){
      arr[i] = Integer.parseInt(stz.nextToken());
    }

    opr = new int[4];
    stz = new StringTokenizer(bf.readLine());
    for(int i=0; i<4; i++){
      opr[i] = Integer.parseInt(stz.nextToken());
    }

    curOpr = new int[n-1];
    dfs(0);
    System.out.println(max);
    System.out.println(min);
  }
  public static void dfs(int idx){
    if(idx == n-1) {
      max = Math.max(cal(),max);
      min = Math.min(cal(),min);
      return;
    }
    for(int i=0; i<4; i++){
      if(opr[i] == 0) continue;

      curOpr[idx] = i+1;

      opr[i]--;
      dfs(idx+1);
      opr[i]++;
    }

  }
  public static int cal(){
    ArrayDeque<Integer>numQue = new ArrayDeque<>();
    ArrayDeque<Integer>oprQue = new ArrayDeque<>();

    numQue.add(arr[0]);
    for(int i=0; i<n-1; i++){
      if(curOpr[i] == 3){
        numQue.add((numQue.pollLast() * arr[i+1]));
      } else if(curOpr[i] == 4){
        numQue.add((numQue.pollLast() / arr[i+1]));
      } else {
        numQue.add(arr[i+1]);
        oprQue.add(curOpr[i]);
      }
    }
    int sum = numQue.poll();
    while(!oprQue.isEmpty()){
      int temp = oprQue.poll();
      if(temp == 1){
        sum +=numQue.poll();
      }
      else {
        sum-=numQue.poll();
      }
    }
    return sum;
  }
}

```
</div>
</details>


<br>

<details>
<summary><strong>암호 만들기</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1759)

<br>

### 난이도 : ⭐⭐

dfs 완전 탐색으로 해결  
char 배열인 answer를 정의해서 사용
### 코드
```java
import java.util.*;
import java.io.*;

public class Main {

  static int c,n;
  static char [] arr;
  static List<String> list = new ArrayList<>();
  static char[] answer;
  public static void main(String[] args) throws Exception {
    BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
    StringTokenizer stz = new StringTokenizer(br.readLine());

    c = Integer.parseInt(stz.nextToken());
    n = Integer.parseInt(stz.nextToken());

    stz = new StringTokenizer(br.readLine());
    arr = new char[n];
    answer = new char[c];
    for (int i=0; i<n; i++) {
      arr[i] = stz.nextToken().charAt(0);
    }
    Arrays.sort(arr);
    dfs(0,0);
    for (String str : list) {
      System.out.println(str);
    }
  }
  static void dfs(int idx, int count){
    if(count == c){
      String secret = new String(answer);
      if (check(secret)) {
        list.add(secret);
      }
      return;
    }
    for(int i=idx; i<n; i++){
      answer[count] = arr[i];
      dfs(i+1, count+1);
    }
  }
  static boolean check(String str){
    int count1 = 0;
    int count2 = 0;
    for (char c : str.toCharArray()) {
      if(c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') count1++;
      else count2++;
      if(count1 >= 1 && count2 >= 2) return true;
    }
    return false;
  }
}
```
</div>
</details>

