# Algorithm Notes for the First Half of the Semester

## 拓扑排序：Kahn算法

### 概述

针对**有向无环图 DAG** 的一种排序方法，解决的是一种**依赖关系排序问题**：如果图中存在一条边 $(v_i, v_j)$，那么 $v_i$ 必须在这个顺序中排在 $v_j$ 前面，时间复杂度 \(O(V+E)\) 。

**注意：**

1. 可能存在多个合法的拓扑序。
2. 有环图不存在拓扑序（这说明拓扑排序**可以判环**）。

### Kahn算法步骤

1. 计算所有顶点的入度。
2. 找到一个入度为 0 的顶点 $u$，把这个顶点追加到当前拓扑序列的末尾，从图中删除这个顶点。
3. 重复操作，直到所有点都被删除，我们就得到了拓扑排序的结果。
4. 如果无法把所有的点都从图中删除（拓扑序列的顶点数 < 有向图的顶点数 ），说明有向图存在环。

### 代码

```c++
#include <iostream>
#include <vector>
#include <queue>
using namespace std;

const int MAXN = 100005;

int n, m;
vector<int> g[MAXN];
int indeg[MAXN];
vector<int> topo;

// 拓扑排序，返回 true 表示无环，false 表示有环
bool topo_sort() {
    queue<int> q;

    for (int i = 1; i <= n; i++) {
        if (indeg[i] == 0) q.push(i);
    }

    while (!q.empty()) {
        int u = q.front();
        q.pop();

        topo.push_back(u);

        for (int v : g[u]) {
            indeg[v]--;

            if (indeg[v] == 0) {
                q.push(v);
            }
        }
    }

    return (int)topo.size() == n;
}

int main() {
    cin >> n >> m;

    for (int i = 1; i <= m; i++) {
        int u, v;
        cin >> u >> v;

        g[u].push_back(v);
        indeg[v]++;
    }

    if (!topo_sort()) {
        cout << "Cycle\n";
    } else {
        for (int x : topo) {
            cout << x << ' ';
        }
        cout << '\n';
    }

    return 0;
}
```

### 拓展

1. **求字典序最小拓扑序**：即 **“编号小的点优先”**，把拓扑排序的 \(queue\) 换成 \(priority\) \(queue\) 即可，保证每次编号最小的点先出队。

   ```c++
   priority_queue<int, vector<int>, greater<int> > pq;
   ```

2. **判断拓扑序是否唯一**：如果某一时刻队列里有多个入度为 0 的点，说明当前可以选多个点，因此拓扑序不唯一,额外使用一个 \(bool\) 型变量 \(unique\) 记录即可，记录方法：

   ```c++
   if (q.size() > 1) unique = false;
   ```

## 并查集

### 概述

**核心思想：** 用树来表示连通分量。

2种操作：

- **Find(u)：** 查找，找到节点 $u$ 所在树的根节点。
- **Merge(u, v)：** 合并，把 $u$ 所在的连通分量和 $v$ 所在的连通分量合并起来。

### 代码

```c++
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

const int MAXN = 500005;

struct Edge {
    int u, v, w;
};

struct Query {
    int id;
    int start;
    int limit;
};

int n, m, q;
vector<Edge> edges;
vector<Query> queries;
vector<int> ans;

int parent[MAXN];
int sz[MAXN];

void init(int n) {
    for (int i = 1; i <= n; i++) {
        parent[i] = i;
        sz[i] = 1;
    }
}

int find(int x) {
    if (parent[x] == x) return x;
    return parent[x] = find(parent[x]);   // 路径压缩
}

void unite(int a, int b) {
    int ra = find(a);
    int rb = find(b);

    if (ra == rb) return;

    // 小集合挂到大集合上
    if (sz[ra] < sz[rb]) {
        swap(ra, rb);
    }

    parent[rb] = ra;
    sz[ra] += sz[rb];
}

int main() {
    cin >> n >> m >> q;

    for (int i = 0; i < m; i++) {
        int u, v, w;
        cin >> u >> v >> w;
        edges.push_back({u, v, w});
    }

    for (int i = 0; i < q; i++) {
        int x, c;
        cin >> x >> c;
        queries.push_back({i, x, c});
    }

    init(n);

    sort(edges.begin(), edges.end(), [](Edge a, Edge b) {
        return a.w < b.w;
    });

    sort(queries.begin(), queries.end(), [](Query a, Query b) {
        return a.limit < b.limit;
    });

    ans.resize(q);

    int edgeIndex = 0;

    for (auto query : queries) {
        int start = query.start;
        int limit = query.limit;

        // 加入所有边权 <= limit 的边
        while (edgeIndex < m && edges[edgeIndex].w <= limit) {
            unite(edges[edgeIndex].u, edges[edgeIndex].v);
            edgeIndex++;
        }

        ans[query.id] = sz[find(start)];
    }

    for (int i = 0; i < q; i++) {
        cout << ans[i] << endl;
    }

    return 0;
}
```

### 补充

**$sort + lambda$ 排序写法**的格式说明：

$sort$ 的基本格式：

```c++
sort(起始位置, 结束位置, 比较规则);
```

例如按照边权 $w$ **从小到大**排序：

