<template>
  <app-layout>
    <encyclopedia-link-list slot="left" :entry-list="elementList" title="Elements" />
    <div slot="main">
      <router-view />
    </div>
    <encyclopedia-link-list slot="right" :entry-list="conceptList" title="Concepts" />
  </app-layout>
</template>

<script lang="ts">
import { Vue, Component } from 'vue-property-decorator'
import { elementNameList, conceptNameList } from './loadData'
import AppLayout from '@/components/AppLayout.vue'
import EncyclopediaArticle from '@/components/EncyclopediaPage/EncyclopediaArticle.vue'
import EncyclopediaLinkList from '@/components/EncyclopediaPage/EncyclopediaLinkList.vue'

// FIXME: Code smell move to interfaces
interface IEntryList {
  name: string
  ready: boolean
}

@Component({
  components: {
    AppLayout,
    EncyclopediaArticle,
    EncyclopediaLinkList,
  },
})
export default class Info extends Vue {
  elementList: IEntryList[] = []
  conceptList: IEntryList[] = []
  sections: IEntryList[] = []

  created(): void {
    this.elementList = elementNameList.map((entryName: string) => ({
      name: entryName,
      ready: true,
    }))
    this.conceptList = conceptNameList.map((entryName: string) => ({
      name: entryName,
      ready: true,
    }))
  }
}
</script>
