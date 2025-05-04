---
layout: post
title: "알고리즘 - 위상 정렬 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
hidden: true
---

<details>
<summary><strong>Strahler 순서</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/9470)

<br>
### 난이도 : ⭐⭐

위상 정렬(Topological Sorting) 을 사용해 해결
순서대로 동작시키기 위해 이미 탐색한 간선은 제거한다. 간선이 제거돼 루트 노드가 됐다면 큐에 삽입한다.
ex) 1 -> 3  , 2- > 3 일 때 해당 간선을 방문하면 삭제해서 1 -> 3 , 2 -> 3이 모두 탐색됐을 떄만 3을 큐에 삽입한다.


### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static Queue<Integer> que;
    static Node[] nodes;
    static int[] degree;
    static ArrayList<Integer>[] graph;
    static StringBuilder sb = new StringBuilder();
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int count = Integer.parseInt(stz.nextToken());
        for(int i=0; i<count; i++){
            stz = new StringTokenizer(br.readLine());
            int t = Integer.parseInt(stz.nextToken());
            int n = Integer.parseInt(stz.nextToken());
            int p = Integer.parseInt(stz.nextToken());
            degree = new int[n+1];
            graph = new ArrayList[n+1];
            que = new ArrayDeque<>();

            nodes = new Node[n+1];

            for(int j=1; j<=n; j++) graph[j] = new ArrayList<>();
            for(int j=0; j<p; j++){
                stz = new StringTokenizer(br.readLine());
                int start = Integer.parseInt(stz.nextToken());
                int end = Integer.parseInt(stz.nextToken());
                degree[end]++;
                graph[start].add(end);
            }


            for(int j=1; j<=n;j++){
                if(degree[j] != 0) continue;
                nodes[j] = new Node(1, 1);
                que.add(j);
            }

            int answer = 0 ;
            while(!que.isEmpty()){
                int num = que.poll();
                int curs = nodes[num].getOrder();
                answer = Math.max(answer,curs);

                if(num == n) break;


                List<Integer> ends = graph[num];
                for(int end : ends){
                    degree[end]--;
                    if(degree[end]==0) que.add(end);

                    if(nodes[end] == null){
                        nodes[end] = new Node(1,curs);
                        continue;
                    }

                    Node node = nodes[end];
                    if(node.max == curs) {
                        node.maxCount++;
                    }
                    if(node.max < curs){
                        node.maxCount = 1;
                        node.max = curs;
                    }
                }
            }
            sb.append(t).append(" ").append(answer).append("\n");
        }
        System.out.println(sb);
    }
    public static class Node{
        public Integer maxCount;
        public int max;

        public Node(int maxCount, int max){
            this.maxCount = maxCount;
            this.max = max;
        }
        public int getOrder(){
            if(maxCount >= 2) return max+1;
            return max;
        }
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>ACM Craft</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1005)

<br>
### 난이도 : ⭐⭐

각 노드별 건설 시간 저장을 위해 time 배열을 생성
`time[num] = Math.max(time[num] , time[curn]+ node[num]);` 을 통해 건설 시간 최소값을 저장
예를 들어  
1 (10 초) -> 3 (20 초)    
2 (5 초) -> 3 (20 초) 일 때 3은 건설하기 위한 최소 시간은 30초임


### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static Queue<Integer> que;
    static int[] node;
    static int[] degree;
    static int[] time;
    static ArrayList<Integer>[] graph;
    static StringBuilder sb = new StringBuilder();
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());
        for(int j =0; j < n; j++){
           stz = new StringTokenizer(br.readLine());

        int nc = Integer.parseInt(stz.nextToken());
        int pc = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(br.readLine());
        graph = new ArrayList[nc+1];
        degree = new int[nc+1];
        node = new int[nc+1];
        time = new int[nc+1];
        for(int i=1; i<= nc; i++){
            node[i] = Integer.parseInt(stz.nextToken());
            graph[i] = new ArrayList<>();
        }

        for(int i=0; i<pc; i++){
            stz = new StringTokenizer(br.readLine());

            int start = Integer.parseInt(stz.nextToken());
            int end = Integer.parseInt(stz.nextToken());
            graph[start].add(end);
            degree[end]++;
        }

        stz = new StringTokenizer(br.readLine());
        int target = Integer.parseInt(stz.nextToken());


        que = new ArrayDeque<>();

        for(int i=1; i<= nc; i++){
            if(degree[i] == 0) {
                que.add(i);
                time[i] = node[i];
            }
        }

       while(!que.isEmpty()){
           int curn = que.poll();

           if(curn == target){
               sb.append(time[curn]).append("\n");
               break;    
           }

           for(int num : graph[curn]){
               time[num] = Math.max(time[num] , time[curn]+ node[num]);

               degree[num]--;
               if(degree[num] == 0) {
                   que.add(num);
               }
           }   
       }
        }
        System.out.print(sb);
    }
}
```
</div>
</details>

<br>

<details>
<summary><strong>선수 과목</strong></summary>
<div markdown="1">

> [문제 링크]("https://www.acmicpc.net/problem/14567)

<br>

### 난이도 : ⭐⭐

### 코드
```java
import java.io.*;
import java.util.*;

public class Main {
    static Queue<int[]> que = new ArrayDeque<>();
    static ArrayList<Integer> [] arr;
    static int [] answer;
    static int []degree;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());
        int m = Integer.parseInt(stz.nextToken());

        degree = new int[n+1];
        arr = new ArrayList[n+1];
        for(int i=1; i<=n; i++){
            arr[i] = new ArrayList<>();
        }


        answer = new int[n+1];
        for(int i=0; i<m; i++){
            stz = new StringTokenizer(br.readLine());
            int start = Integer.parseInt(stz.nextToken());
            int end = Integer.parseInt(stz.nextToken());
            if(arr[start] == null){
                arr[start] = new ArrayList<>();
            }
            arr[start].add(end);

            degree[end]++;
        }
        for(int i=1; i<=n; i++){
            if(degree[i] == 0) que.add(new int[]{i,1});
        }
        bfs();

        for(int i=1; i<=n;i++){
            System.out.print(answer[i] + " ");
        }
    }
    public static void bfs(){
        while(!que.isEmpty()){
            int [] temp = que.poll();
            int target = temp[0];
            int count = temp[1];

            answer[target] = count;

            for(int num: arr[target]){
                degree[num]--;
                if(degree[num] == 0){
                    que.add(new int[]{num, count+1});
                }
            }
        }
    }
}
```
</div>
</details>


