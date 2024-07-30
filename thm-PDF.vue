<template>
  <div id="procesarPDF">
    <div v-if="(showProgressBar && loadedRatio > 0)" class="centrado" >
      <div class="progress-container">
        <span class="pl-2 progress-text">Carga:
          <div class="ml-2 progress-bar" :style="progressBarStyle" >
            <span>{{ Math.floor(loadedRatio * 100) }}%</span>
          </div>
        </span>
      </div>
    </div>

    <thmPDFprint :toPrint="directToPrint" :pdfSrc="pdfSrc"></thmPDFprint>
    <thm-modal v-if="!directToPrint"
      :flatten="flatten"
      fullscreen
      centered
      id="modalPdf"
      ref="modalPdf"
      size="xl"
      modal-class="p-0"
      headerClass="p-0"
      footerClass="p-0"
      v-model="showPdfModal"
      @cancel="onCancel"
      @hidden="onHiddenModal"
      @close="onCloseModal"
    >
      <div ref="divPdf" >
        <span v-if="pdfScroll === null">
          <!-- Vista de inicialización o mensaje de carga -->
        </span>
        <span v-else-if="pdfScroll === true">
          <pdf
            v-for="currentPageSc in numPages"
            :key="currentPageSc"
            :id="`page_${currentPageSc}`"
            class="pdfPage"
            :page="currentPageSc"
            :src="pdfSrc"
            @error="loadPdfError"
            @loaded="documentLoaded(`page_${currentPageSc}`, currentPageSc)"
            @password="password"
          />
          <div v-show="false">
            <pdf ref="pdfAllDoc" v-show="false" :src="pdfSrc"/>
          </div>
        </span>
        <span v-else>
          <pdf
            class="pdfPage"
            ref="pdfAllDoc"
            :page="currentPage"
            :src="pdfSrc"
            @error="loadPdfError"
            @loaded="documentLoaded('pdfAllDoc', currentPage)"
            @password="password"
          />
        </span>
      </div>

      <template #modal-header>
         <b-row class="w-100" cols="12">
          <b-col cols="2" align-self="start" align-v="center" class="text-left">
            <h5 class="text-primary pl-2 pb-0 pt-2">
              <span>Página: <code>{{currentPage}}/{{numPages}}</code></span>
            </h5>
          </b-col>

          <b-col align-h="center" align-v="center" class="text-center p-0">
            <div class="text-primary m-0" :class="{'pt-2':rowsSeleccionadasCount==1}">
              <div v-if="rowsSeleccionadasCount>1">Documentos Concatenados:
                <code>{{rowsSeleccionadasCount}}</code>
                <i id="infoConcat" aria-hidden="true" class="helpIconPaginated ml-3 far fa-question-circle cursor-pointer" v-b-popover.hover.html="'Información Concatenados'"></i>
                <b-popover
                  triggers="hover"
                  placement="bottom"
                  boundary="window"
                  custom-class="popoverInfoConcat"
                  target="infoConcat"
                >
                  <template v-slot:title>
                    <div class="text-muted">
                      {{ rowsSeleccionadasCount }} Documentos Concatenados
                    </div>
                  </template>
                  <b-table
                    class="customTable mb-1"
                    small
                    striped
                    sticky-header
                    outlined
                    head-variant="light"
                    :items="docDetails"
                    :fields="[
                      {'key':'tipoDoc','label':'Tipo de Documento'},
                      {'key':'fileName','label':'Nombre del Documento'},
                      {'key':'fecha','label':'Creación'},
                      {'key':'cuit','label':'Demandado'},
                      {'key':'pages','label':'más Info'}
                    ]"
                  >
                    <template #cell(pages)="data">
                      <div style="white-space: nowrap;">De Página {{data.item.pageMerger}}° a {{(data.item.pageMerger+data.item.pages-1)}}°</div>
                      <div>{{ data.item.pages }} <span v-if="data.item.pages==1">Página</span><span v-else>Páginas</span>
                        <span class="p-0 m-0 cursor-pointer" @click="changeInputPageNumber(data.item.pageMerger)">
                            <svg
                              role="link" class="rota-horizontal" style="background-color:#ffffff;margin-right:4px;"
                              width="20" height="20" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
                            >
                              <image width="20" height="20" xlink:href="data:image/png;base64,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"/>
                            </svg>
                        </span>
                      </div>
                    </template>
                  </b-table>
                </b-popover>
              </div>
              <span v-if="rowsSeleccionadasCount>1">Documento Actual: <code>[<b>{{clientDocView.pageMerger}}</b>-<b>{{(clientDocView.pageMerger+clientDocView.pages-1)}}</b>]/{{numPages}}</code></span>
              <i id="infoFile" aria-hidden="true" class="helpIconPaginated ml-3 far fa-question-circle cursor-pointer" v-b-popover.hover.html="'Información del Archivo'"></i>
              <b-popover
                  triggers="hover"
                  placement="bottom"
                  boundary="window"
                  target="infoFile"
                >
                <template v-slot:title>
                  <template v-if="rowsSeleccionadasCount>1" >
                    <div class="titleSlot">{{clientDocView.pos}}° Documento
                      <span class="small text-muted">
                      de {{ rowsSeleccionadasCount }} Documentos Concatenados
                      </span>
                    </div>
                    <div class="small text-muted" v-if="clientDocView.pageMerger!==(clientDocView.pageMerger+clientDocView.pages-1)">
                      Desde Página {{clientDocView.pageMerger}} y hasta Página {{(clientDocView.pageMerger+clientDocView.pages-1)}}
                    </div>
                  </template>
                  <template v-else>
                    Documento
                  </template>
                </template>
                <div class="p-2">
                  <div>Tipo Doc: <code>{{clientDocView.tipoDoc}}</code></div>
                  <div>Nombre: <code>{{clientDocView.fileName}}</code></div>
                  <div>Creación: <code>{{clientDocView.fecha}}</code></div>
                  <div>CUIT / CUIL / CDI Demandado: <code>{{clientDocView.cuit}}</code></div>
                </div>
              </b-popover>
            </div>
          </b-col>
          <b-col cols="2" align-h="end" class="d-none d-sm-block d-md-block">
          </b-col>
          <button v-if="!flatten" type="button" aria-label="Cerrar" class="close pt-1 pb-0 mr-2 m-0" @click="closeManual">×</button>
        </b-row>
      </template>

      <template #modal-footer="{ cancel }">
        <b-row class="w-100 m-0" align-v="center">
          <b-col v-if="numPages>1" align-self="start" class="p-0">
            <b-form-checkbox size="sm" class="ml-1 mt-2 mb-auto" :checked="pdfScroll" @change="changeScrollToPaginated" switch  style="flex-flow: nowrap;">
              <strong>{{ pdfScroll===true?'Scroll':'Paginado' }}</strong>
            </b-form-checkbox>
          </b-col>
          <b-col v-if="numPages>1" align-self="center" class="text-center p-0">
            <b-button-group v-if="pdfScroll===false" size="sm">
              <b-button :disabled="currentPage==1" variant="secondary" @click="changeInputPageNumber(currentPage-1)"> &lt; </b-button>
              <b-input type="number" :min="1" :max="numPages" :step="1" :value="currentPage" @change="changeInputPageNumber">{{numPages}}</b-input>
              <b-button :disabled="currentPage==numPages" variant="secondary" @click="changeInputPageNumber(currentPage+1)"> &gt; </b-button>
            </b-button-group>
          </b-col>
          <b-col align-self="end" class="text-right p-0">
            <b-button-group size="sm" >
              <b-button variant="light" v-b-popover.hover.top="`tecla ALT izquierda`" @click="resetPanzoom">
                <span>zoom: {{zoom}}%</span>
              </b-button>
              <b-button variant="secondary" v-b-popover.hover.top="`Descargar`" @click.prevent="downloadItem()">
                <b-icon icon="download" aria-hidden="true" font-scale="2"></b-icon>
              </b-button>
              <b-dropdown size="xs" variant="secondary" toggle-class="text-decoration-none" no-caret v-b-popover.hover.top="`Imprimir`">
                <template #button-content>
                  <b-icon icon="printer-fill" aria-hidden="true" font-scale="2"></b-icon>
                </template>
                <b-dropdown-item @click="$refs.pdfAllDoc.print()">Imprimir TODO</b-dropdown-item>
                <b-dropdown-item v-if="numPages>1" @click="$refs.pdfAllDoc.print(100,[currentPage])">Imprimir Pag. {{currentPage}}</b-dropdown-item>
              </b-dropdown>
            </b-button-group>
          </b-col>
        </b-row>
      </template>

    </thm-modal>
  </div>
