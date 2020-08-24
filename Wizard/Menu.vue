<template>
  <div class="row">
    <div class="col-12">
      <breadcrumb :page="page"
                  @page-change="page = $event"/>
    </div>
    <div class="col-12 flip-cards-container">
      <squareOption 
        v-for="(option, index) in optionList"
        :key="index"
        :option="option"/>
    </div>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import squareOption from "@/components/common/menu/SquareOption";
import breadcrumb from "@/components/common/page/TitleBreadcrumb";
import store from "@/store/index";

export default {
  name: "Menu",
  components: {
    squareOption,
    breadcrumb
  },
  data() {
    return {
      page: {
        icon: "fas fa-history",
        title: "PEI",
        routes: ["Inicio", "PEI"]
      }
    };
  },
  computed:{
    ...mapState(['navigateToViewRoute','navigateToViewDisplay'])
    ,
    optionList() {
      return store.getters.getUserSitemap
                  .find(x => x.title == 'PEI').children
                  .map(item => ({ title: item.title, 
                                  description: item.description, 
                                  detail: item.detail, 
                                  icon: item.iconClasses,
                                  mainImage: item.mainImagePath,
                                  hoverImage: item.mainImageHoverPath,
                                  needSelectedStudent: item.needSelectedStudent,
                                  route: item.naviationPath
                                 }));
    }
  },
  watch:{
    navigateToViewDisplay(){
      if(this.navigateToViewDisplay===true){
          this.$router.push({ path: this.navigateToViewRoute });
      }
    }
  }
};
</script>

<style scoped>
.flip-cards-container {
  display: flex;
  flex-wrap: wrap;
}

.flip-cards-container > div {
  min-height: 200px;
}
</style>