```c++
sort(edges.begin(), edges.end(), [](Edge a, Edge b) { return a.w < b.w; });
```

其中：

`edges.begin()` 表示排序起点，`edges.end()` 表示排序终点的后一个位置。

`[](Edge a, Edge b) { return a.w < b.w; }` 是一个 lambda **函数**，可以理解成临时写出来的比较函数。

比较规则：

`return a.w < b.w;` 表示如果 `a.w` 更小，那么 `a` 排在 `b` 前面，所以是升序排序。

## 最小生成树（MST）

### 概述

 一个**带权无向图** $G=(V,E)$，其中每条边 $e$ 有权重 $w_e$。

当满足下面条件时，$T=(V,E')$ 是图 $G$ 的一个 **生成树**：

1. $E' \subseteq E$，并且 $E'$ 恰好有 $|V|-1$ 条边；
2. $T$ 是连通的；
3. **最小生成树，MST**：在所有生成树中，使总边权最小的那一棵生成树。

### Prim算法

贪心算法，时间复杂度 \(O(∣E∣log∣E∣)\) 。

重复下面步骤，直到：$S = V$ ，也就是所有顶点都已经被加入生成树。

#### 步骤

1. 令 $S$ 表示当前已经连入**最小生成树的点集**，使用一个小根堆 $H$ 来维护候选边，堆中元素记录为 `{边权, 目标点}`。

2. 任选一个起点，例如点 $1$，把 `{0, 1}` 放入堆中。

   这里的边权为 $0$，表示从点 $1$ 开始，不需要真正选边，因此不会增加最小生成树的总代价。

3. 从堆 \(H\) 中弹出权值最小的边 \(e\) ，查看边 \(e\) 去往的点 \(x\) ：

   - 如果 \(x\) 已经在 \(S\) 中，就丢弃这条边；
   - 如果 \(x\) 不在 \(S\) 中，则把 \(x\) 加入 \(S\) ，边 \(e\) 属于MST。

   枚举点 $x$ 的所有邻边。对于每一条从 $x$ 连向 $v$ 的边，如果点 $v$ 还没有加入 $S$，就把这条边作为新的候选边加入堆 $H$ 中。

4. 重复步骤3，直到 \(H\) 为空，即得到MST。

**注意：** Prim算法中，无向边要用两条有向边的形式表示。

#### 简 · 代码

```c++
// Prim(稠密图)
vector<pair<int,int>> adj[MAX];
pq.push({0,1}); // first weight,second vertex，小根堆
while(!pq.empty()){
    int u = pq.top().second, w = pq.top().first;
    pq.pop();
    if(visited[u]) continue;
    visited[u] = true;
    cost += w;
    for(auto p:adj[u]){
        int v = p.first;
        int weight = p.second;
        if(!visited[v])
            pq.push({weight, v});
    }
}
```

### Kruskal算法

贪心算法，通过不断选择权重最小的边，并确保选择的边不形成环，最终构建出MST，时间复杂度：$O(∣E∣log∣E∣)$ 。

#### 步骤

使用 **并查集** 来维护连通性。

1. 图中所有边按边权从小到大排序。

2. 初始化一个空的边集，用于存储MST的边（或者用 cost 记录总权值等）。

3. 按顺序枚举边 $e=(u,v)$，并通过查询并查集来判断 $u$ 和 $v$ 是否已经连通。

   如果已经连通，那么舍弃这条边 $e$。

   否则将该边加入MST的边集中，并且在并查集中合并 $u$ 和 $v$。

4. 直到边集中的边数 = 顶点数 - 1 或所有边都已考虑完毕，返回MST的边集作为结果。

**注：**

1. Kruskal常常不需要建图，只需要存储边即可。
2. 并查集只写一个 $find$ 函数就可以。

#### 代码

```c++
// Kruskal（稀疏图，union-find set）
int find(int x) {
    if (parent[x] != x) {
        parent[x] = find(parent[x]);
    }
    return parent[x];
}

parent[i] = i; // initialize

sort(edges.begin(), edges.end(), [](Edge a, Edge b) {
    return a.w < b.w;
});

for (auto e : edges) {
    int u = e.u, v = e.v, w = e.w;
    int root_u = find(u), root_v = find(v);

    if (root_u != root_v) {
        parent[root_u] = root_v;
        cost += w;
        cnt++;

        if (cnt == n - 1) break;
    }
}

if (cnt == n - 1) cout << cost;
else cout << "impossible";

// 判断是否存在最小生成树：
// 加入一条边 cnt++，如果 cnt < n - 1，则不存在
// 最小瓶颈生成树：
// 当 cnt == n - 1 时，输出 e.w
```

## 最短路径

### Dijkstra算法

#### 概述

贪心算法，处理**单源最短路**问题，要求**边权非负**，时间复杂度为：**\[O(m \log m)\]**， \(m\) 是边数。

#### 步骤

1. `dist[i]`表示从源点到节点 \(i\) 的当前最短距离。

2. 准备好小根堆，

   ```c++
   priority_queue<pair<int,int>,vector<pair<int,int>>,greater<pair<int,int>>> pq;
   ```

   小根堆存放`(dist[x],x)`, \(x\) 是节点。

3. 令`dist[源点]=0`,`(0,源点)`进入小根堆。

4. 从小根堆弹出`(dist,u点)`

   - 如果`dist > dist[u]`，说明这是旧记录，忽略，重复步骤4。

   - 反之，考察 \(u\) 的每一条边，假设某边去往 \(v\) ，边权为 \(w\) ：

     如果`dist[u] + w < dist[v]`，令`dist[v] = dist[u] + w`，再把`(dist[v],v)`加入小根堆，处理完 \(u\) 的每一条边之后，重复步骤4。

5. 小根堆为空过程结束，dist表记录了源点到每个节点的最短距离。

#### 代码

```c++
#include <iostream>
#include <vector>
#include <queue>
#include <cstring>

using namespace std;

vector<pair<int, int>> adj[1001];// 邻接表，first表示邻接顶点，second表示边权
int dist[1001];
priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> pq;
// 小根堆，first表示到源点的距离，second表示顶点
// 按照距离排序

int main(){
    int n, m;
    cin >> n >> m;
    int s, t;
    cin >> s >> t;
    for (int i = 0; i < m;i++){
        int u, v, w;
        cin >> u >> v >> w;
        adj[u].push_back({v, w});
    }

    memset(dist, 0x3f, sizeof(dist)); // 初始化，每个点到源点的距离设置为足够大
    dist[s] = 0; // 源点到自身的距离是0

    pq.push({0, s});// 从源点s开始
    while(!pq.empty()){// Dijkstra algorithm
        int d = pq.top().first;
        int u = pq.top().second;
        pq.pop();
        if(d > dist[u]){
            continue;// 跳过旧的记录
        }
        for(auto e: adj[u]){
            int v = e.first;
            int w = e.second;
            if(dist[v] > dist[u] + w){// 松弛操作
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    cout << dist[t] << endl;
    return 0;
}

/* input sample:
5 6
1 4
1 2 2
1 3 1
3 2 1
2 4 2
2 5 1
5 4 1
*/
/* output sample:
4
*/
```

### Bellman-Ford算法

#### 概述

比较暴力的算法，处理**单源最短路**问题，**边权可以为负**，最短路存在要求**无负环**，时间复杂度 **\[O(|V||E|)\]**。

#### 步骤

1. 初始化，`dist[s]=0`，`dist[其他点]=0x3f(无穷大)`。
2. 做\[(|V|-1)\]轮循环，每轮循环对所有边尝试进行一次松弛操作。
3. 当一轮循环中没有成功的松弛操作时/进行了\[(|V|-1)\]轮循环后，算法停止。
4. （可选），再对所有边进行一次松弛操作，如果某条边还能继续松弛成功，说明图中存在负环。

#### 代码

```c++
#include <iostream>
#include <vector>
#include <cstring>

using namespace std;

const int INF = 0x3f3f3f3f;

struct edge{
    int u, v, w;
};
vector<edge> edges;
int dist[1001];
bool relax(int u,int v,int w){
    if(dist[u] != INF && dist[v] > dist[u]+w){
        dist[v] = dist[u] + w;
        return true;
    }
    return false;
}

int main(){
    int n, m;
    cin >> n >> m;
    int s, t;
    cin >> s >> t;
    for (int i = 0; i < m;i++){
        int u,v,w;
        cin>>u>>v>>w;
        edges.push_back({u,v,w});
    }
    memset(dist, 0x3f, sizeof(dist));
    dist[s] = 0;// 初始化
    for (int i = 0; i < n - 1; i++){// Bellman-Ford算法
        for (auto &e: edges){
            relax(e.u, e.v, e.w);
        }
    }

    // 判负环
    for(int i = 0; i < m; i++){
        if(relax(edges[i].u, edges[i].v, edges[i].w)){
            cout << "Negative cycle exists!" << endl;
            return 0;
        }
    }
    cout << dist[t] << endl;
    return 0;
}

/*input sample:
5 6
1 4
1 2 2
1 3 3
3 2 -2
2 4 2
2 5 1
5 4 1
*/
/*output sample:
3
*/
```

#### 扩展

1. 若**最短路径允许经过的边数最多限制为 \(k\) 条**，我们可以把 \(Bellman-Ford\) 的循环上限修改为 \(k\) 次，原因是每多一轮循环，本质上是允许路径多经过一条边。需要注意：每一轮必须使用上一轮的 `dist` 进行松弛，避免同一轮内连续使用刚更新的距离，从而经过超过一条新边。

```c++
int k;
int backup[1001];
for (int i = 0; i < k; i++){
    memcpy(backup, dist, sizeof(dist));
    for (int j = 0; j < m;j++){
        int u = edges[j].u;
        int v = edges[j].v;
        int w = edges[j].w;
        if (backup[u] != INF && dist[v] > backup[u] + w){
            dist[v] = backup[u] + w;
        }
    }
}
```

2. 假设**图中不存在非正环**，要求计算从 $s$ 到 $t$ 的**最短路径条数**。

   **2.1 一种错误方法的讨论**

   类似 $Dijkstra$，在松弛边 $u \to v$ 时，同时更新最短路径条数 $f_{s\to v}$，规则如下：

   $$
   f_{s\to v}:=
   \begin{cases}
   f_{s\to v}, & d_{s\to u}+w_{u\to v}>d_{s\to v}\\[4pt]
   f_{s\to v}+f_{s\to u}, & d_{s\to u}+w_{u\to v}=d_{s\to v}\\[4pt]
   f_{s\to u}, & d_{s\to u}+w_{u\to v}<d_{s\to v}
   \end{cases}
   $$

   是否可行？

   **不可行**。由于 $Bellman-Ford$ 进行多轮全边扫描，同一条边会被反复检查。这样即使某些点的最短距离已经不再变化，若仍在“等长松弛”时直接做 `f[v] += f[u]`，就会把同一份来自 `u` 的贡献在后续轮次里重复加到 `v` 上，导致最短路径条数被**重复统计**。

   **2.2 改进方案**

   为了避免重复统计，需要额外维护一个量 $l[u][v]$，表示**边 $u \to v$ 已经向 $v$ 累计过的那部分 $f[u]$**。

   - 若 $d[u] + w > d[v]$

     经过边 $u \to v$ 不能得到更短或同样短的最短路，不需要更新任何量。

   - 若 $d[u] + w = d[v]$

     补加新增量到 $f[v]$ 上：

     $$
     f[v] \leftarrow f[v] + \bigl(f[u] - l_{u,v}\bigr)
     $$

     同时将这条边已累计的贡献更新为当前的 $f[u]$：

     $$
     l_{u,v} \leftarrow f[u]
     $$

   - 若 $d[u] + w < d[v]$

     通过 $u \to v$ 找到了更短的路径，因此 **$v$ 原来记录的最短路信息全部失效，需要整体更新**。此时应更新距离：

     $$
     d[v] \leftarrow d[u] + w
     $$

     并将最短路径条数直接改为来自 $u$ 的最短路径条数：

     $$
     f[v] \leftarrow f[u]
     $$

     边 $u \to v$ 对 $v$ 的已累计贡献应更新为：

     $$
     l_{u,v} \leftarrow f[u]
     $$

     所有其他指向 $v$ 的旧累计记录都已经对应旧最短路，必须清零，即：

     $$
     l_{x,v} \leftarrow 0 \qquad (x \neq u)
     $$

   **2.3 时间复杂度**

   - 如果在“找到更短路”时直接把所有 `l[x][v]` 全部清零，总复杂度为  \[O(|V|^2|E|)\] 。

   - 如果只重置非零的 `l[x][v]`，则可以将复杂度优化到 \(O(|V||E|)\)，与普通 \(Bellman-Ford\) 同量级。

   **2.4 代码**

   ```c++
   #include <iostream>
   #include <vector>

   using namespace std;

   // Bellman-Ford 算法
   // 允许负权边，但不存在负环
   // 计算从 s 到 t 的最短路径条数

   const long long INF = 0x3f3f3f3f;

   struct edge{
       int u, v, w;
   };
   edge edges[1001];
   long long int dist[1001];
   long long int cnt[1001] = {0}; // cnt[u] 表示从 s 到 u 的最短路径条数
   long long int part[1001][1001] = {0};
   // part[u][v] 表示通过边 (u,v,w)，cnt[u] 已经向 cnt[v] 累计过的那部分值
   // 用来避免重复统计

   int n, m;

   bool relax(int u,int v,int w){
       if(dist[u] == INF){
           return false;
       }
       if(dist[v] > dist[u] + w){
           dist[v] = dist[u] + w;
           cnt[v] = cnt[u];
           for(int i = 1; i <= n; i++){
               part[i][v] = 0;
           }
           part[u][v] = cnt[u];
           return true;
       }
       else if(dist[v] == dist[u] + w){
           cnt[v] = cnt[v] + cnt[u] - part[u][v];
           part[u][v] = cnt[u];
       }
       // else if(dist[v] < dist[u] + w){
       //     // 不需要更新
       // }
       return false;
   }

   int main(){
       cin >> n >> m;
       int s, t;
       cin >> s >> t;
       for (int i = 0; i < m; i++){
           cin >> edges[i].u >> edges[i].v >> edges[i].w;
       }
       for (int i = 1; i <= n; i++){
           dist[i] = INF;
       }
       dist[s] = 0; // 初始化
       cnt[s] = 1;
       for (int i = 0; i < n - 1; i++){ // Bellman-Ford 主过程
           for (int j = 0; j < m; j++){
               relax(edges[j].u, edges[j].v, edges[j].w);
           }
       }
       cout << cnt[t] << endl;
       return 0;
   }

   /*input sample:
   5 6
   1 4
   1 2 2
   1 3 3
   3 2 -2
   2 4 2
   2 5 1
   5 4 1
   */
   /*output sample:
   2
   */
   ```

### Floyd-Warshall算法

#### 概述

求解**图中任意两点之间最短距离**的算法，适用于**任何图**（有向无向均可，边权正负均可），**不能有负环**（保证最短路存在），时间复杂度\[O(|V|^3)\] 。

#### 步骤

1. **初始化**：`dist[i][j]` 表示从顶点 $i$ 到顶点 $j$ 的当前最短距离。初始时，令所有 `dist[i][j] = INF`，表示默认不可达；再令 `dist[i][i] = 0`，表示每个点到自己的距离为 0。读入每一条边 $(u,v,w)$时，令 `dist[u][v] = min(dist[u][v], w)`，这样存在重边时保留其中边权最小的一条。

2. **循环更新**：依次枚举每一个顶点 $k$ 作为中间点。对于每个固定的 $k$，再枚举所有点对 $(i,j)$，检查从 $i$ 到 $j$ 的路径是否可以通过顶点 $k$ 变得更短。若经过 $k$ 的路径长度 `dist[i][k] + dist[k][j]` 小于当前的 `dist[i][j]`，就用它更新 `dist[i][j]`。这个过程对应状态转移：

   $$
   dist[i][j] = \min(dist[i][j],\ dist[i][k] + dist[k][j])
   $$

3. 当所有顶点 $k$ 都枚举完成后，`dist[i][j]` 中存放的就是从 $i$ 到 $j$ 的最短路径长度。

#### 代码

```c++
#include <iostream>
#include <vector>
using namespace std;

// Floyd-Warshall 算法
// 求带权图中所有点对之间的最短路径

const int INF = 0x3f3f3f3f;

vector<vector<int>> dist(301, vector<int>(301, 0));

int main(){
    int n, m;
    cin >> n >> m;
    for (int i = 1; i <= n; i++){
        for (int j = 1; j <= n; j++){ // 初始化距离
            dist[i][j] = INF;
        }
        dist[i][i] = 0;
    }
    for (int i = 0; i < m; i++){ // 初始化图
        int u, v, w;
        cin >> u >> v >> w;
        dist[u][v] = min(dist[u][v], w);
    }
    for (int k = 1; k <= n; k++){ // Floyd-Warshall 算法
        for (int i = 1; i <= n; i++){
            for (int j = 1; j <= n; j++){
                if (dist[i][k] != INF && dist[k][j] != INF){
                    dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]);
                }
            }
        }
    }

    for (int i = 1; i <= n; i++){
        for (int j = 1; j <= n; j++){
            if(dist[i][j] == INF)
                cout << "INF ";
            else
                cout << dist[i][j] << " ";
        }
        cout << endl;
    }
    return 0;
}
```

#### 扩展：图的最小环问题

**Question：** 给定一个带权有向图G，找出G中边权之和最小的环（保证图中存在环），输出环的边权之和即可。

**Answer：** 有向图最小环可以拆成**一条最短路 + 一条回边**，先用 \(Floyd-Warshall\) 求所有点对的最短路，然后枚举

$$
dist[i][j] + weight[j][i]
$$

代码：

```c++
#include <iostream>
#include <vector>
using namespace std;

const int INF=0x3f3f3f3f;
int dist[305][305]; // 两点之前最短距离
int weight[305][305]; //邻接矩阵，两点之间的边权值
int n,m;
int min_cycle=INF;

// 求图的最小环

int main(){

    // 初始化

    cin>>n>>m;
    for(int i=1;i<=n;++i){
        for(int j=1;j<=n;++j){
            dist[i][j]=INF;
            weight[i][j]=INF;
        }
    }
    for(int i=1;i<=n;++i){
        dist[i][i]=0;
    }
    for(int i=0;i<m;++i){
        int u,v,w;
        cin>>u>>v>>w;
        dist[u][v]=w;
        weight[u][v]=w;
    }

    // Floyd-Warshall算法

    for(int k=1;k<=n;++k){
        for(int i=1;i<=n;++i){
            for(int j=1;j<=n;++j){
                if(dist[i][k]<INF && dist[k][j]<=INF){
                    dist[i][j]=min(dist[i][j],dist[i][k]+dist[k][j]);
                }
            }
        }
    }

    //求最小环的边权之和

    for(int i=1;i<=n;++i){
        for(int j=1;j<=n;++j){
            if(i!=j && dist[i][j]<INF && weight[j][i]<INF){
                min_cycle=min(min_cycle,dist[i][j]+weight[j][i]);
            }
        }
    }

    cout<<min_cycle;
    return 0;
}

/*input sample:
5 9
4 2 8
1 4 6
2 3 10
5 3 8
1 2 3
5 2 9
3 1 3
4 3 6
2 1 3
*/
/*output sample:
6
*/
```

## 强连通分量（SCC）

### 定义

1. **强连通：** 给定一个有向图 $G$ ，如果任意一对顶点都可以**互相到达**，那么称 $G$ 是**强连通（strongly connected）** 的。

2. **强连通分量（Strongly connected component, SCC）** 是 $G$ 的一个子图，这个子图：

   - 是**强连通**的
   - 并且是**极大的**，这里 **“极大”的意思是：** 再往这个子图里加入任何一个顶点，都会破坏它的强连通性。

3. **反图（Reverse graph）：** 反图就是把原图中每条边的方向都反过来，它与原图有**相同的强连通分量**。

4. **缩点图（Kernel graph）：** 将每个强连通分量缩成一个顶点，得到的新图是一个**有向无环图（DAG）**，记作：

   $$
   G^{SCC}=(V^{SCC},E^{SCC})
   $$

### Kosaraju 算法

#### 概述

将顶点分组为若干个 **强连通分量（SCCs）**，

时间复杂度：\(O(∣V∣+∣E∣)\) 。

#### 步骤

1. $DFS1$ ：在原图上做后序 DFS ，记录完成顺序。
2. $DFS2$ ：在反图上按照第一次 DFS 完成顺序的逆序做 DFS ，每次 DFS 搜索到的一整块点就是一个 SCC 。

#### 代码

```c++
// SCC Kosaraju

vector<int> adj[MAX], adj_inv[MAX];
vector<vector<int>> scc;
vector<int> post_order, temp_scc, scc_index(MAX);
bool visited[MAX];

void DFS1(int s) { // 原图 DFS
    visited[s] = true;
    for (int u : adj[s]) {
        if (!visited[u]) {
            DFS1(u);
        }
    }
    post_order.push_back(s);
}

void DFS2(int s) { // 反图 DFS
    visited[s] = true;
    temp_scc.push_back(s);
    for (int u : adj_inv[s]) {
        if (!visited[u]) {
            DFS2(u);
        }
    }
}

memset(visited, false, sizeof(visited));

for (int i = 1; i <= n; i++) {
    if (!visited[i]) {
        DFS1(i);
    }
}

memset(visited, false, sizeof(visited));

for (int i = post_order.size() - 1; i >= 0; i--) {
    if (!visited[post_order[i]]) {
        temp_scc.clear();
        DFS2(post_order[i]);
        scc.push_back(temp_scc);
        // 给SCC里的每个点记录所属SCC的编号
        for (int u : temp_scc) {
            scc_index[u] = scc.size() - 1;
        }
    }
}
// 从 start 出发，沿一条路径最多经过多少个点？
// max_count(scc_index[start])
// 在缩点DAG上做记忆化搜索DP
int max_count(int st) {
    if (dp[st] != -1) return dp[st]; // 记忆化搜索
    int maximum = 0;
    for (int u : scc[st]) {
        for (int v : adj[u]) {
            if (scc_index[v] != st) {
                maximum = max(maximum, max_count(scc_index[v]));
            }
        }
    }
    dp[st] = scc[st].size() + maximum;
    return dp[st];
}

// 最少加几条边，才能使原图变成强连通图？
// 统计缩点DAG中入度为0的点数A，出度为0的点数B，答案是max(A,B)
// 注意特判scc.size()==1，说明缩点后只有1个SCC，原图是强连通的，输出0
for (int i = 0; i < scc.size(); i++) {
    for (int u : scc[i]) {
        for (int v : adj[u]) {
            if (scc_index[v] != scc_index[u]) {
                indeg[scc_index[v]]++;
                outdeg[scc_index[u]]++;
            }
        }
    }
}
```

## 网络流

### 最大流问题

#### 定义

1. **流网络（容量网络）**：一个**连通的有向图**

   $$
   G=(V,E)
   $$

   - 若边 $(u,v)\in E$，则它有一个**非负容量**

     $$
     c(u,v)>0
     $$

   - 如果 $(u,v)\notin E$，则规定

     $$
     c(u,v)=0
     $$

   - 图中有两个特殊顶点：**源点** $s$ 和 **汇点** $t$ 。

2. **流（可行流）**：流 $f$ 是一组实数

   $$
   f(u,v)
   $$

   它们需要满足以下条件：

   - **容量限制**：

     $$
     f(u,v)\le c(u,v)
     $$

   - **反对称性**：

     $$
     f(u,v)=-f(v,u)
     $$

   - **流量守恒**：对任意

     $$
     u\in V-\{s,t\}
     $$

     都有

     $$
     \sum_{v} f(u,v)=0
     $$

     也就是说，除了源点 $s$ 和汇点 $t$ 之外，**每个顶点流入它的流量等于流出它的流量**。

   **流 $f$ 的值** 定义为

   $$
   |f|=\sum_{v\in V} f(s,v)
   $$

   也就是：**从源点 $s$ 流出的总流量**，也等于：**流入汇点 $t$ 的总流量**。

3. **最大流问题**  ：在一个流网络中，**找到一个流值最大的可行流**。

   ​            或者说在满足容量限制和流量守恒的前提下,**求从源点 $s$ 到汇点 $t$ 最多能送多少流量**。

4. **剩余容量（ \(slack\) ）**，定义为：

   $$
   c(u,v)-f(u,v)
   $$

   沿着选中的一条路径，能够增加的流量等于路径上所有边剩余容量的最小值：

   $$
   \min_{(u,v)\in path} \big(c(u,v)-f(u,v)\big)
   $$

5. **剩余网络（\(Residual\) \(networks\))** ：

   对于流 $f$，对应的剩余网络记作 $G_f=(V,E_f)$。在这个网络中，边 $(u,v)$ 的容量为

   $$
   c_f(u,v)=c(u,v)-f(u,v)
   $$

   并且我们还需要考虑**反向边**，由反对称性（skew symmetry），

   $$
   c_f(v,u)=c(v,u)-(-f(u,v))=c(v,u)+f(u,v)
   $$

