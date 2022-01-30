---
title: Astar寻路在Unity项目中的简单应用
date: 2022-01-30 11:37:00
tags: 算法
---
### 概念
* 寻找计算两点之间的最短路径，基本思想源于BFS，是Dijkstra寻路的进阶版，相比DijkStra引入了目标点到终点的预估消耗值。

### 算法伪逻辑代码
  ![算法伪码图](Astar_logic.png)
	其中OpenList可以考虑使用最小二叉堆或者优先级队列，这样每次取最小值时间复杂度为O(1)。
---
### Unity逻辑实现

```Csharp
using System;
using System.Collections;
using System.Collections.Generic;
using KeyBoard.Util;
using UnityEngine.Profiling;

namespace KeyBoard.Logic.Map
{
    public abstract class IAstarNode<T> : IComparable
    {
        public float G_Value { get; set; }

        public float H_Value { get; protected set; }

        public abstract bool IsWalkable {get;}

        public float F_Value => G_Value + H_Value;

        public IAstarNode<T> Parent { get; set; }

        public T NodeData { get; private set; }

        public IAstarNode(T data)
        {
            NodeData = data;
        }

        public abstract float GetDistance(IAstarNode<T> target);


        public static bool TryFindPath<Node>(List<Node> nodes, Node start, Node end, int Width, int Length, out List<Node> path) where Node:IAstarNode<T>
        {
#if UNITY_EDITOR
            Profiler.BeginSample("CustomAstar:" + typeof(Node));
            // var watch = System.Diagnostics.Stopwatch.StartNew();
#endif
            path = null;
            if(!nodes.Contains(start) || !nodes.Contains(end))
                return false;
            if(Width < 0 || Length < 0)
                return false;
            var openNodes = new SortedSet<Node>();
            HashSet<Node> closeNodes = new HashSet<Node>();
            path = new List<Node>();
            openNodes.Add(start);
            while(openNodes.Count > 0)
            {
                var curNode = openNodes.Min;
                openNodes.Remove(curNode);
                closeNodes.Add(curNode);

                if(curNode == end) break;

                var idx = nodes.IndexOf(curNode);
                var neighbor = UIUtil.GetNeighbors(idx, (int)Width, (int)Length);
                SearchNeighbor(curNode, nodes.SafeGet(neighbor.left), end, openNodes, closeNodes);
                SearchNeighbor(curNode, nodes.SafeGet(neighbor.right), end, openNodes, closeNodes);
                SearchNeighbor(curNode, nodes.SafeGet(neighbor.up), end, openNodes, closeNodes);
                SearchNeighbor(curNode, nodes.SafeGet(neighbor.down), end, openNodes, closeNodes);
            }

            while(end != null)
            {
                path.Add(end);
                end = end.Parent as Node;
            }
#if UNITY_EDITOR
            // watch.Stop();
            // UnityEngine.Debug.Log($"自定义寻路用时{watch.ElapsedMilliseconds}ms");
            Profiler.EndSample();
#endif
            return path.Count > 0;
        }

        private static void SearchNeighbor<Node>(Node curNode, Node neighborNode, Node end, ISet<Node> openNodes, ISet<Node> closeNodes) where Node:IAstarNode<T>
        {
            if(neighborNode == null || !neighborNode.IsWalkable || closeNodes.Contains(neighborNode))
                return;
            float newCost_G = curNode.G_Value + curNode.GetDistance(neighborNode);
            if(newCost_G < neighborNode.G_Value || !openNodes.Contains(neighborNode))
            {
                neighborNode.G_Value = newCost_G;
                neighborNode.H_Value = neighborNode.GetDistance(end);
                neighborNode.Parent = curNode;
                if(!openNodes.Contains(neighborNode))
                {
                    openNodes.Add(neighborNode);
                }
            }
        }


        public int Compare(IAstarNode<T> x, IAstarNode<T> y)
        {
            if(x.H_Value < y.H_Value)
                return -1;
            if(x.H_Value > y.H_Value)
                return 1;
            return 0;
        }

        public int Compare(object x, object y)
        {
            var a = x as IAstarNode<T>;
            var b = y as IAstarNode<T>;
            if(a != null && b != null)
                return a.CompareTo(b);
            return 0;
        }

        public int CompareTo(object obj)
        {
            if (obj == null) return 1;

            var other = obj as IAstarNode<T>;
            if (other != null)
                return F_Value.CompareTo(other.F_Value);
            else
                throw new ArgumentException("Object is not an IAstarNode");
        }
    }
}

```
   

---
