## 需求

最近做的项目中遇到一个 tab 页，里面有 5 个表格，每个表格都有它对应的分页器，分页器可以进行页码和页数的变化（如果数据足够多）如下图：![表格和分页器的tab页](https://user-gold-cdn.xitu.io/2018/11/17/1671f698223db178?w=944&h=357&f=png&s=19870)
表格＋分页器是很常见的需求，也很简单，大概的代码：

```
<el-table :data="handleData" border size="mini">
    <el-table-column type="index" label="序号" width="50"></el-table-column>
    <el-table-column prop="officeName" label="所在机构"></el-table-column>
    <el-table-column prop="userName" label="操作用户"></el-table-column>
    <el-table-column prop="operateFlag_text" label="操作"></el-table-column>
    <el-table-column prop="createDate_text" label="操作时间"></el-table-column>
    <el-table-column prop="operateResult_text" label="操作结果"></el-table-column>
 </el-table>
 <div class="paging">
   <el-pagination :current-page="pageNo" :page-size="pageSize" :page-sizes="[5,10, 25, 50]" layout="total,sizes, prev, pager, next, jumper" :total="count" @current-change="handleCurrentChange" @size-change="handleSizeChange">
    el-pagination>
  </div>
</el-tab-pane>
```

如果我们毫无追求，大可以写 5 个这样的代码段，表格倒是问题不大，最蠢的是分页器，我们需要写 5 个 pageSize 和 pageNo 以及操作他们的方法。为什么不能公用呢？因为有个需求，当我在“操作记录”中展示第二页的数据的时候点击了别的 tab 页，然后再次回到“操作记录”，需要回到跳转前的页码即第二页，如果我们用同一个参数，是无法保存 pageSize 和 pageNo 的，也就无法保持跳转后不变。  
所以我觉得参数必须要有 5 个，那怎么能减少我的工作量呢，毕竟写五个获取表格数据以及分页器的方法实在是工作量又大又让我觉得不甘心。但是我暂时想不到好的办法，只能先做了再说，首先是获取表格数据的方法：

```
    getTableData(useUrl, pageSize, pageNo) {
      const url = "/getTableData/" + useUrl;
      const params = {
        dataId: this.dataId,
        pageSize,
        pageNo
      };
      this.$axios.post(url, params).then(res => {
          this.handleData = []
          this.handleData.push(...res.data.resultData.list)
      });
    },
```

我们可以看到参数里有 useUrl、pageSize 和 pageNo，这些参数是每个表格独立的，这时候有个问题是我们怎么把从后台得到的数据传递给对应对象的 data。你大可以判断当前的 tab 是哪个，然后赋值给特定的 data。但是这样做貌似也有点蠢，然后我考虑数组是引用类型，如果我把数组传给了函数，在函数里改变对象的data那么我的原数组也会改变。所以我的代码改成了这样（部分和前面一样的代码省略了）

```
    getTableData(useUrl, pageSize, pageNo, data) {
       ...
        data = []
        data.push(...res.data.resultData.list)
      });
    },
```

但是我发现这样写并不能改变原数组的 data。原因就是我把数组赋空值的时候它的内存改变了（不直接赋值是因为 vue 有可能检测不到数组赋值而不更新dom。）所以我又改进了一下，最后的代码

```
    getTableData(useUrl, pageSize, pageNo, data) {
      const url = "/getTableData/" + useUrl;
      const params = {
        dataId: this.dataId,
        pageSize,
        pageNo
      };
      this.$axios.post(url, params).then(res => {
        data.splice(0, data.length, ...res.data.resultData.list);
        this.count = res.data.totalNum;
      });
    },
```

这样写过后就没问题了。然后该考虑什么时候去获取表格数据，最佳的应该是点击 tab 页的时候去获取，然后我就先写了如下代码(省略相似代码)

```
    tabClick(val) {
        switch(val.name){
            case "handle":
                let data = this.handleData
                let pageSize= this.handleSize
                let pageNo = this.handleNo
                this.getTableData("handleUrl",pageSize,pageNo)
            break;
            case "collect":
                let data = this.collectData
                let pageSize= this.collectSize
                let pageNo = this.collectNo
                this.getTableData("collectUrl",pageSize,pageNo)
            break;
            ...
            ...
            ...
        }
    }
```

直到这个时候我终于受不了了（然而实际上后面分页器的操作会更头大），为什么同样的东西要写5遍。这个时候我首先想到既然每个数组里都有 useUrl,pageSize 和 pageNo 以及数据 data，那我为什么不把他们写到一个对象里呢？起码这样我不用在 Vue 的 data 里写那么多独立的数据，于是我写了`handleObj:{useUrl:"handleUrl",data:[],pageSize:5,pageNo:1}`,这个时候我又想我应该给他取个名字，于是我又给它加了 name 属性。当我把五个对象都写出来的时候我又想如果我把这五个对象放到一起会怎样（其实一开始我完全是为了减少 data 对象的长度，但是写成对象数组让我的代码有了质的提升）。所以我的 data 对象里多了这么一个数组

```
      showArr: [
        {name: "handle",useUrl: "handleUrl",data: [],pageSize: 5,pageNo: 1},
        {name: "collect",useUrl: "collectUrl",data: [],pageSize: 5,pageNo: 1},
        {name: "apply",useUrl: "applyUrl",data: [],pageSize: 5,pageNo: 1},
        {name: "use",useUrl: "useUrl",data: [],pageSize: 5,pageNo: 1},
        {name: "evaluate",useUrl: "evaluateUrl",data: [],pageSize: 5,pageNo: 1},
      ]
```

这一步很关键，一个 tab 页是一个对象，而把他们的数据放在一个数组里能做很多事情。然后我注意到这个数组对象的 name 属性居然和我的 tab 的子页的 name 是一样的（虽然是巧合但是设置成一样确实可以省不少事）。于是我的 tabClick 代码变成了这样：

```
 tabClick(val) {
      let array = this.showArr;
        for (let index = 0; index < array.length; index++) {
         const element = array[index];
            if (element.name === val.name) {
             const data = element.data;
             const useUrl = element.useUrl;
             const pageSize = element.pageSize;
             const pageNo = element.pageNo;
             this.getTableData(useUrl, pageSize, pageNo, data);
             break;
        }
      }
 }
```

这时候我们终于看到这个对象数组的作用，终于不用写5句类似的代码了。顺带提一句其实数组遍历我平时用的最多的是 forEach，但是这里我需要 break，因为只要有 name 相等就没必要执行遍历了，而 forEach 是不能用 break 的。我们的分页器方法也可以写成类似的：

```
    handleCurrentChange(pageNo) {
    <!-- pageNo是回调参数，表示当前页码 -->
        const val = this.activeName;   //activeName就是当前tab的name
        let array = this.showArr;
        for (let index = 0; index < array.length; index++) {
         const element = array[index];
            if (element.name === val) {
             const data = element.data;
             const useUrl = element.useUrl;
             const pageSize = element.pageSize;
             const pageNo = pageNo;   //pageNo就不用对象默认的pageNo，而是用户操作改变的pageNo
             this.getTableData(useUrl, pageSize, pageNo, data);
             break;
        }
      }
    },
    handleSizeChange(pageSize) {
      const val = this.activeName;
      ...
      ...
         const pageSize = pageSize;
      ...
    },
```

这个时候我又发现为什么又写了这么多一样的代码，所以我又写了一个函数，这个函数首先必须知道当前 tab 页的 name，然后 pageSize 和 pageNo 是可改变的。代码：

```
    provideParams(name, size, num) {
      let array = this.showArr;
      for (let index = 0; index < array.length; index++) {
        const element = array[index];
        if (element.name === name) {
          const data = element.data;
          const useUrl = element.useUrl;
          const pageSize = size || element.pageSize;
          const pageNo =  num || element.pageNo;
          this.getTableData(useUrl, pageSize, pageNo, data);
          break;
        }
      }
    },
```

然后我把这个方法用到 tabClick 和分页器中，它基本把功能实现，但是大家有没有记得文章开始前的需求，我发现跳转后还是无法回到原来对应的页数，这是为什么呢？难道我这么长时间白折腾了？其实没有，我们忽略了一个问题，我们确实用handleSizeChange(pageSize)，把正确的页码传进函数里保证它的请求参数是对的，但是却没有把它赋值给数组对象，导致正确的页码没有被保存。于是这个函数最后的代码：

```
    provideParams(name, size, num) {
      let array = this.showArr;
      for (let index = 0; index < array.length; index++) {
        const element = array[index];
        if (element.name === name) {
          const data = element.data;
          const useUrl = element.useUrl;
          let pageSize,let pageNo;
          if (size) {
            pageSize = size;
            element.pageSize = size;
          } else {pageSize = element.pageSize}
          if (num) {
            pageNo = num;
            element.pageNo = num;
          } else {pageNo = element.pageNo}
          this.getTableData(useUrl, pageSize, pageNo, data);
          break;
        }
      }
    },
    tabClick(val) {
        this.provideParams(val.name)
    },
    handleCurrentChange(pageNo) {
      const name = this.activeName;
      this.provideParams(name, "", pageNo);
    },
    handleSizeChange(pageSize) {
      const name = this.activeName;
      this.provideParams(name, pageSize);
    },
    <!-- 不要忘记初始的时候需要让第一个表格获取数据 -->
    openDialog() {
      const val = this.activeName;
      this.provideParams(name);
    },
```
这个时候我觉得事情都做完了，我们应该成功了吧！我很想告诉你成功了，事实上我们的代码成功了百分之九十，但是我们的效果可能只有百分之十，因为最难处理的需求，页码保持不变的问题还是没有解决。我当时就跳起来了怎么可能。经过测试我神奇地发现其实表格的内容（即接进来的数据）实现了页码保持不变，也就是说pageNo保存起来了，但是分页器的样式却是一直从第一页开始，样式和数据并没有对应，也就是说是element-ui坑了我们。  
这也是我写这篇文章的目的之一，因为我（一个自学前端，参加工作半年不到的小菜鸟）希望看了文章的大牛们能给我点启发，同时我自己也会研究element-ui分页器的原理，完成最后一步，然后我会更新文章。  
也有人提醒我说可以把数组和分页器写进一个组件里，我所考虑的是数组的参数也就是（prop）是不一样的，如果我写进组件里，然后需要把这么多prop值传进去这是不是一个好的方法。
最后第一次写文章，肯定有许多有待改进的地方，希望大家多提意见，我会审视自己改正不足。