</template>

<script setup>

import {ref, unref, computed, watch, onMounted, onUnmounted, inject, nextTick}       from 'vue';
import thmModal                   from "@/components/comunes/thm-modal.vue";
import thmPDFprint                from "@/components/comunes/thm-PDF-print.vue";
import {localStore}               from '@/plugins/localStorage';
import {formatoFechaISO2String}   from '@/plugins/gestorFecha';
import {showError}                from "@/plugins/common";
import pdfMerger                  from 'pdf-merger-js';
import panzoom                    from 'panzoom';
import '/public/pdfjs/pdf.worker.js';
import pdf from '@jbtje/vite-vue3pdf';

const props = defineProps({
  srcBlobPDF:         { type: Array, default: () => [] },
  initProcess:        { type: Boolean, default: false },
  isDownloadPDF:      { type: Boolean, default: false },
  isPrintPDF:         { type: Boolean, default: false },
  flatten:            { type: Boolean, default: false }   // aplanar, no usar b-modal!!!
});

const emit = defineEmits(['cancel', 'closeModal', 'actionMsgProcess']);

const axiosRequest = inject('axiosRequest');

const pdfScroll = ref(null);
const switchScrollPaginated = ref(100); // si hay mas de 100 Páginas se hace paginado por default, sino scroll
const showPdfModal = ref(false);
const pdfUrl = ref('');
const pdfSrc = ref(null);
const currentPage = ref(1);
const numPages = ref(0);
const loadedRatio = ref(0);
const docDetails = ref([]);
const listTipoDocumento = ref({});
const rowsSeleccionadasCount = ref(1);
const instancePanzoom = ref(null);
const zoom = ref('100');
const observers = ref({});
const acumuladoPage = ref(0);
const showProgressBar = ref(false);
const divPdf = ref(null);
const modalBody = ref(null);
const setScrollPage = ref(null);
const directToPrint = ref(false);