6. **增广路（\(Augmenting\) \(paths\))** ：

   在剩余网络中，一条从源点 $s$ 到汇点 $t$ 的路径 $p$，并且这条路径上的每一条边都具有**正的**剩余容量。

   一条增广路 $p$ 的**可增广流量**定义为这条路径上所有边的剩余容量的最小值。

   $$
   c_f(p)=\min_{(u,v)\in p} c_f(u,v)
   $$

   找到一条增广路后，**对原图的更新**：

   对于路径 $p$ 上的每一条边 $(u,v)$，把该边的流量更新为

   $$
   f(u,v):=f(u,v)+c_f(p)
   $$

   同时把反方向的流量更新为

   $$
   f(v,u):=f(v,u)-c_f(p)
   $$

   于是我们得到一个新的流网络，又可以得到一个新的剩余网络，然后我们继续寻找新的增广路······循环重复这一过程，直到——

   **我们在剩余网络中找不到增广路了，这说明我们得到了最大流。**

   这是一个**定理**：

   **A flow is maximum ⇔ there is no augmenting path.**

#### Dinic算法

##### 分层图（\(Level\) \(graph\))

设 $\text{dist}(v)$ 表示在剩余网络 $G_f$ 中，从源点 $s$ 到顶点 $v$​ 的最短路径长度（**长度按边数计算**）。$G_f$ 的分层图定义为图

