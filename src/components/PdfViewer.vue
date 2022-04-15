<template>
  <div>
    <div id="outerContainer">
      <div class="toolbar">
        <button @click="prev">上一页</button>
        <input
          type="number"
          class="page-number-input"
          v-model="pageNum"
          min="1"
          :max="pageCount"
          @blur="webViewerPageNumberChanged(pageNum)"
        />
        <span class="page-num"> / {{ pageCount }}页</span>
        <button @click="next">下一页</button>
        <button @click="minus">缩小</button>
        <button @click="addscale">放大</button>
        <button @click="print">打印</button>
        <input
          class="pdf-choose"
          type="file"
          name="选择pdf文件"
          id="pdfChoose"
          @change="pdfChange"
        />
      </div>
      <div id="viewerContainer">
        <div id="viewer" class="pdfViewer"></div>
      </div>
    </div>
    <div id="printContainer"></div>
  </div>
</template>
<script>
import "pdfjs-dist/web/pdf_viewer.css";
import { GlobalWorkerOptions, getDocument } from "pdfjs-dist/legacy/build/pdf";
import {
  PDFLinkService,
  PDFFindController,
  PDFScriptingManager,
  PDFSinglePageViewer,
  PDFViewer,
} from "pdfjs-dist/legacy/web/pdf_viewer";
import PDFJSWorker from "pdfjs-dist/legacy/build/pdf.worker.entry";
GlobalWorkerOptions.workerSrc = PDFJSWorker;
import { PDFPrintServiceFactory, PDFPrintService } from "./pdf_print_service"; //"pdfjs-dist/lib/web/pdf_print_service";
import { AutomationEventBus, EventBus } from "./event_utils";

