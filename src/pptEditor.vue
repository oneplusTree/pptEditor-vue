<template>
    <div class="pages" ref="pages" tabindex="-1">
        <!-- 工具栏 -->
        <div class="contorl" v-show="!isPreview" :style="`top:${(height+32)*activeIndex+88}px;`">
            <span :class="(item=='pen'&&isDraw)?'active jk-'+item:'jk-'+item" v-for="(item,index) in controls" @click="setControl(item)" :key="index">

            </span>
        </div>
        <!--画板-->
        <div draggable="true" tabindex="0" :class="item.active?'draw-box active':'draw-box'" v-for="(item,index) in pages" @dblclick="stop()" :isPreview="isPreview" @click="activeCanvas(index)" @contextmenu="contextmenu($event)" :key="index">
            <div class="view" v-show="!item.active&&!isPreview" :style="`width:${width}px;height:${height}px;`">
                <span>单击编辑</span>
            </div>
            <canvas :id="item.id" :width="width" :height="height" tabindex="0"></canvas>
        </div>
        <!--属性-->
        <div class="prop-box" v-show="!isPreview" :style="`top:${(height+32)*activeIndex+11}px;`">
            <span :class="`jk-${item}`" v-for="(item,index) in props" @click.stop="setProp(item)" @mousemove="propType=item" :key="index">
                <div class="colorbox" v-if="'fill stroke back'.indexOf(item)>-1">
                    <colorPiker @change="colorChange" :showView="false" :isShow="true" />
                </div>
                <div class="fonts" v-if="item == 'text'">
                    <span v-for="(f,index) in fonts" @click.stop="setProp(item,f)" :key="index">{{f}}</span>
                </div>
                <div class="range" v-if="'font opacity border lineheight charspacing'.indexOf(item)>-1">
                    <span v-if="item=='font'">
                        <input type="range" min="7" max="100" step="1" @input="setProp(item,$event)" v-model="fontSize">{{fontSize}}
                    </span>
                    <span v-if="item=='opacity'">
                        <input type="range" min="0" max="1" step="0.1" @input="setProp(item,$event)" v-model="opacity">{{opacity}}
                    </span>
                    <span v-if="item=='border'">
                        <input type="range" v-if="item=='border'" value="1" min="0" max="30" step="1" @input="setProp(item,$event)" v-model="strokeWidth">{{strokeWidth}}
                    </span>
                    <span v-if="item=='lineheight'">
                        <input type="range" min="0.1" max="10" step="0.1" @input="setProp(item,$event)" v-model="lineHeight">{{lineHeight}}
                    </span>
                    <span v-if="item=='charspacing'">
                        <input type="range" min="-200" max="800" step="10" @input="setProp(item,$event)" v-model="charSpacing">{{charSpacing}}
                    </span>
                </div>
            </span>
        </div>
        <!--菜单-->
        <menu class="contextMenu" ref="contextMenu" v-show="contextMenuShow">
            <li v-for="(menu,index) in contextMenu" @click="excue(menu.shortcut)" :key="index">
                <span>{{menu.text}}</span>
                <span class="shortcut">{{menu.value}}</span>
            </li>
        </menu>
        <!--弹框组件-->
        <compUI :showShiTiBox="showShiTiBox" :showImgBox="showImgBox" :showSvgBox="showSvgBox" :showVideoBox="showVideoBox" :showTableBox="showTableBox" ref="comp" @close="closeBox" @addComponent="addComponent"></compUI>
    </div>
</template>
<script>
import qdraw from 'qdraw'
import draw from './draw'
import compUI from './compUI'
import history from './history'

// import colorPiker from 'color-picker-vue'
import colorPiker from 'colorpicker-vue'

