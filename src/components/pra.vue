<template>
  <div class="pra" :class="{'cross': isCross}">    
    <mu-content-block>
      <mu-card>
        <mu-card-title title="页面置换算法" subTitle="Page replacement algorithm"/>
        <mu-switch label="横向排列" v-model="isCross" class="iscol"/>
        <mu-card-actions class="setNumber">
          <p>请选择算法: </p>
          <div class="select">
            <mu-radio label="FIFO算法" name="group" nativeValue="FIFO" v-model="algorithm" @change="FIFO"/>
            <mu-radio label="Optimal算法" name="group" nativeValue="OPT" v-model="algorithm" @change="Optimal"/>
            <mu-radio label="LRU算法" name="group" nativeValue="LRU" v-model="algorithm" @change="LRU"/>
          </div>
          <p>请选择最小物理块数: {{M}}</p>
          <mu-slider v-model="M" :step="1" :max="BlockNum" @change="setMNumber" @input="setMNumber" />
          <p>请选择页面个数: {{N}}</p>
          <mu-slider v-model="N" :step="1" :max="DataMax" @change="setPageNumber" @input="setPageNumber"/> 
          <p>请输入页面访问序列: </p>
          <div class="inputData">
            <mu-flexbox>
              <mu-flexbox-item>
               <mu-text-field  type="number" v-for="index in dataArray" v-model="index.data" :key="index.index" @input="workout"/>
              </mu-flexbox-item>
            </mu-flexbox>            
          </div>
          <p v-if="dataArray.length != 0" flex-justify-space-around>
            <mu-raised-button label="随机生成" icon="cached" primary @click="autoGen"/>
            <mu-raised-button label="测试数据" icon="build" primary @click="test"/>
            <mu-raised-button label="计算结果" icon="send" primary @click="workout"/>
          </p>
        </mu-card-actions>
      </mu-card>
    </mu-content-block>
    <mu-content-block>
      <mu-card class="output">
        <mu-card-title subTitle="算法过程"/>        
        <mu-card-actions class="tableContenter">
          <table class="table" cellpadding="0" cellspacing="0">
            <tr>
              <td>访问页</td>
              <td v-for="(data, index) in dataArray" :key="index" class="dataIndex">{{data.data}}</td>
            </tr>
          </table> 
          <table class="table" cellpadding="0" cellspacing="0">
            <tr v-for="(m, mIndex) in Marray" :key="mIndex">
              <td>物理块{{m.index+1}}</td>
              <td v-for="(page, pIndex) in pageArray" :key="pIndex">{{matrix[m.index][page.index]}}</td>
            </tr>
            <tr>
              <td>缺页否</td>
              <td v-for="(page, index) in pageArray" :key="index">{{page.data}}</td>
            </tr>
          </table>      
        </mu-card-actions>
      </mu-card>
    </mu-content-block>

    <mu-content-block>
      <mu-card class="output">
        <mu-card-title subTitle="输出结果"/>
        <mu-card-actions>        
          <p>缺页次数: {{ChangeTimes}}</p>
          <p>缺页率: {{ChangeTimes*100/N}}%</p>
        </mu-card-actions>
      </mu-card>
    </mu-content-block>
    <mu-snackbar v-if="snackbar.is" :message="snackbar.message" action="关闭" @actionClick="hideSnackbar" @close="hideSnackbar"/>
  </div>
</template>