export default {
  data() {
    return {
      CMAP_URL: "",
      CMAP_PACKED: true,
      DEFAULT_URL: "/3.pdf",
      ENABLE_XFA: true,
      SEARCH_FOR: "",
      SANDBOX_BUNDLE_SRC: null,
      container: null,
      viewer: null,
      eventBus: null,
      pdfLinkService: null,
      pdfFindController: null,
      pdfScriptingManager: null,
      pdfViewer: null,

      pdfDocument: null, // pdfjs读取的页面信息
      pageNum: 0, // 当前页数
      pageCount: 0, // 总页数
      pageRendering: false, // 当前页面是否在渲染中
      pageNumPending: null, // 将要进行渲染的页面页数
      scale: 1, // 放大倍数
      maxscale: 5, // 最大放大倍数
      minscale: 0.3, // 最小放大倍数

      printService: null,
      printContainer: null,
      pagesOverview: [],
      l10n: null,
      _printResolution: 0,
      scratchCanvas: null,
    };
  },
  mounted() {
    this.initPDF();
    this.getPdfDocument(this.DEFAULT_URL);
  },
  methods: {
    initPDF() {
      // Some PDFs need external cmaps.
      //
      const CMAP_URL = "../../node_modules/pdfjs-dist/cmaps/";
      const CMAP_PACKED = true;

      const DEFAULT_URL = "/3.pdf";

      const ENABLE_XFA = true;
      const SEARCH_FOR = ""; // try "Mozilla";

      const SANDBOX_BUNDLE_SRC = "../../node_modules/pdfjs-dist/legacy/build/pdf.sandbox.js";//require("pdfjs-dist/legacy/build/pdf.sandbox.js");

      const container = document.getElementById("viewerContainer");
      const viewer = document.getElementById("viewer");

      // const eventBus = new EventBus();
      const isInAutomation = true;
      const eventBus = isInAutomation
        ? new AutomationEventBus()
        : new EventBus();

      // (Optionally) enable hyperlinks within PDF files.
      const pdfLinkService = new PDFLinkService({
        eventBus,
        externalLinkTarget: 0,
        externalLinkRel: "noopener noreferrer nofollow",
        ignoreDestinationZoom: false,
      });

      // (Optionally) enable find controller.
      const pdfFindController = new PDFFindController({
        eventBus,
        linkService: pdfLinkService,
      });

      // (Optionally) enable scripting support.
      // const pdfScriptingManager = new PDFScriptingManager({
      //   eventBus,
      //   sandboxBundleSrc: SANDBOX_BUNDLE_SRC,
      // });

      const pdfViewer = new PDFViewer({
        container,
        viewer,
        eventBus,
        linkService: pdfLinkService,
        findController: pdfFindController,
        // scriptingManager: pdfScriptingManager,
        enableScripting: true, // Only necessary in PDF.js version 2.10.377 and below.
        renderer: "canvas",
        l10n: this.l10n,
        textLayerMode: 1,
        annotationMode: 2,
        imageResourcesPath: "./images/",
        enablePrintAutoRotate: true,
        useOnlyCssZoom: false,
        maxCanvasPixels: 16777216,
        enablePermissions: false,
      });
      pdfLinkService.setViewer(pdfViewer);
      // pdfScriptingManager.setViewer(pdfViewer);

      eventBus.on("pagesinit", function () {
        // We can use pdfViewer now, e.g. let's change default scale.
        pdfViewer.currentScaleValue = "page-width";

        // We can try searching for things.
        if (SEARCH_FOR) {
          eventBus.dispatch("find", { type: "", query: SEARCH_FOR });
        }
      });

      this.CMAP_URL = CMAP_URL;
      this.CMAP_PACKED = CMAP_PACKED;
      this.DEFAULT_URL = DEFAULT_URL;
      this.ENABLE_XFA = ENABLE_XFA;
      this.SEARCH_FOR = SEARCH_FOR;
      this.SANDBOX_BUNDLE_SRC = SANDBOX_BUNDLE_SRC;
      this.container = container;
      this.viewer = viewer;
      this.eventBus = eventBus;
      this.pdfLinkService = pdfLinkService;
      this.pdfFindController = pdfFindController;
      // this.pdfScriptingManager = pdfScriptingManager;
      this.pdfViewer = pdfViewer;

      const _boundEvents = Object.create(null);

      _boundEvents.beforePrint = this.beforePrint.bind(this);
      _boundEvents.afterPrint = this.afterPrint.bind(this);

      eventBus._on("beforeprint", _boundEvents.beforePrint);
      eventBus._on("afterprint", _boundEvents.afterPrint);
      eventBus._on("scalechanging", this.webViewerScaleChanging);

      _boundEvents.windowBeforePrint = () => {
        eventBus.dispatch("beforeprint", { source: window });
      };
      _boundEvents.windowAfterPrint = () => {
        eventBus.dispatch("afterprint", { source: window });
      };
      window.addEventListener("beforeprint", _boundEvents.windowBeforePrint);
      window.addEventListener("afterprint", _boundEvents.windowAfterPrint);
    },
    getPdfDocument(pdfResource) {
      // Loading document.
      const loadingTask = getDocument({
        url: pdfResource,
        cMapUrl: this.CMAP_URL,
        cMapPacked: this.CMAP_PACKED,
        enableXfa: this.ENABLE_XFA,
      });
      loadingTask.promise.then(
      pdfDocument => {
        this.pdfDocument = pdfDocument;
        this.pageNum = this.pdfViewer.currentPageNumber;
        this.pageCount = pdfDocument.numPages;
        // Document loaded, specifying document for the viewer and
        // the (optional) linkService.
        this.pdfViewer.setDocument(pdfDocument);
        // this.pdfScriptingManager.setDocument(pdfDocument);

        this.pdfLinkService.setDocument(pdfDocument, null);
      },
      reason => {
        let key = "loading_error";
        if (reason instanceof InvalidPDFException) {
          key = "invalid_file_error";
        } else if (reason instanceof MissingPDFException) {
          key = "missing_file_error";
        } else if (reason instanceof UnexpectedResponseException) {
          key = "unexpected_response_error";
        }
      }
    );
      // (async () => {
      //   const pdfDocument = await loadingTask.promise;
      //   this.pdfDocument = pdfDocument;
      //   this.pageNum = this.pdfViewer.currentPageNumber;
      //   this.pageCount = pdfDocument.numPages;
      //   // Document loaded, specifying document for the viewer and
      //   // the (optional) linkService.
      //   this.pdfViewer.setDocument(pdfDocument);

      //   this.pdfLinkService.setDocument(pdfDocument, null);
      // })();

      // loadingTask.promise.then(function (pdfDocument) {
      //   // Document loaded, specifying document for the viewer and
      //   // the (optional) linkService.
      //   pdfViewer.setDocument(pdfDocument);

      //   pdfLinkService.setDocument(pdfDocument, null);
      // });
    },
    pdfChange(e) {
      /**
       * 上传pdf文件，并转为pdfjs可用的文件数据格式
       */
      const fileReader = new FileReader();
      fileReader.readAsDataURL(e.target.files[0]);
      fileReader.onload = (fileLoaded) =>
        this.loadPdfJs(fileLoaded.target.result);
    },
    loadPdfJs(pdfResource) {
      this.getPdfDocument(pdfResource);
    },
    addscale() {
      /**
       * 放大
       */
      if (this.scale >= this.maxscale) {
        return;
      }
      this.scale += 0.1;
      this.pdfViewer.increaseScale(this.scale);
    },
    minus() {
      /**
       * 缩小
       */
      if (this.scale <= this.minscale) {
        return;
      }
      this.scale -= 0.1;
      this.pdfViewer.decreaseScale(this.scale);
    },
    prev() {
      this.pdfViewer.previousPage();
      this.pageNum = this.pdfViewer.currentPageNumber;
    },
    next() {
      this.pdfViewer.nextPage();
      this.pageNum = this.pdfViewer.currentPageNumber;
    },
    webViewerPageNumberChanged(value) {
      const pdfViewer = this.pdfViewer;
      // Note that for `<input type="number">` HTML elements, an empty string will
      // be returned for non-number inputs; hence we simply do nothing in that case.
      if (value !== "") {
        this.pdfLinkService.goToPage(value);
      }
    },
    webViewerScaleChanging(evt) {
      this.scale = evt.scale;
    },
    print() {
      if (this.pageCount === 0) {
        return;
      }
      window.print();
    },

    beforePrint() {
      if (this.printService) {
        // There is no way to suppress beforePrint/afterPrint events,
        // but PDFPrintService may generate double events -- this will ignore
        // the second event that will be coming from native window.print().
        return;
      }

      this.printContainer = document.getElementById("printContainer");
      this.pagesOverview = this.pdfViewer.getPagesOverview();
      this._printResolution = 150; //printResolution || 150;
      this.scratchCanvas = document.createElement("canvas");
      const optionalContentConfigPromise =
        this.pdfDocument.getOptionalContentConfig();

      const pagesOverview = this.pagesOverview;
      const printContainer = this.printContainer;
      const printResolution = this._printResolution;

      const printService = PDFPrintServiceFactory.instance.createPrintService(
        this.pdfDocument,
        pagesOverview,
        printContainer,
        printResolution,
        optionalContentConfigPromise,
        this.l10n
      );
      this.printService = printService;

      printService.layout();
    },
    afterPrint() {
      if (this.printService) {
        this.printService.destroy();
        this.printService = null;
      }
    },
    rotatePages(delta) {
      this.pdfViewer.pagesRotation += delta;
      // Note that the thumbnail viewer is updated, and rendering is triggered,
      // in the 'rotationchanging' event handler.
    },
  },
};
</script>
<style lang="scss">
// * {
//   padding: 0;
//   margin: 0;
// }

