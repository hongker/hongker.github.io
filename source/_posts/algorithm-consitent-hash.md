---
title: Go数据结构与算法(13)-一致性哈希算法
date: 2022-11-06 22:26:27
tags: algorithm
---

在参加的某次架构师考试中，遇到了关于一致性哈希算法的问题.平时对哈希算法比较了解，并未深入了解一致性哈希,今天就来盘一盘。

## 什么是一致性哈希算法
>一致性哈希算法在1997年由麻省理工学院提出，是一种特殊的哈希算法，目的是解决分布式缓存的问题。 [1]  在移除或者添加一个服务器时，能够尽可能小地改变已存在的服务请求与处理请求服务器之间的映射关系。一致性哈希解决了简单哈希算法在分布式哈希表( Distributed Hash Table，DHT) 中存在的动态伸缩等问题。

## 与哈希算法的区别
- 哈希算法
使用简单的哈希函数: `m = hash(o) mod n`,其中，o为对象名称，n为机器的数量，m为机器编号。

因为对同一个关键字进行哈希计算，每次计算都是相同的值，这样就可以将某个 key 确定到一个节点了，可以满足分布式系统的负载均衡需求。

但如果节点数量发生了变化，也就是在对系统做扩容或者缩容时，必须迁移改变了映射关系的数据，否则会出现查询不到数据的问题。

- 一致性哈希算法
一致性hash算法正是为了解决此类问题的方法，它可以保证当机器增加或者减少时，节点之间的数据迁移只限于两个节点之间，不会造成全局的网络问题。

一致哈希算法也用了取模运算，但与哈希算法不同的是，哈希算法是对节点的数量进行取模运算，而一致哈希算法是对 2^32 进行取模运算，是一个固定的值。

最终我们可以把一致哈希算法是对 2^32 进行取模运算的结果值组织成一个圆环，被称为哈希环。
一致性哈希要进行两步哈希操作：
    - 1.对存储节点进行哈希计算，也就是对存储节点做哈希映射，比如根据节点的 IP 地址获机器的唯一名称进行哈希
    - 2.当对数据进行存储或访问时，对数据进行哈希映射
所以，一致性哈希是指将「存储节点」和「数据」都映射到一个首尾相连的哈希环上。
在对数据进行存取时，我们先对「数据」进行哈希映射，再根据结果值，往顺时针的方向遍历找到第一个「存储节点」,就用这个节点来存取数据。

所以在一致哈希算法中，如果增加或者移除一个节点，仅影响该节点在哈希环上顺时针相邻的后继节点，其它数据也不会受到影响。
<!--more-->

## Demo
- 哈希环
```go
package consistent

// HashRing 基于uint32类型的哈希环结构,主要用来比较key值
type HashRing []uint32

func (c HashRing) Len() int {
	return len(c)
}

func (c HashRing) Less(i, j int) bool {
	return c[i] < c[j]
}

func (c HashRing) Swap(i, j int) {
	c[i], c[j] = c[j], c[i]
}
```

- 机器节点
```go
package consistent

// Node 基于权重分配的机器节点结构
type Node struct {
	Id       int
	Ip       string
	Port     int
	HostName string
	Weight   int
}

func NewNode(id int, ip string, port int, name string, weight int) *Node {
	return &Node{
		Id:       id,
		Ip:       ip,
		Port:     port,
		HostName: name,
		Weight:   weight,
	}
}

// UUID 生成唯一ID
func (node *Node) UUID(i int) string {
	return node.Ip + "*" + strconv.Itoa(node.Weight) +
		"-" + strconv.Itoa(i) +
		"-" + strconv.Itoa(node.Id)
}
```

- 一致性哈希算法实现
```go
package consistent

const DEFAULT_REPLICAS = 60

type Consistent struct {
	mu        sync.RWMutex
	Nodes     map[uint32]Node
	numReps   int
	Resources map[int]bool
	ring      HashRing
}

func NewConsistent() *Consistent {
	return &Consistent{
		Nodes:     make(map[uint32]Node),
		numReps:   DEFAULT_REPLICAS,
		Resources: make(map[int]bool),
		ring:      HashRing{},
	}
}

// Add 添加机器节点
func (c *Consistent) Add(node *Node) bool {
	c.mu.Lock()
	defer c.mu.Unlock()

	if _, ok := c.Resources[node.Id]; ok {
		return false
	}

	// 根据分片数和节点的权重，分配多个key，实现按权重系数存取数据
	count := c.numReps * node.Weight
	for i := 0; i < count; i++ {
		str := node.UUID(i)
		c.Nodes[c.hashStr(str)] = *(node)
	}
	c.Resources[node.Id] = true
	c.sortHashRing()
	return true
}

// sortHashRing 对哈希环进行排序
func (c *Consistent) sortHashRing() {
	c.ring = HashRing{}
	for k := range c.Nodes {
		c.ring = append(c.ring, k)
	}
	sort.Sort(c.ring)
}

func (c *Consistent) hashStr(key string) uint32 {
	return crc32.ChecksumIEEE([]byte(key))
}

// Get 根据关键字查找对应的节点
func (c *Consistent) Get(key string) Node {
	c.mu.RLock()
	defer c.mu.RUnlock()

	hash := c.hashStr(key)
	i := c.search(hash)

	return c.Nodes[c.ring[i]]
}

// search 根据hash值按顺时针查询节点
func (c *Consistent) search(hash uint32) int {
	n := len(c.ring)
	i := sort.Search(n, func(i int) bool { return c.ring[i] >= hash })
	if i < n {
		return i
	} else {
		return 0
	}
}

// Remove 移除节点
func (c *Consistent) Remove(node *Node) {
	c.mu.Lock()
	defer c.mu.Unlock()

	if _, ok := c.Resources[node.Id]; !ok {
		return
	}

	delete(c.Resources, node.Id)

	// 移除节点也要根据权重将全部key移除掉
	count := c.numReps * node.Weight
	for i := 0; i < count; i++ {
		str := node.UUID(i)
		delete(c.Nodes, c.hashStr(str))
	}
	// 移除后需要重新排序
	c.sortHashRing()
}

```

- Test
```go
package consistent

func TestConsitent(t *testing.T) {
    con := NewConsistent()

	for i := 0; i < 10; i++ {
		si := fmt.Sprintf("%d", i)
		con.Add(NewNode(i, "172.18.1."+si, 8080, "host_"+si, 1))
	}

	for k, v := range con.Nodes {
		fmt.Println("Hash:", k, " IP:", v.Ip)
	}

	ipMap := make(map[string]int, 0)
	for i := 0; i < 1000; i++ {
		si := fmt.Sprintf("key%d", i)
		k := con.Get(si)
		if _, ok := ipMap[k.Ip]; ok {
			ipMap[k.Ip] += 1
		} else {
			ipMap[k.Ip] = 1
		}
	}

	for k, v := range ipMap {
		fmt.Println("Node IP:", k, " count:", v)
	}
}
```