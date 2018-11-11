# 信息检索

## 1 介绍

本次作业基于第一次作业布尔检索，环境配置与作业1相同。在布尔检索项目上新增或更改了以下功能：

1. 新增向量空间模型，为通过关键字查询得出的结果进行排序。
    1. 在本功能中，不允许完全匹配的查询方式，如果查询时选中完全查询则会给出警告信息。
    2. 本功能下，布尔查询原有的排序方法不再使用，故在页面中采取了隐藏处理
    3. 更改显示方式中的显示条目为显示top-k条查询结果。
    4. 为了更便于统计结果的正确性，修改了数据库对应字段的内容，将其全部更换为英文。
2. 更改布尔查询中的部分功能。
    1. 将显示方式中的分页机制更新为向量空间模型的top-k显示，所以去除了分页机制
    2. 修改info信息，在没有向input框中输入关键词是，将会给出提示信息
    3. 修改动画机制，目前支持浏览器chrome，edge,但是效果在edge上最好，在chrome上有少许的卡顿。
    4. 布尔查询下如果未选中排序方式，又使用了升序或降序排序，将会给出警告信息。

## 2 向量空间模型的实现

*注意：fnlp的模型文件地址，即语句：`CNFactory factory = CNFactory.getInstance("E:\\工作 学习\\大三上\\信息检索\\作业\\1.4\\fnlp\\models");`中的地址需要更换为本地地址*

以下为后台实现的部分代码（由于原理类似,此处给出BrandIndexController中关键部分代码）：

```Java
    @RequestMapping(value = "OrderDocByVectorModel", method = RequestMethod.GET)
    @ResponseBody
    public List<ProductDetails> VectorSort(String filed,int resultNum) throws LoadModelException {

        //查询中的某个词项出现在了n个文档中。
        CNFactory factory = CNFactory.getInstance("E:\\工作 学习\\大三上\\信息检索\\作业\\1.4\\fnlp\\models");

        String[] wordOris = factory.seg(filed);
        int wordOrder = 0;//记录查询中词项的位置

        int documentNum = productDetailsBiz.findAll().size();;//所有的文档的数目
        List<ProductDetails> productDetails = productDetailsBiz.findAll();

        List<String> words = new ArrayList<String>();
        for(String word : wordOris){
            if(!words.contains(word)){
                words.add(word);
            }
        }
        Double [][]tfIdfList = new Double[words.size()][documentNum];//一个二维数组，储存的是查询中的词项在所有文档中的权重，最后要利用这个结果计算出所有文档的权重的排序。

        for(String word : words){
            Double letterTF;
            List<BrandIndex> brandIndices = brandIndexBiz.findByFiled(word);
            if(brandIndices.size() != 0 ) {
                letterTF = log10(documentNum / brandIndices.size());
            }
            else
                letterTF = 0.0;
            //至此，查询中词项的idf计算完成，接下来需要计算词项的tdf，进而利用空间向量模型求出各文档权重
            for(int i = 0; i < documentNum; i++){
                BrandIndex brandIndexTemp = brandIndexBiz.findByFiledAndId(word,i+1);
                int tfRaw = 0;
                Double nLized = 0.0;

                for(String wordTmp :wordOris){
                    if(wordTmp.equals(word))
                        tfRaw++;
                }
                Double tfWght = 0.0;
                if(tfRaw != 0)
                    tfWght = log10(tfRaw) + 1;  //计算词项在查询中的tdf
                if(brandIndexTemp != null) {
                    nLized = brandIndexTemp.getWtd();//词项在文档中的tdf
                }

                tfIdfList[wordOrder][i] = nLized * tfWght * letterTF; //权重计算，完成对文档的评分。
            }
            wordOrder++;
        }

        Double []docWeight = new Double[documentNum];
        for(int i = 0; i < documentNum; i++){
            docWeight[i] = 0.0;
            for(int j = 0; j < words.size(); j++){
                docWeight[i] += tfIdfList[j][i];
            }
        }

        NewType []newType = new NewType[documentNum];

        int count = 0;
        for(ProductDetails productDetails1 : productDetails){
            newType[count] = new NewType();//开辟一个空间
            newType[count].setWeight(docWeight[count]);
            newType[count].setProductDetailsB(productDetails1);
            count++;
        }

        return HeapSort(newType,resultNum);
    }


    private List<ProductDetails> HeapSort(NewType[] newTypes, int resultNum) {
        List<ProductDetails> result = new ArrayList<ProductDetails>(resultNum);

        //下述过程完成堆排序，并将堆排序结果存储在result中。
        //如果结果权重为0或者查询所需的长度已经达到最大，则停止操作
        initBuildHeap(newTypes, 0, newTypes.length - 1);
        for (int i = newTypes.length - 1; i >= 0; i--) {
            swap(newTypes, 0, i);
            rebuildHeap(newTypes, 0, i);
            if(newTypes[i].getWeight() != 0 && result.size() < resultNum) {
                result.add(newTypes[i].getProductDetailsB());
                System.out.println("权重：" +  newTypes[i].getWeight() +  "  商品名：" + newTypes[i].getProductDetailsB().getGood().getGoodName());
            }
        }
        return result;
    }

    //构建初始化大根堆 此步骤时间复杂度是：N（logN）
    private static void initBuildHeap(NewType[] newTypes, int index, int end) {

        if (newTypes == null || index > end) {
            return;
        }
        for (int i = end; i >= index; i--) {
            int parent = (i - 1) / 2;
            if (newTypes[i].getWeight() > newTypes[parent].getWeight()) {
                swap(newTypes, i, parent);
                initBuildHeap(newTypes, index, i);
            }
        }
    }

    //维护新的大根堆结构
    private static void rebuildHeap(NewType[] newTypes, int index, int end) {
        if (newTypes == null || index > end) {
            return;
        }
        int left = 2 * index + 1;
        int right = 2 * index + 2;
        if (left < end && newTypes[index].getWeight() < newTypes[left].getWeight()) {
            swap(newTypes, index, left);
            rebuildHeap(newTypes, left, end);
        }
        if (right < end && newTypes[index].getWeight() < newTypes[right].getWeight()) {
            swap(newTypes, index, right);
            rebuildHeap(newTypes, right, end);
        }
    }

    //交互数组中 left 和 right 位置
    private static void swap(NewType[] newTypes, int left, int right) {
        NewType temp = newTypes[left];
        newTypes[left] = newTypes[right];
        newTypes[right] = temp;
    }



//定义一个类，用来存放返回的数据和其对应的权重
    public static class NewType{
        private Double weight;
        private ProductDetails productDetails;

        public Double getWeight() {
            return weight;
        }

        public void setWeight(Double weight) {
            this.weight = weight;
        }

        public ProductDetails getProductDetailsB() {
            return productDetails;
        }

        public void setProductDetailsB(ProductDetails productDetails) {
            this.productDetails = productDetails;
        }
    }

```