// html {
//   height: 100%;
//   width: 100%;
// }

// body {
//   height: 100%;
//   width: 100%;
//   background-color: #808080;
// }
.toolbar {
  height: 32px;
  padding: 0px 16px;
  background: rgba(103, 103, 103, 1);
  display: flex;
  align-items: center;
  justify-content: center;
  .page-number-input {
    width: 32px;
    border: none;
    border-radius: 2px;
    padding: 2px 4px;
  }
  .page-num {
    color: white;
  }
  button {
    margin: 0 10px;
    cursor: pointer;
  }
  .pdf-choose {
    margin-left: 100px;
    color: white;
  }
}
#viewerContainer {
  overflow: auto;
  position: absolute;
  top: 32px;
  right: 0;
  bottom: 0;
  left: 0;
  background-color: rgba(0, 0, 0, 0.2);
}

@page {
  margin: 0;
}

#printContainer {
  display: none;
}

@media print {
  /* General rules for printing. */
  body {
    background: rgba(0, 0, 0, 0) none;
  }

  /* Rules for browsers that don't support mozPrintCallback. */
  #sidebarContainer,
  #secondaryToolbar,
  .toolbar,
  #loadingBox,
  #errorWrapper,
  .textLayer,
  .canvasWrapper {
    display: none;
  }
  #viewerContainer {
    overflow: visible;
  }

  #mainContainer,
  #viewerContainer,
  .page,
  .page canvas {
    position: static;
    padding: 0;
    margin: 0;
  }

  .page {
    float: left;
    display: none;
    border: none;
    box-shadow: none;
    background-clip: content-box;
    background-color: rgba(255, 255, 255, 1);
  }

  .page[data-loaded] {
    display: block;
  }

  /* Rules for browsers that support PDF.js printing */
  body[data-pdfjsprinting] #outerContainer {
    display: none;
  }
  body[data-pdfjsprinting] #printContainer {
    display: block;
  }
  #printContainer {
    height: 100%;
  }
  /* wrapper around (scaled) print canvas elements */
  #printContainer > .printedPage {
    page-break-after: always;
    page-break-inside: avoid;

    /* The wrapper always cover the whole page. */
    height: 100%;
    width: 100%;

    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }

  #printContainer > .xfaPrintedPage .xfaPage {
    position: absolute;
  }

  #printContainer > .xfaPrintedPage {
    page-break-after: always;
    page-break-inside: avoid;
    width: 100%;
    height: 100%;
    position: relative;
  }

  #printContainer > .printedPage canvas,
  #printContainer > .printedPage img {
    /* The intrinsic canvas / image size will make sure that we fit the page. */
    max-width: 100%;
    max-height: 100%;

    direction: ltr;
    display: block;
  }
}
</style>