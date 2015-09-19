title: Floodlight No.6
date: 2014/10/30 22:26:36 
tags: 
- Floodlight
- Controller
categories: Floodlight
---

## TopologyManager Module ##

该模块接收LinkDiscover的跟新消息，当有链路跟新时，将重新计算topo。主要调用TopologyInstance类中的compute函数。compute函数即注释如下
<!--more-->
    public void compute() {

        // Step 1: Compute clusters ignoring broadcast domain links
        // Create nodes for clusters in the higher level topology
        // Must ignore blocked links.
        identifyOpenflowDomains();//利用强连通分量Tarjan算法，将topo划分强连通子树

        // Step 1.1: Add links to clusters
        // Avoid adding blocked links to clusters
        addLinksToOpenflowDomains();//将同属于一个连通分量的switch的link添加到连通分量中

        // Step 2. Compute shortest path trees in each cluster for 
        // unicast routing.  The trees are rooted at the destination.
        // Cost for tunnel links and direct links are the same.
        calculateShortestPathTreeInClusters();//使用dijstra算法，计算一个cluster内部的所有点最短路径

        // Step 3. Compute broadcast tree in each cluster.
        // Cost for tunnel links are high to discourage use of 
        // tunnel links.  The cost is set to the number of nodes
        // in the cluster + 1, to use as minimum number of 
        // clusters as possible.
        calculateBroadcastNodePortsInClusters();

        // Step 4. print topology.
        // printTopology();
    }