以上代码各部分的功能都已经在注释中给出，此处简要的说明整体的思路：

对于文档和查询采用不同的权重计算机制。

```txt
文档：对数tf, 无idf因子， 余弦长度归一化
查询：对数tf, idf，无归一化
```

如下图所示：
![image](./img/图片1.png)

将查询表示成为tf-idf权重向量，文档表示为同一空间下的tf-idf向量，然后计算两个向量的余弦相似度，最后按照相似度大小将文档进行排序，并将top-k返回给用户。此处采用堆排序实现文档的排序。

## 运行结果及截图

1. 在商品品牌，关键字检索；向量空间模型条件下，输入`Lenovo little new` ，结果如下图：
    ![image](./img/商品品牌的检索实例.png)
各查询答案对应的权重通过控制台输出，结果如下图：

    ![image](./img/Levono.JPG)

2. 在商品描述，关键字检索；向量空间模型, 显示前6条 条件下，输入`beautiful good`, 结果如下图：

    ![image](./img/按照商品描述结果.png)

    查询答案（包括未输出的答案）对应的权重通过控制台输出，结果如下图所示：

    ![image](./img/beautifulgood.JPG)

3. 布尔查询的各情况均可以与作业1中一样实现。

## 总结

本次作业中最大的问题在于如何高效的对查询出的结果计算权重，并快速的进行排序，以满足用户的需求。所以，对权重的计算，在预处理阶段直接计算了所有文档的归一化后的对数tf; 对于结果的排序，则采用了堆排序的方式，因为堆排序在最坏的情况下复杂度为O(nlogn), 对比快速排序法在最坏情况下的O(n^2)，且考虑到返回top-k项的情况，堆排序要更优于其他的排序方式。

所做的不足的地方是由于在之前的项目上进行增加更改功能，用了大量的条件语句，项目代码结构变得有些混乱。。之后的作业中将会更加注重整体的规划，使代码结构更加的明晰。