let pdfScrollChangePageObserver = null;
let timerId = null;

const clientDocView = computed(() =>
  docDetails.value.find(
    (doc) => doc.pageMerger <= currentPage.value && currentPage.value <= doc.pageMerger + doc.pages - 1
  )
);

const progressBarStyle = computed(() => ({
  width: `${Math.floor(loadedRatio.value * 100)}%`
}));

onMounted(() => {
  localStore('tipoDocumento', null, 'byIntPk').then((resolve) => {
    if (resolve)
      for (const element of resolve) {
        if (String(element.pk)) listTipoDocumento.value[element.pk.toString()] = element.description.toString();
      }
  }).catch((error) => {
    throw error;
  });
});

onUnmounted(() => {
  onHiddenModal();
});

watch(
  () => props.initProcess, (newValue, oldValue) => {
    if (newValue !== oldValue && newValue === true) {
      actionsProcessPDF();
    }
  },
  { deep: true }
);

function loadPdfError(error) {
  onCancel();
  showError({ message: error.message });
  showPdfModal.value = false;
}

function progress(status) {
  const ratio = status.loaded / status.total;
  loadedRatio.value = Math.min(ratio, 1);
  if (loadedRatio.value === 1) {
    clearTimeout(timerId);
    timerId = setTimeout(() => {
      showProgressBar.value = false;
      loadedRatio.value=0;
      actionMsgProcess(false);

      if (props.isPrintPDF) {
        listenToPrint();
      }
    }, 1000);
  }
}

