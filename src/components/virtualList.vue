<template>
    <div class="outContainer">
        <!-- scrollerContainer为支持滚动条的容器，定义整个虚拟列表的高度 --> 
        <div class="scrollerContainer" ref="scrollerContainerRef" @scroll="onScroll">
            <!-- 绝对定位实现撑开滚动容器 -->
            <div class="pillarDom" :style="{ height: `${pillarDomHeight}px` }"></div>
            <div class="contentList" :style="styleTranslate">
                <div class="item" v-for="oneData in renderData" :key="oneData.id">{{ oneData.content }}</div>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    name:'virtualList',
    data(){
        return{
            //单位数据高度
            itemHeight:120,
            //所有数据
            allData:[],
            //渲染区域与视口区域没有重合的问题，当一个元素完全以到了可视区域之外，需要重新计算startoffset
            /*
            transform原本是（0，0）渲染发生变化，要把视口区域也变化，所以需要调整transform进行修正
            */
            startOffset :0,
            //开始位置
            start:0,
            scrollerContainerRefHeight:0,
        }
    },
    methods:{
        //模拟网络延迟
        loadData(){
            return new Promise(resolve=>{
                const tmpList = []
                for(let i = 0;i<10000;i++){
                    tmpList.push({ id: i, content: `数据${i}` })
                }
                setTimeout(() => {
                    resolve(tmpList)
                }, 1000)
            })
        },
        async init(){
            this.allData = await this.loadData()
        },
        onScroll(e){
            // console.log('onScroll',e)
            // 获取触发滚动事件的元素
            const scrollDom = e.target
            if (!scrollDom) return

            // 获取滚动的距离
            const { scrollTop } = scrollDom
            // 根据滚动的距离，计算此时视口顶部需要显示的第一个元素
            this.start = Math.floor(scrollTop / this.itemHeight)
            // 修正偏移量
            
            let realStart = Math.max(0, this.start - this.pageItemCount)
            this.startOffset = realStart * this.itemHeight
            // console.log('onScroll',this.start,this.startOffset)
        }
    },
    computed:{
        //滚动条高度(总的数据高度)
        pillarDomHeight(){
            return this.itemHeight * this.allData.length
        },
        //设置y偏移
        styleTranslate (){
            return `transform:translate(0,${this.startOffset}px)`
        },
        //每一页的高度，视口高度除以每一项的高度(向上取整  )
        pageItemCount(){
            // console.log('pageItemCount',Math.ceil(this.scrollerCotainerRefHeight / this.itemHeight)+1)
            return Math.ceil(this.scrollerContainerRefHeight / this.itemHeight)+1
        },
        //结束位置
        end(){
            return this.start + this.pageItemCount
        },
        //从总数据截取部分数据进行显示
        renderData(){
            // 前面多渲染，避免初始小于0,缓冲一屏
            const realStart = Math.max(0,this.start - this.pageItemCount)
            const realEnd = Math.min(this.end+this.pageItemCount,this.allData.length)
            return this.allData.slice(realStart,realEnd)
        },

    },
    created(){
        this.$nextTick(()=>{
            //mounted里面获取不到数据，数据是异步操作加载后生成的
            //mounted之后获得视口高度
            this.scrollerContainerRefHeight = this.$refs.scrollerContainerRef ? this.$refs.scrollerContainerRef.offsetHeight : 0
            // console.log('created',this.$refs.scrollerContainerRef.offsetHeight)
        })
    },
    mounted(){
        this.init()
    }
}
</script>

<style lang="scss" scoped>
.outContainer {
  margin: auto;
  height: 450px;
  width: 50%;
  border: 10px red solid;
}
// 绝对定位实现虚拟列表
.scrollerContainer {
  height: 100%;
  width: 100%;
  overflow: auto;
  position: relative;
  -webkit-overflow-scrolling: touch;
}
.pillarDom {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: -1;
}
.contentList {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
}
.item {
  display: flex;
  justify-content: center;
  height: calc(v-bind(itemHeight) * 1px);
  line-height: calc(v-bind(itemHeight) * 1px);
  border-bottom: 8px solid skyblue;
  width: 100%;
  // 这里同样很重要，盒模型必须为border-box，item元素的高度才不会因为border值而超出设置的高度
  box-sizing: border-box;
  background-color: pink;
  &:last-child {
    border-bottom: none;
  }
}
</style>