$$
G_L=(V,E_L,c_f)
$$

其中边集 $E_L$ 由所有满足下面条件的边组成：

$$
(u,v)\in E_f \quad \text{且} \quad \text{dist}(v)=\text{dist}(u)+1
$$

##### **阻塞流（\(Blocking\) \(flow\))**

阻塞流是分层图 $G_L$ 上的一个流，并且满足：每一条从 $s$ 到 $t$ 的路径上，至少都有一条已经饱和的边。也就是说，$s$ 到 $t$ 被“堵住了”。

##### 步骤

1. BFS建分层图。
2. DFS找增广路。
3. 循环前两步直到BFS失败，得出最大流。

##### 代码

```c++
#include <iostream>
#include <vector>
#include <queue>
#include <cstring>
using namespace std;

const int INF=0x3f3f3f3f;

int n,m;
struct Edge{
    int to; //这条边指向的节点
    int cap; //当前剩余容量
    int rev; //反向边在对方邻接表中的下标
};
vector<Edge> graph[205];
int level[205];
int cur[205]; //当前弧优化

bool bfs(int s,int t){
    memset(level,-1,sizeof(level));
    queue<int> q;
    level[s]=0;
    q.push(s);

    while(!q.empty()){
        int u=q.front();
        q.pop();

        for(int i=0;i<graph[u].size();++i){
            Edge &e=graph[u][i];
            if(e.cap>0 && level[e.to]==-1){
                level[e.to]=level[u]+1;
                q.push(e.to);
            }
        }
    }

    return level[t] != -1;
}

int dfs(int u,int t,int flow){
    if(u==t) return flow;
    for(int &i = cur[u];i<graph[u].size();i++){
        Edge &e=graph[u][i];
        if(e.cap>0 && level[e.to]==level[u]+1){
            int d=dfs(e.to,t,min(flow,e.cap));
            if(d>0){
                e.cap-=d;
                graph[e.to][e.rev].cap+=d;
                return d;
            }
        }
    }

    return 0;
}

int dinic(int s,int t){
    int maxflow=0;
    while(bfs(s,t)){
        memset(cur,0,sizeof(cur));
        int f;
        while((f=dfs(s,t,INF))>0){
            maxflow+=f;
        }
    }

    return maxflow;
}

void addEdge(int u,int v,int c){
    Edge a={v,c,(int)graph[v].size()};
    Edge b={u,0,(int)graph[u].size()};
    graph[u].push_back(a);
    graph[v].push_back(b);
}

int main(){
    while(cin>>n>>m){
        for(int i=1;i<=m;++i){
            graph[i].clear();
        }

        for(int i=0;i<n;++i){
            int s,e,c;
            cin>>s>>e>>c;
            addEdge(s,e,c);
        }

        cout<<dinic(1,m)<<endl;
    }
    return 0;
}

/*input sample:
5 4
1 2 40
1 4 20
2 4 20
2 3 30
3 4 10
*/
/*output sample:
50
*/
```