function listenToPrint () {
  onCancel();
  onHiddenModal();
}

function password(updatePassword, reason) {
  updatePassword(prompt('password is "test"'));
}

function changeScrollToPaginated(pdfScrollValue) {
  actionMsgProcess(true);
  pdfScroll.value = null;
  nextTick(() => {
    initializeCommon();
    pdfSrc.value = pdf.createLoadingTask(pdfUrl.value, { onProgress: progress });
    pdfScroll.value = pdfScrollValue;
    if (pdfScroll.value) {
      setScrollPage.value = currentPage.value
    }
  });
}

function changeInputPageNumber(n) {
  console.warn("Ir a página ",n);

  modalBody.value.focus();
  const intN = Number(n);
  if (!intN || intN < 1 || !numPages.value || intN > numPages.value) {
    return;
  }
  currentPage.value = intN;

  if (pdfScroll.value) {

    // primero llevo al inicio!
    modalBody.value.scrollTo({
      top: 0,
      behavior: 'instant',
    });

    // scroll to page head!
    const rect = document.querySelector(`#page_${intN}`).getBoundingClientRect();
    nextTick(() => {
      modalBody.value.scrollTo({
        top: rect.top,
        behavior: 'smooth',
      });
    });

  } else {
    nextTick(() => {
      modalBody.value.scrollTo({
        top: 0,
        behavior: 'auto',
      });
    });
  }

}

function parseContentDisposition(disposition) {
  const result = {};
    disposition.split(';').forEach(part => {
    const [key, value] = part.split('=').map(s => s.trim());
    if (key && value) {
      result[key] = value.replace(/(^"|"$)/g, ''); // Remove any surrounding double quotes
    }
  });
  return result;
}

function getFileDetails(response) {
  const fileDetails = {filename:'',idTipoDocumento:null,demandadoCuit:'',fechaCreacion:''};
  const myHeaders = new Headers(response.headers);
  if (myHeaders.has('content-disposition')) {
    const fileDetailsTemp = parseContentDisposition(myHeaders.get('content-disposition'));
    if (fileDetailsTemp.filename){
      fileDetails.filename = fileDetailsTemp.filename;
    }
    if (fileDetailsTemp.idTipoDocumento){
      fileDetails.idTipoDocumento = parseInt(fileDetailsTemp.idTipoDocumento);
    }
    if (fileDetailsTemp.fechaCreacion){
      fileDetails.fechaCreacion = formatoFechaISO2String(fileDetailsTemp.fechaCreacion, true);
    }
    if (fileDetailsTemp.demandadoCuit){
      fileDetails.demandadoCuit = fileDetailsTemp.demandadoCuit;
    }
  }
  return fileDetails;
}

async function addDoc(merger, response, pos) {

  await merger.add(URL.createObjectURL(new Blob([response.data], { type: response.headers['content-type'] }))).then(() => {
    if (props.isDownloadPDF && rowsSeleccionadasCount.value > 1) {
      return;
    }
    let fileDetails = getFileDetails(response);
    let detalleDoc = {
      fecha:      fileDetails.fechaCreacion,
      cuit:       fileDetails.demandadoCuit,
      pos:        pos,
      tipoDoc:    listTipoDocumento.value[fileDetails.idTipoDocumento],
      fileName:   fileDetails.filename,
      pages:      null,
      pageMerger: null,
    };

    const pageCount = getPageCount(merger);
    if (acumuladoPage.value === 0) {
      acumuladoPage.value   = pageCount;
      detalleDoc.pages      = acumuladoPage.value;
      detalleDoc.pageMerger = 1;
      docDetails.value.push(detalleDoc);
    } else {
      detalleDoc.pages      =  pageCount - acumuladoPage.value;
      detalleDoc.pageMerger =  acumuladoPage.value + 1;
      acumuladoPage.value   = pageCount;
      docDetails.value.push(detalleDoc);
    }
    numPages.value = pageCount;

  }).catch((error) => {
    throw error;
  });

}

