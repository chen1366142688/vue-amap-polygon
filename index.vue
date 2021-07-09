<template>
  <div id="showSchool">
    <div class="amap-wrapper">
      <el-amap class="amap-box" vid="amap" ref="map" :center="center" :zoom="zoom" :events="events">
        <!-- 行政区划分 -->
        <el-amap-polygon
          strokeColor="#0091ea"
          strokeOpacity="0.8"
          strokeStyle="dashed"
          fillColor="#c9ebf7"
          fillOpacity="0.5"
          v-for="(polygon, index) in polygons"
          :key="index"
          strokeWeight="1"
          :path="polygon.Ce.path"
          :events="events"
        ></el-amap-polygon>
        <!-- 地图点标记 -->
        <el-amap-marker
          v-for="(marker,index) in markers"
          :key="index+1"
          :events="marker.events"
          :icon="marker.icon"
          :position="marker.position"
        ></el-amap-marker>
        <!-- 信息窗口 -->
        <el-amap-info-window
          v-if="window"
          :position="window.position"
          :visible="window.visible"
          :content="window.content"
          :offset="window.offset"
          :close-when-click-map="true"
          :is-custom="true"
        >
          <div id="info-window">
            <div class="info-top">
              <!-- <div class="left">
                <img src="../../../images/school_icon.png" alt="" />
              </div> -->
              <div class="right">
                  <!-- 0未知 1 幼儿园 2小学 3初中 4高中 5大学 -->
                <div class="school-name">{{ window.info.schoolName }}<span class="grade">{{ window.info.schoolType == '1' ? '幼儿园': window.info.schoolType == '2' ? '小学': window.info.schoolType == '3' ? '初中': window.info.schoolType == '4' ? '高中': window.info.schoolType == '5' ? '大学':'未知类型'}}</span></div>
                <div class="address" style="margin-top:10px;">地址：{{ window.info.address }}</div>
              </div>
            </div>
            <div class="info-middle">
              <div class="line1">
                <p>
                  <img src="../../../images/icon_person.png" alt="">
                  <span>联系人-{{ window.info.teacherName }}：{{ window.info.phoneNumber }}</span>
                </p>
                <p>
                  <img src="../../../images/icon_student.png" alt="">
                  <span>学生人数： {{ window.info.studentCount }}人</span>
                </p>
              </div>
              <div class="line2">
                <div>
                  <p class="test-title">
                      <img src="../../../images/icon-test.png" alt="">
                      <span>体质测试</span> 
                  </p>
                  <p class="testItem">平均分：<span>{{ window.info.testScore.average }}</span> 优良率：<span>{{ window.info.testScore.excellentRate }}%</span> 合格率：<span>{{ window.info.testScore.passRate }}%</span></p>
                </div>
                <div>
                  <p class="test-title test-titles">
                      <img src="../../../images/icon_teacher.png" alt="">
                      <span>学科评价</span> 
                  </p>
                  <p  class="testItem">平均分：<span>{{ window.info.subjectEvaluation.average }}</span> 优良率：<span>{{ window.info.subjectEvaluation.excellentRate }}%</span> 合格率：<span>{{ window.info.subjectEvaluation.passRate }}%</span></p>
                </div>
              </div>
              <div style="margin-top:10px;color:#3385FF;cursor: pointer;" @click="checkDetail">查看详情 ></div>
            </div>
            <img src="../../../images/icon_triangle.png" alt="" class="bottom_icon">
          </div>
        </el-amap-info-window>
        <div class="searchData">
            <Input prefix="ios-search" search enter-button="搜索" placeholder="请输入学校名称搜索" class="searchKung" @on-search="searchSchool"/>
            <Icon type="ios-search" size="24" class="searchKungImg"/>
            <ul class="resultBox" v-if="isShowList">
                <li class="resultItem" v-for="(item,index) in searchData" :key="item.schoolName" @click="toActionSchool(item)">
                    <Icon type="ios-search" size="24" class="ios-search-result"/>
                    <span v-if="item">{{item.schoolName}}</span>  
                </li>
            </ul>
        </div>
      </el-amap>
    </div>
  </div>
</template>