### 最大流的扩展应用

**最大流dinic算法模板代码**

```c++
#include <iostream>
#include <vector>
#include <queue>
#include <algorithm>
#include <climits>
using namespace std;

constexpr int INF = 2e9;

// 边结构
struct Edge {
    int to, capacity, flow, rev;
    Edge(int to, int capacity, int rev) : to(to), capacity(capacity), flow(0), rev(rev) {}
};

struct MaxFlow {
    // n为点数, 编号从1到n
    int n;
    vector<vector<Edge>> graph;
    vector<int> level; // 层次图
    vector<int> iter;  // 当前弧优化

    MaxFlow(int n_) : n(n_){
        level.resize(n + 1);
        iter.resize(n + 1);
        graph.resize(n + 1);
    }

    // 添加边
    void addEdge(int from, int to, int capacity) {
        if(from > n || to > n) {
            cerr << "Error: edge node number exceed n" << endl;
            return;
        }
        graph[from].push_back(Edge(to, capacity, graph[to].size()));
        graph[to].push_back(Edge(from, 0, graph[from].size() - 1)); // 反向边容量为0
    }

    // BFS构建层次图
    bool level_graph(int s, int t) {
        fill(level.begin(), level.end(), -1);
        queue<int> q;
        level[s] = 0;
        q.push(s);

        while (!q.empty() && level[t] == -1) {
            int v = q.front();
            q.pop();
            for (const Edge& e : graph[v]) {
                if (level[e.to] < 0 && e.capacity > e.flow) {
                    level[e.to] = level[v] + 1;
                    q.push(e.to);
                }
            }
        }
        return level[t] >= 0;
    }

    // DFS寻找增广路径
    int dfs(int v, int t, int flow) {
        if (v == t)
            return flow;
        for (int& i = iter[v]; i < (int)graph[v].size(); i++) {
            Edge& e = graph[v][i];
            if (level[e.to] == level[v] + 1 && e.capacity > e.flow) {
                int d = dfs(e.to, t, min(flow, e.capacity - e.flow));
                if (d > 0) {
                    e.flow += d;
                    graph[e.to][e.rev].flow -= d;
                    return d;
                }
            }
        }
        return 0;
    }

    // Dinic算法计算最大流
    int dinic(int s, int t) {
        if(s > n || t > n) {
            cerr << "Error: s or t exceed n" << endl;
            return -1;
        }
        int flow = 0;
        while (level_graph(s, t)) {
            fill(iter.begin(), iter.end(), 0);
            int f;
            while ((f = dfs(s, t, INF)) > 0) {
                flow += f;
            }
        }
        return flow;
    }
};

int main() {
    int n = 5, s = 1, t = 5;
    MaxFlow maxflow(n);
    maxflow.addEdge(1, 2, 1);
    maxflow.addEdge(1, 3, 2);
    maxflow.addEdge(1, 4, 3);
    maxflow.addEdge(2, 5, 3);
    maxflow.addEdge(3, 5, 2);
    maxflow.addEdge(4, 5, 1);
    maxflow.addEdge(2, 3, 2);

    // 计算最大流
    int result = maxflow.dinic(s, t);
    cout << result << endl;
    return 0;
}
```