export default {
    components: { compUI, colorPiker, },
    name: 'pptEditor',
    props: {
        'width': { type: Number, required: true, default: 788 },
        'height': { type: Number, required: true, default: 443 },
        'isPreview': { type: Boolean, default: false, }
    },
    watch: {
        isPreview(v) { this.cancelSelect(v); this.canvas.isPreview = v; },
    },
    data() {
        return {
            fontSize: 12, opacity: 0, strokeWidth: 0, lineHeight: 1, charSpacing: 0,
            showShiTiBox: false, showImgBox: false, showSvgBox: false, showVideoBox: false, showTableBox: false,
            jsonData: [], props: [], activeIndex: 0, propType: '',
            propText: [
                'undo', 'redo', 'bold', 'italic', 'underline', 'linethrough', 'text',
                'font', 'lineheight', 'charspacing', 'alignleft', 'alignright', 'aligncenter', 'alignjustify',
                'fill', 'stroke', 'back', 'opacity', 'border',
                'flipx', 'flipy', 'remove', 'lock'
            ],
            static: ['undo', 'redo', 'fill', 'stroke', 'back', 'opacity', 'border', 'flipx', 'flipy', 'group', 'ungroup', 'remove', 'lock'],
            group: ['undo', 'redo', 'groupleft', 'groupright', 'grouptop', 'groupbottom', 'groupcenterv', 'groupcenterh',  /* 'groupvertical', 'grouphorizontal', */ 'group', 'ungroup', 'remove', 'lock'],
            isDraw: false,
            controls: [/* 'info', */ 'plus', 'minus', 'pen', 'text', 'image', 'shape', 'line','link', 'table', 'media', 'book', 'view'],
            canvas: '', draws: [], pages: [{ active: true, id: `canvas-${new Date() * 1}` }],
            contextMenuShow: false,
            contextMenu: [
                { text: '剪切', value: '⌘X', shortcut: 'cut' },
                { text: '复制', value: '⌘C', shortcut: 'copy' },
                { text: '粘贴', value: '⌘V', shortcut: 'paste' },
                { text: '删除', value: '⌫', shortcut: 'delete' },
                { text: '锁定/解锁', value: '⌘L', shortcut: 'lock' },
                // { text: '组合', value: '⌘G', shortcut: 'group' },
                // { text: '拆分', value: '⌘↑G', shortcut: 'ungroup' },
                { text: '上移', value: '⌘↑', shortcut: 'moveup' },
                { text: '下移', value: '⌘↓', shortcut: 'movedown' },
                { text: '顶层', value: '⌘⇧↑', shortcut: 'movefront' },
                { text: '底层', value: '⌘⇧↓', shortcut: 'moveback' },
            ],
            fonts: [
                '幼圆', '新宋体', '微软雅黑', '宋体', '隶书', '楷体', '黑体', '仿宋',// 自带标准字体
                '华文中宋', '华文新魂', '华文细黑', '华文宋体', '华文隶书', '华文楷体', '华文琥珀', '华文行楷', '华文仿宋', '华文云彩', // win7 华文系列
                '方正准圆简体', '方正姚体', '方正细圆简体', '方正舒体', //方正系列
                'Arial', 'Akzidenz Grotesk', 'Bodoni', 'Consolas', 'Courier', 'Comic sans ms', 'Delicious', 'Engagement',
                'Frutiger', 'Futura', 'Gill Sans', 'Georgia', 'Garamond', 'Hoefler Text', 'Helvetica Neue', 'Helvetica', 'Impact', 'Monaco',
                'Myriad pro', 'Officina', 'Optima', 'Plaster', 'PingFang SC', 'Roman', 'Times', 'Tahoma', 'Univers', 'Vani', 'Verdanna', // 常用英文字体
            ],
            objStyle: {}, copyData: []
        }
    },
    beforeMount() {
        //加载字体
        var container = document.getElementsByTagName("head")[0];
        var addStyle = document.createElement("link");
        addStyle.rel = "stylesheet";
        addStyle.type = "text/css";
        addStyle.media = "screen";
        addStyle.href = '//at.alicdn.com/t/font_335448_i2m88q76oa40wwmi.css';
        container.appendChild(addStyle);
        var meta = document.createElement('meta')
        meta.name = 'renderer'
        meta.content = 'webkit'
        container.appendChild(meta);
    },
    created() {
        // box.oncontextmenu = (e) => { e.preventDefault(); }
        document.onclick = () => { this.contextMenuShow = false }
        document.ondragover = (e) => { e.preventDefault() }
        document.ondrop = (e) => { draw.ondrop(e) }
        draw.pages = this.pages
        draw.draws = this.draws
        draw.width = this.width
        draw.height = this.height
        draw.screenChange()
    },
    methods: {
        // resize(w, h, p) { draw.resize(w, h, p) },
        stop() {
            draw.togglePlay()
        },
        contextmenu(e) {
            e.preventDefault();
            if (this.isPreview) return
            let menu = this.$refs.contextMenu
            let y = document.body.clientHeight - e.clientY > 308 ? e.clientY : e.clientY - 308
            let x = document.body.clientWidth - e.clientX > 212 ? e.clientX : e.clientX - 202
            menu.style.left = x + 'px'
            menu.style.top = y + 'px'
            this.contextMenuShow = true
            return
        },
        colorChange(v) {
            switch (this.propType) {
                case 'fill': draw.setFill(v); break;
                case 'stroke': draw.setStroke(v); break;
                case 'back': draw.setBgColor(v); break;
            }
        },
        cancelSelect(v) {
            draw.cancelSelect()
        },
        closeBox() {
            this.showShiTiBox = false
            this.showImgBox = false
            this.showSvgBox = false
            this.showVideoBox = false
            this.showTableBox = false
        },
        getImage(index, formart = 'jpeg') {
            if (index >= 0 && index < this.draws.length) {
                return this.draws[index].toDataURL(formart)
            }
            if (this.draws.length > 0) {
                let data = []
                this.draws.map((x) => {
                    let d = x.toDataURL(formart)
                    data.push(d)
                })
                return data
            }
        },
        getJsonData() {
            let jsonData = []
            if (this.draws.length > 0) {
                this.draws.map((x) => {
                    let d = JSON.stringify(x)
                    let rd = d.replace(/"stroke":null,|"transformMatrix":null,|"visible":true,|"opacity":1,|"flipY":false,|"scaleX":1,|"scaleY":1,|"flipX":false,|"angle":0,|"skewX":0,|"skewY":0,|"strokeDashArray":null,|"shadow":null,|"clipTo":null,|"backgroundColor":"",|"textDecoration":"",|"textBackgroundColor":"",|"fontStyle":"",|"fontWeight":"",|"fill":"",|"fill":"rgb(0,0,0)",|"fillRule":"nonzero",|"strokeMiterLimit":10,|"strokeLineJoin":"miter",|"strokeLineCap":"butt",|"fontWeight":"normal",|"originX":"left",|"originY":"top",|"textAlign":"left",|"charSpacing":0,/gm, '')
                    let data = JSON.parse(rd)
                    jsonData.push(data)
                })
                return jsonData
            }
            return jsonData
        },
        updateJosn() {
            // this.$emit('getDataJson', '此被废弃 ，因为非常耗内存 请使用 getJsonData')
            setTimeout(() => {
                // console.log()
                history.update(JSON.stringify(this.canvas))
            }, 500);
            return;
            //此被废弃 ，因为非常耗内存 请使用 getJsonData 
            if (this.draws.length > 0) {
                this.jsonData = []
                this.draws.map((x) => {
                    let d = JSON.stringify(x)
                    d = JSON.parse(d)
                    this.jsonData.push(d)
                })
                this.$emit('getDataJson', this.jsonData)
            }
        },
        setControl(item) {
            switch (item) {
                case 'info': break;
                case 'plus': this.addPage(); break;
                case 'minus': this.removePage(); break;
                case 'pen': this.isDraw = !this.isDraw; draw.setFreeDrawingMode(this.isDraw); break;
                case 'text': this.addComponent('Textbox'); break;
                case 'image': this.showImgBox = true; break;
                case 'shape': this.showSvgBox = true; break;
                case 'line': draw.drawMode('line'); break;
                case 'link': draw.drawMode('link'); break;
                case 'table': this.showTableBox = true; break;
                case 'media': this.showVideoBox = true; break;
                case 'book': this.showShiTiBox = true; break;
                case 'view': draw.elementFullScreen(this.$refs.pages); break;
                // case 'fullscreen': this.toggleFullScreen(this.$refs.pages); break;
            }
            item != 'pen' && (this.isDraw = false, draw.setFreeDrawingMode(false))
           'line,link,rect,circle,triangle'.indexOf(item)<0 && draw.drawMode(false);
            this.cancelSelect(false)
        },
        activeCanvas(index, event) {
            if (!this.pages[index].active) {
                this.pages.map(x => x.active = false)
                this.pages[index].active = true
                this.canvas = this.draws[index]
                draw.init(this.canvas, this.callback)
                draw.setFreeDrawingMode(this.isDraw)
                this.activeIndex = index
                history.init(this.canvas)
            }
        },
        toggleProp(type) {
            let align = 'Left'
            switch (type) {
                case 'bold': draw.toggleBold(); break;
                case 'italic': draw.toggleItalic(); break;
                case 'underline': draw.toggleUnderline(); break;
                case 'linethrough': draw.toggleLinethrough(); break;
                case 'overline': draw.toggleOverline(); break;
                // case 'lock': draw.toggleLock(); break;
                case 'alignleft':
                case 'alignright':
                case 'aligncenter':
                case 'alignjustify': draw.setActiveProp('textAlign', type); break;
            }
            this.updateJosn()
        },
        setAlign(name, o) {
            // let ls = g.map((x) => { return { l: x.left, r: x.left + x.width, t: x.top, b: x.top + x.height } })
            // console.log(ls)
            let g = o.getObjects()
            var ox = o.width / 2, oy = o.height / 2, r = 6.5;

            switch (name) {
                case 'groupleft': g.map(x => x.set('left', !x.flipY ? -ox : -ox + x.width * x.zoomX + r).setCoords()); break;
                case 'groupright': g.map(x => x.set('left', ox - x.width * x.zoomX - r).setCoords()); break;
                case 'grouptop': g.map(x => x.set('top', -oy).setCoords()); break;
                case 'groupbottom': g.map(x => x.set('top', oy - x.height * x.zoomY - r).setCoords()); break;
                case 'groupcenterv': g.map((x) => x.set('left', -x.width * x.zoomX / 2).setCoords()); break;
                case 'groupcenterh': g.map((x) => x.set('top', -x.height * x.zoomY / 2).setCoords()); break;
                case 'groupvertical':// 垂直分布算法
                    /*count = g.length
                    bs = g.map(x => x.top + x.height)
                    ts = g.map(x => x.top)
                    maxb = Math.max.apply(null, bs)
                    mint = Math.min.apply(null, ts)
                    let y = (maxb - mint) / count
                    newboxs = g.map((x, i) => mint + y * i)
                    g.map((x, i) => i != 0 && i != count - 1 && x.set('top', mint + o).setCoords())*/
                    let arr = g.map(x => x.height).join('+')
                    let t = (o.height - eval(arr)) / g.length
                    g.map(x => x.set('top', t).setCoords())
                    break;
                case 'grouphorizontal': // 水平分布算法
                    // count = g.length

                    // newboxs = g.map(x=>)
                    break;
            }
            draw.render()
        },
        setProp(name, event) {
            if ('group ungroup'.indexOf(name) > -1) {
                name == 'group' ? draw.group() : draw.ungroup()
                return;
            }
            if ('undo redo'.indexOf(name) > -1) {
                name == 'redo' ? history.redo() : history.undo()
                return;
            }
            let obj = draw.getSelected()
            let g = draw.getSelectedGroup()
            if (!obj && !g) return
            //group 
            if ('groupleft groupright grouptop groupbottom groupcenterv groupcenterh groupvertical grouphorizontal'.indexOf(name) > -1) {
                this.setAlign(name, g)
                return
            }
            // static
            if ('flipx flipy lock remove text'.indexOf(name) > -1) {
                switch (name) {
                    case 'flipx': draw.toggleFlipX(); break;
                    case 'flipy': draw.toggleFlipY(); break;
                    case 'lock': draw.toggleLock(); break;
                    case 'remove': draw.removeSelected(); break;
                    case 'text': event && draw.setFontFamily(event); break;
                }
                return
            }
            // text
            if ('font opacity border lineheight charspacing'.indexOf(name) > -1 && event) {
                let v = event.target.value
                switch (name) {
                    case 'font': draw.setFontSize(v); break;
                    case 'opacity': draw.setOpacity(v); break;
                    case 'border': draw.setStrokeWidth(v); break;
                    case 'lineheight': draw.setLineHeight(v); break;
                    case 'charspacing': draw.setCharSpacing(v); break;
                }
                return
            }
            // text toggle
            if (obj.text) {
                this.toggleProp(name)
            }
            // draw.setActiveProp(name, value)
            this.updateJosn()
        },
        addPage(i) {
            i = i || 1  //for 里面使用闭包传入默认为1
            //在聚焦的下面插入
            //   let index = this.pages.indexOf(this.pages.filter(x => x.active)[0]) + 1
            //   this.pages.splice(index, 0, { active: false })

            //强暴插入
            let id = `canvas-${new Date() * 1}`
            this.pages.push({ active: false, id: id })
            setTimeout((id) => {
                let canvas = new qdraw.Canvas(id)
                this.draws.push(canvas)
            }, 300, id);
        },
        removePage(index) {
            let deleteIndex = index || this.activeIndex

            if (this.pages.length > 1) {
                this.pages.splice(deleteIndex, 1)
                this.draws.splice(deleteIndex, 1)
                this.activeIndex = deleteIndex - 1
                this.updateJosn()
            }

        },
        setStyle(name, value) {
            draw.setActiveStyle(name, value > 0 ? parseFloat(value) : value)
            this.updateJosn()
        },
        excue(type) {
            switch (type) {
                case 'cut': document.execCommand('cut'); break;
                case 'copy': document.execCommand('copy'); break;
                case 'paste': let obj = JSON.parse(draw.copyData); this.canvas.loadObjFormJson(obj); break;
                case 'delete': draw.removeSelected(); break;
                case 'lock': draw.toggleLock(); break;
                case 'group': draw.group(); break;
                case 'ungroup': draw.ungroup(); break;
                case 'moveup': draw.bringForward(); break;
                case 'movedown': draw.sendBackwards(); break;
                case 'movefront': draw.bringToFront(); break;
                case 'moveback': draw.sendToBack(); break;
            }
            this.updateJosn()
            // console.log(this.activeObj)
        },
        setColor(name, value) {
            switch (name) {
                case 'pagebg': draw.setCanvasBgColor(value); break;
                case 'fill': draw.setFill(value); break;
                case 'stroke': draw.setStroke(value); break;
                case 'objbg': draw.setBgColor(value); break;
                case 'textbg': draw.setTextBgColor(value); break;
            }
            this.updateJosn()
        },
        addShape(options) {
            if (!options || !options.type) return;
            switch (options.type) {
                case 'rect': draw.addRect(options); break;
                case 'circle': draw.addCircle(options); break;
                case 'triangle': draw.addTriangle(options); break;
                case 'line': draw.addLine(options); break;
                case 'polygon': draw.addPolygon(options); break;
            }
            this.updateJosn()
        },
        addComponent(shap, item) {
            switch (shap) {
                case 'svg': if (item.src) draw.addShapeByUrl(item.src); else if (item.data) draw.loadSVG(item.data, item.options); break;
                case 'Line': draw.addLine(); break
                case 'Circle': draw.addCircle(); break
                case 'Triangle': draw.addTriangle(); break
                case 'Polygon': draw.addPolygon(); break
                case 'Rect': draw.addRect(); break
                case 'Textbox': draw.addIText({ text: '双击修改文字', fontSize: 14, left: 20, top: 20 }); break
                case 'video': draw.createVideo(item.src); break
                case 'image': draw.addImage({ src: item.src, }); break;
            }
            this.showImgBox = false
            this.showSvgBox = false
            this.showVideoBox = false
            setTimeout(() => { this.updateJosn() }, 1000);
        },
        getObjStyle() {
            draw.getObjStyle()
        },
        save(type) {
            type == 'pdf' ? '' : ''
        },
        initPPT(data) {
            try {
                if (typeof data != 'object' || this.draws.length <= 0) return;
                data.map((j, i) => {
                    if (i == 0) {
                        this.draws[i].loadFromJSON(j, () => {
                            this.draws[i].isPreview = this.isPreview
                            this.draws[i].renderAll()
                        })
                    } else {
                        setTimeout(() => {
                            let id = `canvas-${new Date() * 1}`
                            this.pages.push({ active: false, id: id })
                        }, 100 * i);
                        setTimeout((i, j) => {
                            let canvas = new qdraw.Canvas(this.pages[i].id)
                            this.draws.push(canvas)
                        }, 300 * i, i, j);
                        setTimeout((i, j) => {
                            //   console.log(i, 'index')
                            this.draws[i].loadFromJSON(j, () => {
                                this.draws[i].isPreview = this.isPreview
                                this.draws[i].renderAll()
                            })
                        }, 500 * i, i, j);
                    }
                })
            } catch (e) {
                console.log(e.message)
            }
        },
        callback() {
            let obj = draw.getSelected()
            let group = draw.getSelectedGroup()
            this.props = obj ? (obj.text ? this.propText : this.static) : (group ? this.group : [])
            if (obj) {
                let { fontSize, opacity, strokeWidth, lineHeight, charSpacing } = obj
                this.fontSize = fontSize, this.opacity = opacity,
                    this.strokeWidth = strokeWidth, this.lineHeight = lineHeight,
                    this.charSpacing = charSpacing
            }
        }
    },
    mounted() {
        // let id = `canvas-${new Date() * 1}`
        // this.pages.push({ active: true, id: `canvas-${new Date() * 1}` })
        let canvas = new qdraw.Canvas(this.pages[0].id);
        // console.log('isPreview', this.isPreview)
        canvas.isPreview = this.isPreview        
        this.canvas = canvas
        draw.init(canvas, this.callback)

        history.init(canvas)
        this.draws.push(canvas)
    },

}
</script>

<style lang="less">
@import './assets/edit.less';
</style>