<script>
export default {
  name: 'index',
  data () {
    return {
      algorithm: 'FIFO',//默认算法
      DataMax: 100,//最大页面数
      BlockNum: 10,//最大物理块数
      DataShow: [],//要显示的数据
      DataShowEnable: [],//标记该数据是否显示
      Data: [],//数据
      Block: [],//物理块
      count: [],//计数器
      N: 1,//页面个数
      M: 1,//最小物理块
      ChangeTimes: 0,//缺页次数
      dataArray: [],//界面显示数据数组      
      Marray: [],//界面显示物理块数组
      pageArray: [],//界面显示页数组记并录是否缺页      
      isCross: false,//是否横向排版
      matrix: [],//界面最终显示算法过程的矩阵
      snackbar: {
        is: false,
        message: ''
      }
    }
  },
  methods: {
    initMatrix () {
      //初始化数组
      let matrixData = new Array(this.BlockNum)
      let matrixBool = new Array(this.BlockNum)
      let matrixNbsp = new Array(this.BlockNum)
      for (let i = 0; i < this.BlockNum; i++) {
        matrixData[i] = new Array(this.DataMax)
        matrixBool[i] = new Array(this.DataMax)
        matrixNbsp[i] = new Array(this.DataMax)
        for (let j = 0; j < this.DataMax; j++) {
          matrixData[i][j] = 0
          matrixBool[i][j] = false          
          matrixNbsp[i][j] = ' '
        }
      }
      this.matrix = matrixNbsp
      this.DataShow = matrixData
      this.DataShowEnable = matrixBool

      let tempArraycount = new Array(this.BlockNum);
      let tempArrayblock = new Array(this.BlockNum);
      for (let i = 0; i < this.BlockNum; i++) {
        tempArrayblock[i] = 0
        tempArraycount[i] = 0
      }
      this.Block = tempArraycount
      this.count = tempArrayblock
    },
    FIFO () {      
      let i = 0 ,j = 0
      let find = false
      let point = 0
      let temp = 0
      let tempData = []

      this.ChangeTimes = 0
      //缺页次数清零

      for (i = 0; i < this.dataArray.length; i++) {
        tempData[i] = this.dataArray[i].data
        //从界面接收访问序列
        this.pageArray[i] = {index : i, data: '√'}
        //初始化是否缺页标记为全部缺页
      }
      this.Data = tempData
      //存储访问序列

      for( j = 0; j < this.M; j++) {        
        for( i = 0; i < this.N; i++) {
          this.DataShowEnable[j][i] = false
          //初始化为false，表示没有要显示的数据
        }
      }

      for( i = 0; i < this.M; i++){
        this.count[i] = 0
        //计数器清零
        this.Block[i] = 0
        //初始化物理块
      }

      for(i = 0;i < this.N; i++){
        //遍历所有访问序列
        for(j = 0;j < this.M;j++) {
          this.count[j]++
          //计数器增加
        }
        find = false;
        //表示块中有没有该数据
        for(j = 0;j < this.M;j++) {
          //遍历所有物理块
          if( this.Block[j] === this.Data[i] ) {            
            find = true
            //该访问序列已经在物理块中
          }
        }
        if( find ) {          
          this.pageArray[i].data = ''
          //若该访问序列已经在物理块中，则改正缺页标记为空
          continue
          //继续遍历下一个访问序列
        }
        this.ChangeTimes++
        //若该访问序列不在物理块中，则缺页次数自增
        if( (i+1) > this.M ) { 
          //获得要替换的块指针
          temp = 0
          for(j = 0; j < this.M;j++) {            
            if( temp < this.count[j] ) {
             temp = this.count[j];
             point = j
             //获得离的最远的指针
            }
          }
        } else {
          point = i          
        }
        this.Block[point] = this.Data[i];
        //替换
        this.count[point] = 0
        //更新计数值
        for(j = 0;j < this.M; j++) {
          //保存要显示的数据
          this.DataShow[j][i] = this.Block[j]
          this.DataShowEnable[ i < this.M ? ( j <= i ? j : i) : j][i] = true
          //设置显示数据
        }
      }
      if(this.dataArray.length != 0){
        this.showSnackbar('计算已完成', 2000)
      }
      this.output()
      //输出到界面
    },    
    Optimal () {
      let i = 0 ,j = 0, k
      let find = false
      let point = 0
      let temp = 0
      //临时变量，比较离的最远的时候用      
      let tempData = []
      this.ChangeTimes = 0
      //缺页次数清零

      for (i = 0; i < this.dataArray.length; i++) {
        tempData[i] = this.dataArray[i].data
        //从界面接收访问序列
        this.pageArray[i] = {index : i, data: '√'}
        //初始化是否缺页标记为全部缺页
      }
      this.Data = tempData
      //存储访问序列

      for( j = 0; j < this.M; j++) {        
        for( i = 0; i < this.N; i++) {
          this.DataShowEnable[j][i] = false
          //初始化为false，表示没有要显示的数据
        }
      }

      for( i = 0; i < this.M; i++){
        this.Block[i] = 0
        //初始化物理块
      }

      for(i = 0;i < this.N;i++) {
        //遍历所有访问序列
        find = false
        //表示块中有没有该数据
        for(j = 0;j < this.M;j++) {
          //遍历所有物理块
          if( this.Block[j] == this.Data[i] ) find = true
          //该访问序列已经在物理块中
        }
        if( find ) {          
          this.pageArray[i].data = ''
          //若该访问序列已经在物理块中，则改正缺页标记为空
          continue
          //继续遍历下一个访问序列
        }   
        this.ChangeTimes++
        //若该访问序列不在物理块中，则缺页次数自增
        for(j = 0;j < this.M;j++) {
          //找到下一个值的位置
          find = false
          for( k = i;k < this.N;k++) {
            if( this.Block[j] == this.Data[k] ) {
              find = true
              this.count[j] = k
              break
            }
          }
          if( !find ) this.count[j] = this.N
        }
        if( (i+1) > this.M ) {
        //获得要替换的块指针
         temp = 0;
          for(j = 0;j <this.M; j++) {
            if( temp < this.count[j] ) {
              temp = this.count[j]
              point = j
            }
          }
        } else {
          point = i
        }      
        this.Block[point] = this.Data[i];
        //替换
        for(j = 0;j < this.M; j++) {
          //保存要显示的数据
          this.DataShow[j][i] = this.Block[j]
          this.DataShowEnable[ i < this.M ? ( j <= i ? j : i) : j][i] = true
          //设置显示数据
        }
      }
      if(this.dataArray.length != 0){
        this.showSnackbar('计算已完成', 2000)
      }
      this.output()
      //输出到界面
    },
    LRU () {
      let i = 0 ,j = 0
      let find = false
      let point = 0
      let temp = 0
      let tempData = []

      this.ChangeTimes = 0
      //缺页次数清零

      for (i = 0; i < this.dataArray.length; i++) {
        tempData[i] = this.dataArray[i].data
        //从界面接收访问序列
        this.pageArray[i] = {index : i, data: '√'}
        //初始化是否缺页标记为全部缺页
      }
      this.Data = tempData
      //存储访问序列

      for( j = 0; j < this.M; j++) {        
        for( i = 0; i < this.N; i++) {
          this.DataShowEnable[j][i] = false
          //初始化为false，表示没有要显示的数据
        }
      }

      for( i = 0; i < this.M; i++){
        this.count[i] = 0
        //计数器清零
        this.Block[i] = 0
        //初始化物理块
      }

      for(i=0;i<this.N;i++) {
        //遍历所有访问序列
        for(j=0;j<this.M;j++) {
          this.count[j]++
          //计数器自增
        }          
        find = false;
        //表示块中有没有该数据
        for(j=0;j<this.M;j++) {
          if( this.Block[j] == this.Data[i] ) {
            //该访问序列已经在物理块中
            this.count[j] = 0
            find = true
          }
        }
        if( find ) {          
          this.pageArray[i].data = ''
          //若该访问序列已经在物理块中，则改正缺页标记为空
          continue
          //继续遍历下一个访问序列
        }      
        this.ChangeTimes++;
        //若该访问序列不在物理块中，则缺页次数自增
        if( (i+1) > this.M ) {       
          temp = 0;
          for(j=0;j<this.M;j++) {
            if( temp < this.count[j] ) {
              temp = this.count[j]
              point = j
              //获得离的最远的指针
            }
          }
        } else {
          point = i
        }      
        this.Block[point] = this.Data[i];
        //替换
        this.count[point] = 0;
        //更新计数值
        for(j = 0;j < this.M; j++) {
          //保存要显示的数据
          this.DataShow[j][i] = this.Block[j]
          this.DataShowEnable[ i < this.M ? ( j <= i ? j : i) : j][i] = true
          //设置显示数据
        }
      }
      if(this.dataArray.length != 0){
        this.showSnackbar('计算已完成', 2000)
      }
      this.output()
      //输出到界面
    },
    output() {
      //输出到界面
      let i,j;
      for(j = 0; j < this.M; j++) {        
        for(i = 0; i <this.N; i++) {
          if(this.DataShowEnable[j][i]) {
            this.matrix[j][i] = this.DataShow[j][i]
          } else {
            this.matrix[j][i] = " "
          }          
        }        
      }
    },
    setPageNumber () {
      //设置界面显示页面访问序列和是否缺页标记
      this.dataArray = []
      this.pageArray = []         
      for (let i = 0; i < this.N; i++) {
        this.dataArray[i] = {index : i, data: null}
        this.pageArray[i] = {index : i, data: '√'}
      }      
    },
    setMNumber () {
      //设置页面物理块数
      this.Marray = []     
      for (let i = 0; i < this.M; i++) {
        this.Marray[i] = {index : i, data: null}
      }      
    },
    autoGen () {
      //随机生成访问序列
      this.showSnackbar('随机生成序列已完成', 2000)
      for (let i = 0; i < this.dataArray.length; i++) {
        this.dataArray[i] = {index : i, data: Math.ceil(Math.random()*9)}
      }      
    },
    test () {
      //快速设置为指导书中测试访问序列
      this.showSnackbar('已设置为测试数据', 2000)
      this.M = 3
      this.N = 20
      let testData = [4,3,2,1,4,3,5,4,3,2,1,5,6,2,3,7,1,2,6,1]      
      for (let i = 0; i < 20; i++) {
        this.dataArray[i] = {index : i, data: testData[i]}
      }
    },
    workout () {
      //计算结果      
      switch(this.algorithm) {
        case 'FIFO': this.FIFO()
          break
        case 'OPT': this.Optimal()
          break
        case 'LRU': this.LRU()
          break
      }
    },
    showSnackbar (msg, time) {
      //显示提示框      
      this.snackbar.message = msg
      this.snackbar.is = true
      if (this.snackTimer) clearTimeout(this.snackTimer)
      this.snackTimer = setTimeout(() => { this.snackbar.is = false }, time)
    },
    hideSnackbar () {
      //关闭提示框
      this.snackbar.is = false
      if (this.snackTimer) clearTimeout(this.snackTimer)
    }    
  },
  created () {
    this.initMatrix()
    //初始化数组
  }
}
</script>

<style scoped>
.pra{
  max-width: 75%;
  min-width: 400px;
  margin: 30px auto;
}
.setNumber{
  padding: 4%;
}
.mu-text-field{
  max-width: 62px;
  margin: 1rem;
}
.select{
  margin:20px 0px;
}
.mu-radio{
  margin-right: 3rem;
}
.output{
  padding: 4%;
}
.table{
  border: 1px solid #000;
  margin-bottom: 10px;
}
.table >tr>td {
  border: 1px solid #000;
  width: 2em;
  height: 2em;
  text-align: center;
  font-size: 1em;
}

.table >tr>td:nth-child(1){
   width: 5em;
}

.tableContenter{
  overflow-x: auto; 
}
.cross{
  min-width: 100%;
}
.cross>.mu-content-block{
  max-width: 50%;
  display: inline-block;
}
.cross>.mu-content-block:nth-child(1),
.cross>.mu-content-block:nth-child(2){
  max-width: 50%;  
  float: left;
}
.iscol{
  margin-left: 1em;
}
p[flex-justify-space-around] {
  display: flex;
  justify-content: space-around;
}
</style>