#### 最小割问题（Min-cut Problem）

##### 概述

- **割的定义：** 对于一个流网络，**一个割** $cut(S,T)$ 是把所有顶点划分成两个集合 $S$ 和 $T$，并满足：源点 $s \in S$，汇点 $t \in T$。
- 割的**代价**定义为：$\sum_{u\in S,\ v\in T} c(u,v)$ ，也就是：所有**从 $S$ 指向 $T$** 的边的容量之和。
- **最大流与最小割：最大流的值等于最小割的值。**

##### 如何找到最小割？

1. 在流网络上先求出**最大流**。
2. 在**剩余网络**中，找出从源点 $s$ 出发**能够到达的所有顶点**。
3. “从 $s$ 可达的顶点”和“从 $s$ 不可达的顶点”这两部分就构成了一个**最小割**。

#### 多源点多汇点问题

##### 概述

- 存在多个源点 $s_1, s_2, \ldots, s_n$，以及多个汇点 $t_1, t_2, \ldots, t_m$。
- 流的大小定义为流入 $t_1, t_2, \ldots, t_m$ 的总流量之和。

##### 解决方法

- 新建一个**超级源点** $S$ 和一个**超级汇点** $T$。
- 用容量足够大的边（可视为无穷大）把 $S$ 连到所有原来的源点 $s_1,s_2,\ldots,s_n$。
- 再用容量足够大的边把所有原来的汇点 $t_1,t_2,\ldots,t_m$ 连到 $T$。

