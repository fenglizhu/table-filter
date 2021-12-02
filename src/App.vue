<template>
  <div class="title"><h2>表格数据筛选</h2></div>
  <div class="filter-content">
    <div class="filter-item" v-for="(val, key, index) in state.allfilterMap" :key="index">
      <div class="label">{{key}}</div>
      <ul class="filter-dowm">
        <li v-for="(item,indexs) in val.data" :key="indexs" :class="[{selected:item.status }]" @click="selectItem(item, val)">{{item.name}} </li>
      </ul>
    </div>
  </div>
  <div class="button-group">
    <div class="json-tip">导入JSON格式：{ data: [ { } ] }</div>
    <button @click="csvToTable" class="import">导入数据</button>
    <button @click="tableToCsv" class="exprot">导出数据</button>
    <input type="file" ref="csvFile" @change="changeFile" hidden>
  </div>
  <table class="table">
    <thead>
      <tr>
        <th v-for="(val, key, ind) in state.allfilterMap" :key="ind">{{key}}</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(item,index) in resultData" :key="index">
        <td v-for="(val, key, ind) in state.allfilterMap" :key="ind">{{item[key]}}</td>
      </tr>
    </tbody>
  </table>
  
</template>
<script lang="ts">
import { defineComponent, onMounted, reactive, ref } from 'vue';
export default defineComponent({
  name: 'App',
  setup() {

    onMounted(()=>{
      initData()
    })
    // 精确查询多条件并集查询，经常用到的是表格
    let searchInfo: any = {};
    // 过滤集合，为了后面可以更改数据
    let state:any = reactive({
      allfilterMap: {}
    });

    // 最终呈现的数据
    let resultData:any = ref([]);

    // 文件dom
    let csvFile = ref<HTMLElement | null>(null);

    // 数据
    let data:any[] = [
      {name:'刘备', book: '三国演义', gender: '男'},
      {name:'贾宝玉', book: '红楼梦', gender: '男'},
      {name:'武松', book: '水浒传', gender: '男'},
      {name:'林黛玉', book: '红楼梦', gender: '女'},
      {name:'史湘云', book: '红楼梦', gender: '女'},
      {name:'孙悟空', book: '西游记', gender: '男'},
      {name:'甘昭烈', book: '三国演义', gender: '女'},
      {name:'贾雨村', book: '红楼梦', gender: '男'},
      {name:'宋江', book: '水浒传', gender: '男'},
      {name:'关羽', book: '三国演义', gender: '男'},
      {name:'赵云', book: '三国演义', gender: '男'},
      {name:'沙悟净', book: '西游记', gender: '男'},
      {name:'唐僧', book: '西游记', gender: '男'},
      {name:'孙权', book: '三国演义', gender: '男'},
    ]
    
    let initData = () =>{
      console.log(state.allfilterMap);
      
      data.forEach(element => {
        for (let key in element) {
          if(!state.allfilterMap[key]){
            state.allfilterMap[key] = {
              selectName: ['全部'],
              data: [{name: '全部', status: true}]
            }
          }
          let flag: boolean = false;
          for (let i = 0; i < state.allfilterMap[key].data.length; i++) {
            if(state.allfilterMap[key].data[i].name == element[key]) {
              flag = true;
            }
          }
          if(!flag) {
            state.allfilterMap[key].data.push({name: element[key], status: true})
            state.allfilterMap[key].selectName.push(element[key])
          }
        }
      });

      for (const key in state.allfilterMap) {
        searchInfo[key] = [...state.allfilterMap[key].selectName]
      }

      resultData.value = data;
    }
    

    let filterDataFnc = () => {
      var filterKeys = Object.keys(searchInfo);
      let filterData = data.filter((item) =>{
        return filterKeys.every((key) =>{
          if(!searchInfo[key].length){
            return false;
          }
          if(!item[key]){
            return false;
          }
          // 求并集
          return !!~ searchInfo[key].indexOf(item[key].toString());
        })
      })
      return filterData;
    }

    /**
     * 筛选数组
     * item 当前数据
     * val 父级数据
     */
    let selectItem = (item:any,val:any) => {
      item.status = !item.status;
      if (item.status) {
        val.selectName.push(item.name);
        // 全选需要把所有的加进去
        if(item.name == '全部') {
          for (let i = 0; i < val.data.length; i++) {
            val.data[i].status = true;
            if(!val.selectName.includes(val.data[i].name)) {
              val.selectName.push(val.data[i].name);
            }
          }
        
        // 如果都有，全部需要加上
        } else {
          let flag = false;
          for (let i = 0; i < val.data.length; i++) {
            if (val.data[i].name != '全部' && !val.data[i].status) {
              flag = true
            }
          }
          if (!flag) {
            val.data[0].status = true;
          }
        }
      } else {
        // 全选全部去掉
        if(item.name == '全部') {
          for (let i = 0; i < val.data.length; i++) {
            val.data[i].status = false;
            for (let j = 0; j < val.selectName.length; j++) {
              if (val.selectName[j] == val.data[i].name) {
                val.selectName.splice(j,1);
                j--
              }
              
            }
          }
        
        // 只要有一个没有选中，全部需要去掉
        } else {
          for (let i = 0; i < val.selectName.length; i++) {
            if (val.selectName[i] == item.name) {
              val.selectName.splice(i,1);
              break
            }
          }
          val.data[0].status = false;
        }
      }

      for (const key in state.allfilterMap) {
        searchInfo[key] = [...state.allfilterMap[key].selectName]
      }
      
      resultData.value = filterDataFnc()
    }

    let tableToCsv = () => {
      var key:any = [];
      var title = [];
      var deleteTitle:any = [];
      var fileName: string = new Date().getTime() +  '.csv'
      var row = '', csv = '';
      /**
       * 导出格式需要拼接成的字符串 '\""名称,"数量"\r\n"张三","23.25468946921\"';
       * 判断如果有表头，没有表头取第一条数据的对象key
       */
      if(resultData.value.length){
          for(var k in resultData.value[0]){
              if(deleteTitle.indexOf(k) < 0){
                  key.push(k);
                  title.push(k);
              }
          }
      }
      
      title.map(function (item) {
          row += item + ',';
      });
      /**
       * 去掉最后一个字符串
       */
      row = row.slice(0, row.length-1);
      /**
       * 换行
       */
      csv += row + '\r\n';

      if(resultData.value.length){
          resultData.value.map(function (ele:any) {
              row = '';
              /**
               * 有表头
               */
              if(key.length){
                  key.map(function (tit:any) {
                      row += '"' + (ele[tit] != undefined ? ele[tit] : '') + '",';
                  })
              }else{
                  for(var keys in ele){
                      row += '"' + ele[keys] + '",';
                  }
              }
              row = row.slice(0, row.length-1);
              csv += row + '\r\n';
          })
      }
      if(!csv){
          return;
      }
      exportAs(csv, fileName)
      
    }
    /**
     * 导出下载数据
     */
    let exportAs =  (data:any,fileName:string) => {
      var browser = getBrowser();
      if(browser == 'ie11'){
          var utf = "\uFEFF";
          var csvData = new Blob([utf + data], {
              type: 'text/csv'
          });
          let navigator:any = window.navigator;
          navigator.msSaveBlob(csvData, fileName);

      }else if(browser != 'ie' && browser != 'ie11'){
          
          var uft = "\uFEFF";
          var herf = '';
          /**
           * Blob文件对象
           */
          let windowURL:any = window.URL;
          if(window.Blob && windowURL && windowURL.createObjectURL){
            var csvData = new Blob([uft + data],{
                type: 'text/csv'
            })
            /**
             * 类似 blob:d3958f5c-0777-0845-9dcf-2cb28783acaf 字符串
             */
            herf = URL.createObjectURL(csvData);
          }
          var link = document.createElement('a');
          link.id = 'download';
          link.href = herf;
          document.body.appendChild(link);
          var linkDom:any = document.getElementById('download');
          linkDom.setAttribute('download',fileName);
          linkDom.click()
          document.body.removeChild(linkDom);
      }
    }
    /**
     * 判断浏览器类型
     */
    var getBrowser = function () {
      var browser = '';
      if(!!window.ActiveXObject || "ActiveXObject" in window){
        browser = 'ie';
        if(navigator.userAgent.indexOf('Trident') > -1 && navigator.userAgent.indexOf('rv:11.0') > -1){
            browser = 'ie11';
        }
      }else if(navigator.userAgent.indexOf('Edg') > -1){
        browser = 'Edge';
      }else{
        browser = 'other';
      }
      return browser
    }

    let csvToTable = () =>{
      csvFile.value && csvFile.value.click()
    }

    let changeFile = (e:any) =>{
      var file = e.target.files[0];
      let reader= new FileReader();
      reader.readAsText(file,'utf8');
      reader.onload = (event:any) =>{
        let result = event.target?.result;
        if(result) {
          result = JSON.parse(result);
        }
        if(result.data) {
          data = result.data;
          searchInfo = {};
          state.allfilterMap = {}
          initData();
        }
      }
      e.target.value = ''
    }

    return {
      state,
      selectItem,
      resultData,
      tableToCsv,
      csvToTable,
      csvFile,
      changeFile
    }
    
  }
})
</script>

