<template>
  <div id="pageContainer">
    <canvas id="pdf-container"></canvas>
    <div class="stepper">
      <div>
        <input type="button" value="Previous" @click="previousPage" />
      </div>
      <div class="info">Page {{ pageNumber }} of {{ numPages }}</div>
      <div><input type="button" value="Next" @click="nextPage" /></div>
    </div>
  </div>
</template>

<script>
import "pdfjs-dist/web/pdf_viewer.css";
import { GlobalWorkerOptions, getDocument } from "pdfjs-dist/legacy/build/pdf";
import PDFJSWorker from "pdfjs-dist/legacy/build/pdf.worker.entry";
GlobalWorkerOptions.workerSrc = PDFJSWorker;

export default {
  data() {
    return {
      url: "https://raw.githubusercontent.com/mozilla/pdf.js/ba2edeae/web/compressed.tracemonkey-pldi-09.pdf",
      numPages: 1,
      pageNumber: 1,
      pdfDocument: null,
    };
  },

  mounted() {
    this.getPdfDocument();
  },

  methods: {
    async getPdfDocument() {
      getDocument(this.url).promise.then(
        (pdf) => {
          this.numPages = pdf.numPages;
          this.pdfDocument = pdf;
          this.getPage(this.pageNumber);
        },
        (reason) => {
          console.error(reason);
        }
      );
    },

    getPage(pageNumer) {
      this.pdfDocument.getPage(pageNumer).then((page) => {
        var viewport = page.getViewport({ scale: 1.5 });

        var canvas = document.getElementById("pdf-container");
        var context = canvas.getContext("2d");
        canvas.height = viewport.height;
        canvas.width = viewport.width;

        var renderContext = {
          canvasContext: context,
          viewport: viewport,
        };

        var renderTask = page.render(renderContext);

        renderTask.promise.then(function () {
          console.log("Page rendered");
        });
      });
    },

    previousPage() {
      this.pageNumber =
        this.pageNumber > 1 ? this.pageNumber - 1 : this.pageNumber;
      this.getPage(this.pageNumber);
    },

    nextPage() {
      this.pageNumber =
        this.pageNumber < this.numPages ? this.pageNumber + 1 : this.pageNumber;
      this.getPage(this.pageNumber);
    },
  },
};
</script>

<style>
#pageContainer {
  margin: auto;
}

.stepper {
  display: flex;
  align-items: center;
  justify-content: center;
  position: fixed;
  top: 10px;
  right: 30px;
}

.stepper .info {
  padding-left: 10px;
  padding-right: 10px;
}
</style>
