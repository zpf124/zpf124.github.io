---
title: Greenfoot游戏——推箱子
date: 2016-01-09 12:03:15
categories: 
 - Development
 - Java
tags:
 - Java
 - Game
 - Greenfoot
---

***作为一个新人，第一次写博客，文章的调理以及结构可能不够清晰，如果有哪里说的不清楚欢迎各位提醒我。***

*本应用内的**素材剽窃于**此[html5 游戏][1]中，写此文前未得到原作者授权允许，如果涉及侵权，烦劳原作者联系我，立即删除。*
<http://blog.csdn.net/lufy_legend/article/details/8607933>

----------

# 前言
`Greenfoot`： Greenfoot是一个基于java的小型框架，可以简易的实现基于GUI界面的游戏应用，所以一般被当作是Java教学用的软件框架。
[官方网站][2] [中文API][3] (为2.5版本，略旧最新API为3.0)

`注意`：虽然Greenfoot一般是被当作学习Java用的练习框架/工具，但请先掌握基本的`Java语法`后再来阅读本文。

<!-- more  -->

# 正文
好了终于要是真正制作这个推箱子的游戏了。在此之前需要先准备好相应的素材,各位可以直接从我上传的 [源代码][4] 中提取。

## 构思基本结构
首先，我们要想清楚，我们这个游戏大概需要哪些东西。
- 地图：我们的游戏不能只有一关，所以我们需要想办法记录每一关的地图数据。
- 可操作的人物：游戏自然要有个我们可以操作的对象。
- 箱子：既然是推箱子的游戏必然不能少箱子。
- 目标地：箱子需要推到指定的目标点。
- 障碍物：我们需要一些障碍物来限制人物的运动。

之后我们先把这几个类建立好了。

![项目基本结构][5]

其中为了美观，我专门单独做了一个地板`Floor`，不做采用地图背景色也是可行的。


在这之前我们需要先规定好一个基本的世界类。
我们的图片都是48*48的所以世界的单位长度最好直接设置成48（像素）。
宽高呢这时可以随意指定的，我这里指定为宽`11`高`9`，现在世界就是一个由 11*9 个边长为 48像素 的小格子拼起来的。
所以我GameWorld代码如下。

```java
public class GameWorld extends World
{
    public GameWorld()
    {    
        super(11, 9, 48);
    }
}
```

## 制作存储地图的类

对地图的储存我选择使用一个二维数组来存储一个关卡。

像这样 map中包含 9 个一维数组，每个一维数组里有 11 个数字，二维数组里的每个数字对应一个游戏界面上的点。
当然，如果你上面设定的世界宽高和我不一样的，那么这里也要进行对应的变化。
```java
int[][] map = 	{
    {-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1},  
    {-1,-1, 1, 1, 1, 1, 1, 1,-1,-1,-1},  
    {-1,-1, 1, 0, 0, 0, 0, 1, 1,-1,-1},  
    {-1,-1, 1, 2, 2, 6, 0, 0, 1,-1,-1},  
    {-1,-1, 1, 4, 4, 3, 4, 4, 1,-1,-1},  
    {-1,-1, 1, 0, 0, 6, 2, 2, 1,-1,-1},  
    {-1,-1, 1, 1, 0, 0, 0, 0, 1,-1,-1},  
    {-1,-1,-1, 1, 1, 1, 1, 1, 1,-1,-1},  
    {-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1}   
}
```

但是，我们的游戏不能只有一关吧，所以我们需要在来一个数组将多个地图存起来。

	private static List<int[][]> maps;

为了以后的扩展，方便给，我这里选中使用java.util.List类来存地图数据，继续用数组也是可以的。

然后我选择在静态代码块中初始化数据，这样当类初次加载的时候数据就会存在。

```java
static {
    maps = new ArrayList<int[][]>();
    
    maps.add(new int[][]{
        {-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1},  
        {-1,-1, 1, 1, 1, 1, 1, 1, 1,-1,-1},  
        {-1,-1, 1, 0, 0, 4, 0, 0, 1,-1,-1},  
        {-1,-1, 1, 0, 0, 2, 0, 0, 1,-1,-1},  
        {-1,-1, 1, 4, 2, 3, 2, 4, 1,-1,-1},  
        {-1,-1, 1, 0, 0, 2, 0, 0, 1,-1,-1},  
        {-1,-1, 1, 0, 0, 4, 0, 0, 1,-1,-1},  
        {-1,-1, 1, 1, 1, 1, 1, 1, 1,-1,-1},  
        {-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1}  
    });
    .......
}
```

你会发现，咦？那个数组里存的数字怎么有好几种啊，都是什么意思啊？
在这里，我提前定义好了不同的类型去表示不同的物体。
	

```java
    public static final int TYPE_EMPTY = -1;                // 空白
    public static final int TYPE_FLOOR = 0;                 // 地板
    public static final int TYPE_WAll = 1;                  // 墙壁
    public static final int TYPE_BOX = 2;                   // 箱子
    public static final int TYPE_PERSON = 3;                // 人物
    public static final int TYPE_TARGET = 4;                // 目标点（可与箱子或者人重合，以加和表示）
```
这回你知道这些数字代表什么意思了吧，那我需要提问了**最上面那个数组里的6代表什么呢**？

对，如果你仔细看过上面几行你就会明白6代表4+2也就是箱子和目标点都存在一个位置上。

接下来我们需要制作一个读取地图方法。
```java
	/**
     * 方法需要传入一个world指定的数据要加载到那个world上，以及一个levle表示要加载的关卡。
     * @param world 需要加载数据的world
     * @param level 指定加载的关卡
     */
	public static void loadMap(World world, int level){
        int[][] map = maps.get(level);
        for (int i = 0; i < map.length; i++) {
            for (int j = 0; j < map[i].length; j++) {
	            // 不论这个位置是什么，只要不是空白就需要铺一个地板
                if(map[i][j] != MapData.TYPE_EMPTY){
                    world.addObject(new Floor(), j, i);
                }
                // 根据不同的类型创建不同的物体
                switch (map[i][j]) {
                    case MapData.TYPE_WAll:
                        world.addObject(new Wall(), j, i);
                        break;
                    case MapData.TYPE_BOX:
                        world.addObject(new Box(), j, i);
                        break;
                    case MapData.TYPE_TARGET:
                        world.addObject(new Target(), j, i);
                        break;
                    case MapData.TYPE_PERSON:
	                    // 此处为了保证一个地图上只有一个可控制对象，使用单例，具体内容我们到person中再讲
                        world.addObject(Person.getInstance(), j, i);
                        break;
                    case MapData.TYPE_BOX +  MapData.TYPE_TARGET:
                        world.addObject(new Box(), j, i);
                        world.addObject(new Target(), j, i);
                        break;
                    case MapData.TYPE_PERSON +  MapData.TYPE_TARGET:
                        world.addObject(Person.getInstance(), j, i);
                        world.addObject(new Target(), j, i);
                        break;
                }
            }
        }
    }
```

这样地图类就基本完成了。后续内容稍后更新....。




[1]: http://download.csdn.net/download/zpf124/9396945 "素材来源的html5游戏"
[2]: http://www.greenfoot.org/ "官方网站"
[3]: http://www.greenfoot.org/files/translations/Chinese/API/ "中文API"
[4]: http://download.csdn.net/download/zpf124/9396945 "推箱子源代码"
[5]: /assets/images/20160102223628753.png "项目基本结构"