然后就把原问题转化成了一个普通的**单源单汇最大流**问题。

#### 点容量

##### 概述

顶点 $v$ 有一个容量 $c(v)$，满足：$\sum_{(u,v)\in E} f(u,v) \le c(v)$ 。

也就是说，**流入顶点 $v$ 的总流量不能超过这个点的容量**。

##### 解决方法

- 对于每个顶点 $v$，把它拆成两个点：
  - $v_{in}$
  - $v_{out}$
- 然后加一条边  $e=(v_{in},v_{out})$ ，并令这条边的容量为 $c(e)=c(v)$ 。

#### 二分图最大匹配（Maximum Bipartite Matching）

##### 概述

- **已知：**

  两组顶点，分别有 $n$ 个和 $m$ 个顶点。

  边只存在于这两组顶点之间。

   一条边 $(u,v)$ 表示：第 1 组中的顶点 $u$ 可以和第 2 组中的顶点 $v$ 匹配。

- **目标：**

  在 **“每个顶点最多只能匹配一次”** 的条件下，求**最大的匹配数**。

##### 如何转化成最大流？

- 建立超级源点 $s$ 和超级汇点 $t$。把源点与第 1 组连接，把第 2 组与汇点连接。

- 因为每个顶点最多只能匹配一次，所以：

  从源点连向左侧各点的边容量设为 1，从右侧各点连向汇点的边容量设为 1。

- 两组之间原来有边，就保留对应的边，这些边通常也设为容量 1。

##### 时间复杂度（Dinic 算法）

- 也叫 **Hopcroft–Karp algorithm**。

- 需要 $O(\sqrt{|V|})$ 轮迭代，而每一轮寻找一个阻塞流需要 $O(|E|)$ 时间。

  因此总时间复杂度是 $O(\sqrt{|V|}\,|E|)$ 。
