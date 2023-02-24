<template>
    <div class="outContainer">
        <!-- scrollerContainer为支持滚动条的容器，定义整个虚拟列表的高度 --> 
        <div class="scrollerContainer" ref="scrollerContainerRef" @scroll="onScroll">
            <!-- 绝对定位实现撑开滚动容器 -->
            <div class="pillarDom" :style="{ height: `${pillarDomHeight}px` }"></div>
            <div class="contentList" :style="styleTranslate" ref="contentListRef">
                <div class="item" v-for="oneData in renderData" :data-index="oneData.arrPos" :key="oneData.id">
                    <h4>{{ oneData.arrPos }} : {{ oneData.title }}</h4>
                    <p>{{ oneData.content }}</p>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import Mock from 'mockjs'
import { markRaw } from 'vue'
export default {
    name:'virtualList2',
    data(){
        return{
            // 默认渲染高度
            maybeHeight:250,
            // 所有数据
            allData:[],
            // 滚动容器高度
            scrollerContainerRefHeight:0,
            // 内容列表容器dom
            contentListRef:0,
            // 当前视口第一个元素的起始位置
            start:0,
            // transform设置偏移 让视口范围和渲染范围统一
            startOffset:0,
            pillarDomHeight:0,
            //加入缓冲数据
            cacheCount:5,
        }
    },
    methods:{
        //模拟网络延迟
        loadData(){
            return new Promise(resolve => {
                const data = Mock.mock({
                'list|1000': [
                    {
                    // 数据项的数据
                    // 属性 id 是一个自增数，起始值为 1，每次增 1
                    'id|+1': 1,
                    title: '@ctitle(10, 20)',
                    content: '@cparagraph(1, 7)',
                    },
                ],
                })
                resolve(data.list)
            })
        },
        onScroll(e){
            const scrollerContainerDom = e.target
            if (!scrollerContainerDom) return

            const { scrollTop } = scrollerContainerDom
            // let idx = 0
            // let dataItem = this.positionDataArr[idx]
            // // 遍历元素直到元素底部位置大于滚动高度
            // while (dataItem.endPos <= scrollTop) {
            //     idx++
            //     dataItem = this.positionDataArr[idx]
            // }
            // this.start = idx
            this.start = this.findStartByBinarySearch(this.positionDataArr,scrollTop)
            const realStart = Math.max(0,this.start - this.cacheCount)
            // 渲染与视口不一致进行修正
            this.startOffset = this.positionDataArr[realStart].startPos
            // console.log(start.value)
        },
        updatedHeightAndPos(){
            const contentListDom = this.contentListRef
            if (!contentListDom) return

            const childrenElementArr = contentListDom.children
            for (let i = 0; i < childrenElementArr.length; i++) {
                const childEle = childrenElementArr[i]
                // 获取当前数据dom节点的数据再allData数组中的索引位置
                const dataIndexStr = childEle.dataset['index']
                if (!dataIndexStr) continue
                
                const dataIndex = parseInt(dataIndexStr)
                // 从allData数据中获取到该数据
                const dataItem = this.positionDataArr[dataIndex]
                if (!dataItem) continue

                // 获取元素的实际高度
                const { height } = childEle.getBoundingClientRect()
                const oldHeight = dataItem.height
                /*
                计算当前数据dom元素的旧高度和当前高度的差值

                如：
                oldHeight为100px，height为50px, 那么dffVal为 50px，那么 oldHeight - dffVal 为 50px
                oldHeight为50px，height为100px, 那么dffVal为 -50px，那么 oldHeight - dffVal 为 100px
                */
                const dffVal = oldHeight - height
                if (dffVal != 0) {
                    // 当前dom元素的实际高度与allData中记录的高度不一致，则更新高度以及元素位置信息
                    dataItem.height = oldHeight - dffVal
                    dataItem.endPos = dataItem.endPos - dffVal
                    //更新后续dom元素
                    for (let j = dataIndex + 1; j < this.positionDataArr.length; j++) {
                        const jPosDataItem = this.positionDataArr[j]
                        // j位置的上一个位置的元素
                        const jPrevPosDataItem = this.positionDataArr[j - 1]

                        jPosDataItem.startPos = jPrevPosDataItem.endPos
                        jPosDataItem.endPos = jPosDataItem.startPos + jPosDataItem.height
                    }
                }
            }
            this.pillarDomHeight = this.positionDataArr.length > 0 ? this.positionDataArr[this.positionDataArr.length - 1].endPos : 0
        },
        findStartByBinarySearch(_positionDataArr, scrollTop) {
            let startIdx = 0
            let endIdx = _positionDataArr.length - 1
            let resultIdx = undefined
            while (startIdx <= endIdx) {
                // Math.trunc 去除小数部分，只取整数部分. 取startIdx 到 endIdx的中间索引号
                const middleIdx = Math.floor((startIdx + endIdx) / 2)
                // 获取中间索引号对应元素的位置信息
                const middleEle = _positionDataArr[middleIdx]
                // 获取中间索引号对应元素的底部位置
                const middleEleEndPos = middleEle.endPos
                if (middleEleEndPos === scrollTop) {
                // 当前滚动高度等于中间索引号对应元素的底部位置，则start为中间索引号的下一个位置
                return middleIdx + 1
                } else if (middleEleEndPos < scrollTop) {
                // 当前滚动高度大于中间索引号对应元素的底部位置，则调整查找区间为右区间
                startIdx = middleIdx + 1
                } else if (middleEleEndPos > scrollTop) {
                // 当前滚动高度大于中间索引号对应元素的底部位置，则调整查找区间为左区间
                if (resultIdx === undefined || resultIdx > middleIdx) {
                    // 存储元素 middleEleEndPos>scrollTop 元素的最小数组索引号
                    resultIdx = middleIdx
                }
                // 调整查找区间为左区间
                endIdx = middleIdx - 1
                }
            }
            return resultIdx
        },
        async init(){
            const tmpAllData = await this.loadData()
            this.allData = tmpAllData.map((item, idx) => markRaw({...item,arrPos:idx}))
            this.positionDataArr = this.allData.map((_,idx)=>({
                arrPos:idx,
                startPos:this.maybeHeight*idx,
                endPos:this.maybeHeight*idx,
                height:this.maybeHeight, 
            }))
        },
    },
    computed:{
        //设置y偏移
        styleTranslate (){
            return `transform:translate(0,${this.startOffset}px)`
        },
        //结束位置
        end(){
            if (!this.allData || this.allData.length <= 0) return 0

            // 将start作为遍历allData的开始位置
            let endPos = this.start
            // contentDomTotalHeight存放从start位置开始的dom节点总高度
            let contentDomTotalHeight = this.positionDataArr[endPos].height
            // 获取视口高度
            const viewPortHeight = this.scrollerContainerRefHeight
            // 从start位置开始遍历allData的同时，统计数据dom节点的累计高度，直至累计高度超过了视口高度
            while (contentDomTotalHeight < viewPortHeight) {
                endPos++
                contentDomTotalHeight += this.positionDataArr[endPos].height
            }
            // 因为数组的slice方法是包头不包尾的所以还需要再endPos上+1，才会是预期的元素数量
            endPos += 1
            // 因为存在在某个元素位置开区间滚动的情况，此时该元素不会完全移出视口，但又使得视口多出了位置，因此要再+1，渲染下一个元素来占满视口区域
            return endPos + 1
        },
        //从总数据截取部分数据进行显示
        renderData(){
            const realStart = Math.max(0,this.start - this.cacheCount)
            // 避免最后一个元素的数组下标超出实际的数组长度
            const realEnd = Math.min(this.end + this.cacheCount, this.allData.length)
            return this.allData.slice(realStart, realEnd)
        },

    },
    created(){
        this.$nextTick(()=>{
            //mounted里面获取不到数据，数据是异步操作加载后生成的
            //mounted之后获得视口高度
            this.scrollerContainerRefHeight = this.$refs.scrollerContainerRef ? this.$refs.scrollerContainerRef.offsetHeight : 0
            this.contentListRef = this.$refs.contentListRef
        })
    },
    mounted(){
        this.init()

    },
    updated(){
        this.$nextTick(()=>{
            this.updatedHeightAndPos()
        })
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
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: calc(v-bind(itemHeight) * 1px);
  line-height: calc(v-bind(itemHeight) * 1px);
  padding: 5px 10px;
  border-bottom: 8px solid skyblue;
  width: 100%;
  // 这里同样很重要，盒模型必须为border-box，item元素的高度才不会因为border值而超出设置的高度
  box-sizing: border-box;
  background-color: pink;
//   &:last-child {
//     border-bottom: none;
//   }
}
</style>