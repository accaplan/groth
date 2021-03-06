<template lang="pug">
  section.album-viewer.relative.flex.justify-between.bg-off-white.h-screen.overflow-hidden.lg-border-t.border-grey-darkest.cursor-pointer(v-touch:swipe="onSwipe")
    //- divider
    //- .hidden.sm-block.h-screen.absolute.w-1x2.right-0.top-0.pointer-events-none.border-l.border-grey-lighter

    //- VERSO
    figure.h-screen.relative.hidden.sm-block(v-if="verso && !isPortrait")
      transition(name="leaf")
        component(:is="verso.type", :data="verso.data", :key="verso.index", @click="onLeafClick('verso')")
      //- caption
      transition(name="leaf")
        leaf-caption.absolute.w-full.z-10.top-0.left-0.h-50vh.sm-h-screen.translx-100.bg-black(v-if="caption === 'verso'", :leaf="verso.data.primary", @close="closeCaption")
      //- caption toggle
      nav.absolute.w-full.left-0.bottom-0.text-center.text-xs
        button.p-6.inline-block.text-xs.icon-elipsis-after(@click.stop="openCaption('verso')", v-show="hasCaption('verso')")
      //- pg number
      //- .absolute.p-6.left-0.bottom-0.text-xs {{verso.index + 1}}

    //- recto
    figure.h-screen.relative(v-if="recto", ref="recto")
      transition(name="leaf")
        component(:is="recto.type", :data="recto.data", :key="recto.index", :isRecto="true", @click.native="$event => onLeafClick('recto', $event)")
      //- caption
      transition(name="leaf")
        leaf-caption.absolute.w-full.z-10.top-0.left-0.h-50vh.lg-h-screen.bg-black(v-if="caption === 'recto'", :leaf="recto.data.primary", @close="closeCaption", :class="{'translx_-100': !isPortrait}")
      //- caption toggle
      nav.absolute.top-0.right-0.lg-w-full.lg-top-auto.lg-bottom-0.text-center.text-xs
        button.p-6.inline-block.text-xs.icon-elipsis-after(@click.stop="openCaption('recto')", v-show="hasCaption('recto')")
      //- pg number
      //- .absolute.p-8.sm-p-6.right-0.bottom-0.w-full.lg-w-auto.text-xs.text-center
        span {{recto.index + 1}}
</template>

<script>
import titleleaf from './Viewer__TitleLeaf'
import leaf from './Viewer__Leaf'
import leafCaption from './Viewer__Leaf__Caption'
import { getRandomInt } from '@/utils'
import _get from 'lodash/get'
import _shuffle from 'lodash/shuffle'
const newLeaf = (type, data, i) => { return { type: type, data: data, index: i } }
export default {
  name: 'Viewer',
  components: { titleleaf, leaf, leafCaption },
  props: {
    slices: { type: Array, default: () => [] },
    title: { type: Array, default: () => ['', ''] }
  },
  data () {
    return {
      leaves: null,
      verso: null,
      recto: null,
      isPortrait: window.innerWidth < window.innerHeight,
      afterResize: null,
      caption: null,
      autoplay: null
    }
  },
  computed: {
    activeIndexes () {
      const ii = leaf => !leaf ? -1 : leaf.index
      return [ii(this.verso), ii(this.recto)]
    }
  },
  methods: {
    hasCaption (side) {
      const title = _get(this[side], 'data.primary.title')
      return title && title.length > 0
    },
    closeCaption () {
      this.caption = null
      // this.play()
    },
    onLeafClick (side = 'verso', event, isPrev) {
      // exit captions ?
      if (this.caption) return this.closeCaption()
      // next/prev
      const prev = isPrev || (event && (event.x < this.$refs.recto.offsetWidth * 0.35))
      this.next(side, prev)
      this.pause() // this.play(8000)
    },
    openCaption (side) {
      this.pause()
      this.caption = side
    },
    getNextIndex () {
      return getRandomInt(0, this.leaves.length)
    },
    next (side, reverse) {
      if (this.isPortrait) {
        // PORTRAIT - looped next/prev
        let i = this[side].index
        const ll = this.leaves.length
        i = reverse ? i - 1 < 0 ? ll - 1 : i - 1 // prev
          : i + 1 === ll ? 0 : i + 1 // next
        this[side] = newLeaf('leaf', this.leaves[i], i)
      } else {
        // LANDSCAPE - random inactive slide
        const index = this.getNextIndex()
        const work = this.leaves[index]
        const isActv = this.activeIndexes.includes(index)
        if (isActv) return this.next(side)
        this[side] = newLeaf('leaf', work, index)
      }
    },
    onResize () {
      clearTimeout(this.afterResize)
      this.afterResize = setTimeout(() => {
        this.isPortrait = window.innerWidth < window.innerHeight
        this.setLeaves()
      }, 200)
    },
    setLeaves (intro) {
      this.leaves = this.leaves || _shuffle(this.slices)
      if (this.isPortrait) {
        // PORTRAIT: disable verso, set recto
        this.verso = null
        this.recto = this.recto || newLeaf('leaf', this.leaves[0], 0)
      } else if (intro) {
        // LND:INTRO - set title cards
        this.verso = newLeaf('titleleaf', { title: this.title, theme: 'black' }, -1)
        this.recto = newLeaf('titleleaf', { title: this.title, theme: 'black' }, -2)
      } else {
        // PRT > LND - reset
        this.next('verso')
        this.next('recto')
      }
    },
    onSwipe (dir) {
      if (dir === 'left' || dir === 'right') {
        const prev = dir === 'right'
        this.onLeafClick('recto', null, prev)
        // ipad landscape
        if (this.verso) this.onLeafClick('verso', null, prev)
      }
    },
    play (interval = 6000) {
      clearTimeout(this.autoplay)
      this.autoplay = setTimeout(() => {
        if (this.isPortrait) return this.next('recto')
        const sides = ['verso', 'recto']
        this.next(sides[Math.round(Math.random())])
        this.play()
      }, interval)
    },
    pause () {
      clearTimeout(this.autoplay)
    }
  },
  created () {
    this.setLeaves(true)
  },
  mounted () {
    this.play(4000)
    window.addEventListener('resize', this.onResize)
  },
  destoryed () {
    window.removeEventListener('resize', this.onResize)
  }
}
</script>

<style>
.leaf-leave-active{transition:opacity 150ms;}
.leaf-enter-active{transition:opacity 150ms 50ms;}
.leaf-enter, .leaf-leave-to{opacity:0;}
</style>

<style>
.album-viewer figure{
  flex:1 0 50%;
  /* show dots on hover */
  @media (hover:hover) {
    & nav {
      opacity:0;
      transition:opacity 500ms;
    }
    &:hover nav {
      opacity: 1;
    }
  }
}
.icon-elipsis-after:after{
  content: '';
  display: inline-block;
  font-size:16px;
  height:0.5em;
  width:1.5em;
  background-image:url('~@/assets/icon-bullet.svg');
  background-size:auto 100%;
}
</style>
