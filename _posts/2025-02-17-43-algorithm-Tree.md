---
layout: post
title: "알고리즘 - Tree 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
hidden: true
---

<details>
<summary><strong>트리</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1068)

<br>
### 난이도 : ⭐⭐

root node부터 dfs 순회하면서 삭제 대상 node가 나올시 해당 node 부터 탐색 제외


### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static int answer = 0;
    static ArrayList<Integer>[] arr;
    static int[] child;
    static int target;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int num = Integer.parseInt(stz.nextToken());

        arr = new ArrayList[num];
        child = new int[num];
        for(int i=0; i<num; i++) {
            arr[i] = new ArrayList<>();
        }

        int root = 0;
        stz = new StringTokenizer(br.readLine());
        for(int i=0; i<num; i++){
            int parent = Integer.parseInt(stz.nextToken());
            if(parent == -1) {
                root = i;
                continue;
            }
            arr[parent].add(i);
            child[parent]++;
        }

        stz = new StringTokenizer(br.readLine());
        target = Integer.parseInt(stz.nextToken());

        dfs(root);
        System.out.println(target == root? 0: answer);
    }

    public static void dfs(int node){
        if(arr[node].contains(target)){
            arr[node].remove(Integer.valueOf(target));
        }

        if(arr[node].size() == 0){
            answer++;
            return;
        }
        for(int child :arr[node]){
            if(child == target) {
                continue;
            }
            dfs(child);
        }
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>나무 위의 빗물</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/17073)

<br>
### 난이도 : ⭐⭐

물이 더이상 떨어지지 않을 때는 물이 전부 leaf node에 있을 때 이므로 고정된 물 수와 leaf node 수를 나눈다.

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
<summary><strong>트리 순회</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1991)

<br>
### 난이도 : ⭐⭐

전위 중위 후위 순회 알고리즘 구현

### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static Node[] arr;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int count = Integer.parseInt(stz.nextToken());

        arr = new Node[count];
        for(int i=0; i< count ; i++){
            stz = new StringTokenizer(br.readLine());
            char node = stz.nextToken().charAt(0);
            char left = stz.nextToken().charAt(0);
            char right = stz.nextToken().charAt(0);

            if(arr[node-'A'] == null){
                arr[node-'A'] = new Node(node);
            }
            if(left != '.'){
                arr[left-'A'] = new Node(left);
                arr[node-'A'].left = arr[left-'A'];
            }
            if(right!= '.'){
                arr[right-'A'] = new Node(right);
                arr[node-'A'].right = arr[right-'A'];
            }
        }
        preOrder(arr[0]);
        System.out.println();
        inOrder(arr[0]);
        System.out.println();
        postOrder(arr[0]);
    }
    public static void preOrder(Node root){
        if(root == null) return;
        System.out.print(root.value);
        preOrder(root.left);
        preOrder(root.right);

    }

    public static void inOrder(Node root){
        if(root == null) return;
        inOrder(root.left);
        System.out.print(root.value);
        inOrder(root.right);

    }

    public static void postOrder(Node root){
        if(root == null) return;
        postOrder(root.left);
        postOrder(root.right);
        System.out.print(root.value);

    }

    public static class Node{
        char value;
        Node left;
        Node right;
        public Node(char value){
            this.value = value;
        }
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>트리의 기둥과 가지</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/20924)

<br>
### 난이도 : ⭐⭐
방향이 없는 간선이라 구현시 주의  
기가 노드는 bfs로 가지 길이는 dfs로 구현함

### 코드
```java
import java.util.*;
import java.io.*;


class Main {
    static int count;
    static int root;
    static ArrayList<int[]>[] arr;
    static int []check;
    static int giga;
    static int stick = 0;
    static int leaf = 0;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        count = Integer.parseInt(stz.nextToken());
        root = Integer.parseInt(stz.nextToken());

        arr = new ArrayList[count+1];
        check = new int[count+1];
        for(int i=1; i<= count; i++){
            arr[i] = new ArrayList<>();
        }


        for(int i=1; i<count; i++){
            stz = new StringTokenizer(br.readLine());
            int node1 = Integer.parseInt(stz.nextToken());
            int node2 = Integer.parseInt(stz.nextToken());
            int cost = Integer.parseInt(stz.nextToken());
            arr[node1].add(new int[]{node2,cost});
            arr[node2].add(new int[]{node1,cost});
        }

        check[root] = 1;
        findGiga(root,0);
        findLeaf(giga,0);

        StringBuilder sb = new StringBuilder();
        sb.append(stick).append(" ").append(leaf);
        System.out.println(sb);


    }
    public static void findGiga(int node, int length){
        ArrayDeque<int[]> que = new ArrayDeque();
        que.add(new int[]{node, length});

        while(!que.isEmpty()){
            int[] temp = que.poll();
            int curNode = temp[0];
            int curLength = temp[1];

            if(curNode == root){
                if(arr[curNode].size() == 0 || arr[curNode].size() >= 2) {
                    giga = curNode;
                    stick = curLength;
                    return;
                }
            } else {
                if(arr[curNode].size() == 1 || arr[curNode].size() >= 3) {
                    giga = curNode;
                    stick = curLength;
                    return;
                }
            }
            for(int[] nodeInfo :arr[curNode]){
                if(check[nodeInfo[0]] == 1) continue;
                check[nodeInfo[0]] = 1;
                que.add(new int[]{nodeInfo[0],curLength+nodeInfo[1]});
            }
        }
    }

    public static void findLeaf(int node, int length){
        //leafnode
        if(arr[node].size() == 1){
            leaf = Math.max(leaf, length);
        }
        for(int[] nodeInfo :arr[node]){
            if(check[nodeInfo[0]] == 1) continue;
            check[nodeInfo[0]] = 1;
            findLeaf(nodeInfo[0],length+nodeInfo[1]);
        }
    }
}
```
</div>
</details>