<script>
import mapIcon from '../../../images/schoolLocation.png'
// import schoolIcon from '../../../images/school_icon.png'
import schoolData from '../../../libs/schoolData.json';
export default {
  name: "showSchool",
  data() {
    return {
      zoom:13,
      center: [104.182981,30.653647],
      polygons: [],
      district: null,
      placeSearch: null,
      markers: {},
      data: [],
      searchData:[],
      windows: [],
      window: "",
      isShowList:false,
      events: {
        init: (o) => {},
        'moveend': () => {},
        'zoomchange': () => {},
        'click': (e) => {
            this.window.visible = false;
            this.isShowList = false;
        }
      },
    };
  },
  props: {
    value: {
      type: Boolean,
      default: false
    }
  },
  created() {},
  mounted() {
    this.data = schoolData.data;
    setTimeout(() => {
      this.drawBounds();
      this.point();
    }, 200);
  },
  beforeDestroy() {
    this.$refs.map.$$getInstance().destroy();
  },
  methods: {
    /*地图上添加行政区边界*/
    drawBounds() {
      let that = this;
      //加载行政区划插件
      if (!that.district) {
        //实例化DistrictSearch
        var opts = {
          subdistrict: 0, //获取边界不需要返回下级行政区
          extensions: "all", //返回行政区边界坐标组等具体信息
          level: "district", //查询行政级别为 市
        };
        that.district = new AMap.DistrictSearch(opts);
      }
      //行政区查询
      that.district.search("龙泉驿区", function (status, result) {
        that.polygons = [];
        var bounds = result.districtList[0].boundaries;
        if (bounds) {
          for (var i = 0, l = bounds.length; i < l; i++) {
            //生成行政区划polygon
            var polygon = new AMap.Polygon({
              strokeWeight: 1,
              path: bounds[i],
              fillOpacity: 0.4,
              fillColor: "#80d8ff",
              strokeColor: "#0091ea",
            });
            that.polygons.push(polygon);
          }
        }
        AMap.Polygon.bind(that.polygons);
        // that.$refs.map.$amap.setFitView(that.polygons); //视口自适应
      });
    },
    /*地图上添加标记点和详细信息*/
    point() {
      const markers = [];
      const windows = [];
      const that = this;
      this.data.forEach((item, index) => {
        item.url = mapIcon
        markers.push({
          position: item.position.split(","),
          icon:item.url, //不设置默认蓝色水滴
          events: {
            mouseover() {
              // 方法：鼠标移动到点标记上，显示相应窗体
              that.windows.forEach((window) => {
                window.visible = false; // 关闭窗体
              });
              that.window = that.windows[index];
              that.$nextTick(() => {
                that.window.visible = true;
              });
            },
          },
        });
        windows.push({
          position: item.position.split(","),
          isCustom: true,
          offset: [5, -39], // 窗体偏移
          showShadow: false,
          visible: false, // 初始是否显示
          info:item
        });
      });
      //  加点
      this.markers = markers;
      // 加弹窗
      this.windows = windows;
      that.window = that.windows[12];
      that.window.visible = true;
    },
    /*跳转到学校后台*/
    checkDetail() {
    },
    /*查询学校*/
    searchSchool(value){
        this.searchData = this.data.filter((val,index,arr) => {
            return val.schoolName.includes(value) == true;
        });
        this.isShowList = true;
    },
    /*滑动地图到当前学校*/
    toActionSchool(windowData){
        let vm = this;
        let positionData = windowData.position.split(',');
        let index = 0;
        this.$refs.map.$$getInstance().panTo([positionData[0], positionData[1]]);
        for (let [key,val] of vm.data.entries()) {
            if(val.schoolName == windowData.schoolName){
                index = key;
                break;
            }
        }
        this.window.visible = false;
        this.window = this.windows[index];
        this.window.visible = true;
    },
  },
};
</script>
<style lang="less" scoped>
.amap-wrapper {
  width: calc(100%-250px);
  height: 100vh;
}
.amap-box{
    width:100%;
    height: 100%;
    position: relative;
    .searchData{
        position: absolute;
        top:27px;
        left:34px;
        z-index: 200;
        .searchKung{
            width:282px;
            height:32px;
        }
        .searchKungImg{
            position: absolute;
            top:5px;
            left:5px;
            z-index: 2001;
        }
        .resultBox{
            width: 282px;
            overflow-y: scroll;
            position: absolute;
            top:34px;
            left:0px;
            background: #FFFFFF;
            list-style: none;
            max-height: 340px;
            .resultItem{
                width:100%;
                height:34px;
                display: flex;
                align-items: center;
                border-bottom:1px solid #F0F0F0;
                cursor: pointer;
                .ios-search-result{
                    margin-right:10px;
                    margin-left:10px;
                }
            }
        }
    }
}
::-webkit-scrollbar {
  width: 0;
  height: 0;
}
#info-window {
  position: relative;
  padding: 0;
  display: flex;
  flex-direction: column;
  width: 408px;
  background:rgba(255,255,255,1);
  box-shadow:0px 3px 12px 2px rgba(0, 0, 0, 0.15);
  border-radius:8px;
  background-clip: border-box;
  box-shadow:0px 4px 15px 3px rgba(0, 0, 0, 0.15);
  flex: 1 1 auto;
  .info-top {
    background:#fff;
    display:flex;
    box-sizing: border-box;
    padding:15px;
    border-bottom: 1px solid #EEEEEE;
    .right{
      .school-name{
        font-size:16px;
        font-weight:bold;
        .grade{
          display:inline-block;
          width:67px;
          height:24PX;
          background:rgba(174,220,252,0.6);
          border-radius:5px;
          text-align: center;
          line-height:24PX;
          font-size:14px;
          margin-left: 10px;
          color:rgba(50, 118, 222, 1);
        }
      }
    }
  }
  .info-middle {
    font-size: 14px;
    padding: 15px;
    line-height: 20px;
    background-color: white;
    .line1{
      p{
        display:flex;
        align-items:center;
        margin-bottom:10px;
        span{
          margin-left:6px;
        }
      }
    }
    .line2{
      .test-title{
          display:flex;
        align-items:center;
        margin-bottom:2px;
        span{
          margin-left:6px;
        }
      }
      div .testItem{
          padding-left:25px;
        span{margin-right:6px;}
        span:nth-of-type(1){
          color:rgba(253, 108, 45, 1);
        }
        span:nth-of-type(2){
          color:#2ABB2D;
        }
        span:nth-of-type(3){
          color:#3276DE;
        }
      }
      .test-titles{
          margin-top:8px;
      }
    }
  }
  .bottom_icon{
    position: absolute;
    bottom: -12px;
    left: 50%;
    transform: translateX(-50%);
  }
}
</style>
