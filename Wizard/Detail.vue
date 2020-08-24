<template>
  <div>
    <div class="row">
      <Master :key="reRenderTabsPei" @reRender="reRender()"/> <!--Test-->
    </div>
  </div>
</template>
<script>
import Master from "@/components/pei/Master";
export default {
  components: {
    Master
  },
  data() {
    return {
      reRenderTabsPei: 0,
      page: {
        icon: "fas fa-desktop",
        title: "I. Información del Estudiante",
        routes: ["Inicio", "PEI", "Informacion"]
      }
    };
  },
  methods: {
    reRender() {
      this.reRenderTabsPei += 1;
    }
  }
};
</script>