async function alFinalHacerGenSrcPDF(merger) {
  const mergedPdfBuffer = await merger.saveAsBuffer();
  pdfUrl.value = URL.createObjectURL(new Blob([mergedPdfBuffer], { type: 'application/pdf' }));
  pdfSrc.value = pdf.createLoadingTask(pdfUrl.value, { onProgress: progress });
}

function initializeCommon() {
  observers.value = {};
  if (pdfScrollChangePageObserver) {
    pdfScrollChangePageObserver.disconnect();
  }
  pdfScrollChangePageObserver = new IntersectionObserver(observerCallback, {
    root: modalBody.value,
    rootMargin: '0px',
    threshold: 0.1,
  });
}

async function actionsProcessPDF() {

  directToPrint.value=false;
  showPdfModal.value = false;
  rowsSeleccionadasCount.value = props.srcBlobPDF.length;

  if (rowsSeleccionadasCount.value == 0) {
    showError({ message: 'No se Detecto Documento' });
    onHiddenModal();
    onCancel();
    return;
  }

  currentPage.value = 1;
  acumuladoPage.value = 0;
  docDetails.value = [];
  actionMsgProcess(true);

  let cancelRequest = false;
  const merger = new pdfMerger();

  for (let i = 0; i < rowsSeleccionadasCount.value; i++) {

    console.warn('URL DOC: ', props.srcBlobPDF[i]);
    await axiosRequest('get', props.srcBlobPDF[i], null, null, null, 'blob').then(async (response) => {
      if (!response) {
        throw new Error({ message: 'No hay Respuesta', name: 'Error Interno' });
      }
      await addDoc( merger, response, i+1).catch((error) => {
        throw error;
      });

      loadedRatio.value=i/rowsSeleccionadasCount.value;

      if (i === rowsSeleccionadasCount.value - 1) {

        if (props.isDownloadPDF) {
          await alFinalHacerSavePDF(merger).catch((error) => {
            throw error;
          });
        } else {
          await alFinalHacerGenSrcPDF(merger).catch((error) => {
            throw error;
          });

          if (props.isPrintPDF) {
            directToPrint.value=true;
          } else {
            pdfScroll.value = (numPages.value >= switchScrollPaginated.value ? false : true);
            nextTick(() => {
              showPdfModal.value = true;
              nextTick(() => {
                modalBody.value = document.querySelector('#modalPdf > div.modal-dialog > div.modal-content > div.modal-body');
                initializeCommon();
              });
            });
          }

        }
        actionMsgProcess(false);
      }
    }).catch((error) => {
      cancelRequest = true;
      showError(error);
      actionMsgProcess(false);
      onHiddenModal();
      onCancel();
    });

    if (cancelRequest) {
      // cancel request
      actionMsgProcess(false);
      break;
    }
  }
}

function getPageCount(merger) {
  let dCount = { pageCount: 1 };
  if (merger._doc) {
    dCount = merger._doc;
  } else if (merger.doc) {
    dCount = merger.doc;
  }
  return dCount.pageCount;
}

function observerCallback(entries) {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      currentPage.value = Number(entry.target.id.replace('page_', ''));
    }
  });
}

function documentLoaded(type, currentPage) {
  if (pdfScroll.value && !observers.value[currentPage]) {
    // docscroll
    pdfScrollChangePageObserver.observe(document.querySelector(`#page_${currentPage}`));
    observers.value[currentPage] = true;
  }
  if (currentPage < numPages.value) {
    return;
  }
  nextTick(() => {
    zoomPdf();

    // seteo la hoja cuando es scroll y se hace cambio de paginado a scroll.
    // el valor setScrollPage.value solo se setea al hacer ese cambio por eso se blanquea, es un detector.
    if (setScrollPage.value) {
      nextTick(() => {
        setTimeout(() => {
          changeInputPageNumber(setScrollPage.value);
          setScrollPage.value = null;
        },1500);
      });
    }

  });
}

