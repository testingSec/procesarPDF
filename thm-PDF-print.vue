<template>
  <div v-if="toPrint">
    <pdf v-show="false" ref="thmPDFprint" :src="src"/>
  </div>
</template>
<script>
import {showError} from "@/plugins/common";
import '/public/pdfjs/pdf.worker.js';
import pdf from '@jbtje/vite-vue3pdf';

export default {
  name: 'thm-PDF-print',
  props: {
    toPrint:{type: Boolean,default: false},
    pdfSrc: {type: Object, default() {return {}}}
  },
  components: {
    pdf
  },
  watch: {
    toPrint: {
      handler(newValue, oldValue) {
        if (newValue!==oldValue && newValue){
          this.printer();
        }
      },
      deep: true
    },
    pdfSrc: {
      handler(newValue, oldValue) {
        if (newValue!==oldValue && newValue){
          this.src = newValue;
        }
      },
      deep: true
    }
  },
  data() {
    return {
      src: null
    }
  },
  methods: {
    printer(){
      if (!this.toPrint) return;
      if (!this.src)     return;

      this.pdfSrc.promise.then(() => {
        setTimeout(() => {
          this.$refs.thmPDFprint.print();
        },300);
      }).catch(error => {
        showError(error);
      });
    }
  }
};
</script>