<style>
* {
  margin: 0;
  padding: 0;
}
ul , ul li{
  list-style: none;
}
.title {
  text-align: center;
}
.button-group{
  display: flex;
  justify-content: flex-end;
  padding: 0 0 10px 0;
}
.button-group .json-tip {
  padding: 0 10px;
  margin-right: 10px;
  background: #f4f4f5;
  color: #666;
  line-height: 40px;
}
.button-group button {
  padding: 10px 20px;
  margin-right: 10px;
  border: none;
  outline: 0;
  cursor: pointer;
  color: white;
}
.button-group .import{
  background: rgb(35, 81, 142);
}
.button-group .exprot{
  background: rgb(35, 81, 142);
}
.filter-content{
  font-size: 13px;
}
.filter-content .filter-item {
  display: flex;
  align-items: baseline;
  margin-right: 20px;
}
.filter-content .filter-item .label {
  width: 80px;
  padding-right: 20px;
  text-align: right;
}
.filter-selected{
  display: flex;
  max-width: 200px;
  min-height: 35px;
  padding: 0 10px;
  line-height: 35px;
  cursor: pointer;
}
.filter-content .filter-item .filter-dowm{
  display: flex;
      flex-wrap: wrap;
  width: 100%;
  max-height: 300px;
  overflow: auto;
  background: white;
}
.filter-content .filter-item .filter-dowm li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 5px 10px;
  margin: 10px 10px 0 0;
  border: 1px solid #e0e0e0;
  border-radius: 30px;
  cursor: pointer;
}
.filter-content .filter-item .filter-dowm li.selected{
  border: 1px solid green;
  color: green;
}
.table{
  width: 100%;
  border: 1px solid #f0f0f0;
  text-align: center;
  font-size: 13px;
}
.table thead tr th {
  border-right: 1px solid #f0f0f0;
  border-bottom: 1px solid #f0f0f0;
  padding: 10px 0;
}
.table tbody tr td {
  border-right: 1px solid #f0f0f0;
  border-bottom: 1px solid #f0f0f0;
  padding: 6px 0;
}
</style>