function zoomPdf() {

  const divPdfElement = unref(divPdf);
  if (!divPdfElement) {
    showError({ message: 'Element #divPdf not found' });
    return;
  }

  if (instancePanzoom.value) {
    instancePanzoom.value.dispose(); // Clean up existing instance if any
  }

  instancePanzoom.value = panzoom(divPdfElement, {
    maxZoom: 2,
    minZoom: 0.1,
    zoomSpeed: 0.05,
    transformOrigin: { x: 0.5, y: 0.5 },
    beforeWheel(e) {
      // allow wheel-zoom only if altKey is down. Otherwise - ignore
      return !e.altKey;
    },
  }).on('zoom', (e) => {
    zoom.value = (e.getTransform().scale * 100).toFixed(0);
  });
}

function resetPanzoom() {
  if (instancePanzoom.value) {
    instancePanzoom.value.moveTo(0, 0);
    instancePanzoom.value.zoomAbs(0, 0, 1);
  }
}

function getFileNameForDownload() {
  return rowsSeleccionadasCount.value > 1 ? 'DocumentosConcatenados.pdf' : docDetails.value[0].fileName;
}

async function alFinalHacerSavePDF(merger) {
  // direct save, no visualiza
  const fileName = getFileNameForDownload();
  await merger.save(fileName.replace('.pdf', '')).catch((error) => {
    throw error;
  });
  setTimeout(() => {
    showProgressBar.value = false;
    loadedRatio.value=0;
  }, 300);
  closeManual();
  onHiddenModal();
}

function downloadItem() {
  // botón download
  const link = document.createElement('a');
  link.href = pdfUrl.value;
  link.download = getFileNameForDownload();
  link.click();
}

function printCurrentPage() {
  document.querySelector(`#page_${currentPage.value}`).print();
}

function actionMsgProcess(doit) {
  if (doit && !showProgressBar.value) {
    showProgressBar.value = true;
  }
  emit('actionMsgProcess', doit);
}

function onCancel() {
  emit('cancel');
}

function onHiddenModal() {
  showPdfModal.value = false;
  zoom.value = '100';
  URL.revokeObjectURL(pdfUrl.value);
  pdfUrl.value = '';
  pdfSrc.value = null;
  observers.value = {};
  if (instancePanzoom.value) {
    instancePanzoom.value.dispose();
    instancePanzoom.value = null;
  }
  if (pdfScrollChangePageObserver) {
    pdfScrollChangePageObserver.disconnect();
    pdfScrollChangePageObserver = null;
  }
}

function onCloseModal() {
  emit('closeModal');
}

function closeManual() {
  onCancel();
  showPdfModal.value = false;
}
</script>
<style>
  .popover.popoverInfoConcat {
      max-width: 900px!important;
  }
  .pdfPage canvas, .pdfPage .annotationLayer {
      width: 100% !important;
      height: auto !important;
  }
</style>
<style scoped>
  .customTable {
      border: ridge;
  }
  .helpIconPaginated:hover {
      color: rgb(46 227 3);
      opacity: .8;
      text-decoration: none;
      cursor: help;
  }
  .pdfPage {
    background:white;
    display:block;
    margin: 0 auto;
    margin-bottom:0.5cm;
    box-shadow:0 0 0.5cm rgba(0,0,0,0.5);
  }
  .btn {
    text-transform: none;
    font-size: 12px;
    min-height: 33px;
  }
  .centrado{
    z-index: 1100;
    position: absolute;
    width: 400px;
    top: 50vh;
    left: 0;
    right: 0;
    bottom: 0;
    margin: auto;
  }
  .progress-bar {
    background-color: green;
    color: white;
  }
  .progress-container {
    border: double;
    padding: 1px;
  }
  .progress-text {
    background-color: white;
    display: flex;
    flex-wrap: nowrap;
  }
</style>