<script setup>
import { computed, onUnmounted, ref } from "vue";
import { Download, ImagePlus, RefreshCw, Trash2, Upload } from "@lucide/vue";

const fileInput = ref(null);
const sourceFile = ref(null);
const outputBlob = ref(null);
const previewUrl = ref("");
const dimensions = ref(null);
const quality = ref(90);
const isDragging = ref(false);
const isConverting = ref(false);
const errorMessage = ref("");

const outputName = computed(() => {
  if (!sourceFile.value) return "converted.jpg";
  return sourceFile.value.name.replace(/\.(heic|heif)$/i, "") + ".jpg";
});

const formatBytes = (bytes) => {
  if (!Number.isFinite(bytes) || bytes === 0) return "0 B";
  const units = ["B", "KB", "MB", "GB"];
  const index = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1);
  const value = bytes / 1024 ** index;
  return `${value.toFixed(index === 0 ? 0 : 1)} ${units[index]}`;
};

const sizeDifference = computed(() => {
  if (!sourceFile.value || !outputBlob.value) return "";
  const percent = Math.round((1 - outputBlob.value.size / sourceFile.value.size) * 100);
  return percent >= 0 ? `${percent}% smaller` : `${Math.abs(percent)}% larger`;
});

const clearPreview = () => {
  if (previewUrl.value) URL.revokeObjectURL(previewUrl.value);
  previewUrl.value = "";
  outputBlob.value = null;
  dimensions.value = null;
};

const readDimensions = (url) => new Promise((resolve, reject) => {
  const image = new Image();
  image.onload = () => resolve({ width: image.naturalWidth, height: image.naturalHeight });
  image.onerror = reject;
  image.src = url;
});

const convert = async (file = sourceFile.value) => {
  if (!file || isConverting.value) return;

  errorMessage.value = "";
  clearPreview();
  sourceFile.value = file;
  isConverting.value = true;

  try {
    const { heicTo, isHeic } = await import("heic-to/csp");
    const extensionMatches = /\.(heic|heif)$/i.test(file.name);
    if (!extensionMatches && !(await isHeic(file))) {
      throw new Error("Choose a valid HEIC or HEIF image.");
    }

    const converted = await heicTo({
      blob: file,
      type: "image/jpeg",
      quality: quality.value / 100,
    });
    const jpeg = Array.isArray(converted) ? converted[0] : converted;
    if (!(jpeg instanceof Blob)) throw new Error("The image could not be converted.");

    outputBlob.value = jpeg;
    previewUrl.value = URL.createObjectURL(jpeg);
    dimensions.value = await readDimensions(previewUrl.value);
  } catch (error) {
    clearPreview();
    errorMessage.value = error.message || "The image could not be converted.";
  } finally {
    isConverting.value = false;
  }
};

const selectFile = (event) => {
  const [file] = event.target.files;
  if (file) convert(file);
  event.target.value = "";
};

const dropFile = (event) => {
  isDragging.value = false;
  const [file] = event.dataTransfer.files;
  if (file) convert(file);
};

const clear = () => {
  clearPreview();
  sourceFile.value = null;
  errorMessage.value = "";
};

onUnmounted(clearPreview);
</script>

<template>
  <div class="page">
    <header class="site-header">
      <div class="shell header-row">
        <a class="brand" href="/" aria-label="HEIC to JPG home">
          <span class="brand-mark image-mark" aria-hidden="true">JPG</span>
          <span>
            <strong>HEIC to JPG</strong>
            <small>chleb.app</small>
          </span>
        </a>
        <span class="privacy"><span class="privacy-dot"></span>Runs locally in your browser</span>
      </div>
    </header>

    <main class="shell main-content">
      <div class="intro">
        <p class="eyebrow">IMAGE UTILITY</p>
        <h1>HEIC to JPG converter</h1>
        <p>Turn iPhone photos into compatible JPG files. Your images never leave your device.</p>
      </div>

      <section class="converter" aria-labelledby="converter-title">
        <h2 id="converter-title" class="visually-hidden">HEIC to JPG converter</h2>
        <div class="converter-strip">
          <span>HEIC / HEIF</span>
          <span class="strip-line" aria-hidden="true"></span>
          <span>JPG</span>
        </div>

        <div class="image-workspace">
          <input
            ref="fileInput"
            class="visually-hidden"
            type="file"
            accept=".heic,.heif,image/heic,image/heif"
            @change="selectFile"
          />

          <button
            v-if="!outputBlob"
            class="drop-zone"
            :class="{ dragging: isDragging }"
            type="button"
            :disabled="isConverting"
            @click="fileInput.click()"
            @dragenter.prevent="isDragging = true"
            @dragover.prevent="isDragging = true"
            @dragleave.prevent="isDragging = false"
            @drop.prevent="dropFile"
          >
            <span class="drop-icon" aria-hidden="true">
              <ImagePlus v-if="!isConverting" :size="28" />
              <RefreshCw v-else class="spin" :size="28" />
            </span>
            <strong>{{ isConverting ? "Converting image..." : "Drop a HEIC photo here" }}</strong>
            <span>{{ isConverting ? "This can take a moment for large photos." : "or click to choose a file" }}</span>
            <small>HEIC or HEIF · processed on this device</small>
          </button>

          <div v-else class="result-grid">
            <div class="preview-wrap">
              <img :src="previewUrl" :alt="`Preview of ${outputName}`" />
            </div>
            <div class="result-details">
              <p class="result-kicker">CONVERSION COMPLETE</p>
              <h3>{{ outputName }}</h3>
              <dl>
                <div>
                  <dt>Original</dt>
                  <dd>{{ formatBytes(sourceFile.size) }}</dd>
                </div>
                <div>
                  <dt>JPG</dt>
                  <dd>{{ formatBytes(outputBlob.size) }}</dd>
                </div>
                <div v-if="dimensions">
                  <dt>Dimensions</dt>
                  <dd>{{ dimensions.width }} × {{ dimensions.height }}</dd>
                </div>
                <div>
                  <dt>Difference</dt>
                  <dd class="result-positive">{{ sizeDifference }}</dd>
                </div>
              </dl>
              <a class="button primary download-button" :href="previewUrl" :download="outputName">
                <Download :size="18" aria-hidden="true" />
                Download JPG
              </a>
            </div>
          </div>

          <p v-if="errorMessage" class="conversion-error" role="alert">{{ errorMessage }}</p>
        </div>

        <div class="toolbar image-toolbar">
          <label class="quality-control" for="quality">
            <span>JPG quality</span>
            <input id="quality" v-model.number="quality" type="range" min="50" max="100" step="5" :disabled="isConverting" />
            <strong>{{ quality }}%</strong>
          </label>
          <div class="actions">
            <button class="button secondary" type="button" :disabled="!sourceFile || isConverting" @click="clear">
              <Trash2 :size="18" aria-hidden="true" />
              Clear
            </button>
            <button v-if="outputBlob" class="button secondary" type="button" :disabled="isConverting" @click="convert()">
              <RefreshCw :size="18" aria-hidden="true" />
              Convert again
            </button>
            <button v-else class="button primary" type="button" :disabled="isConverting" @click="fileInput.click()">
              <Upload :size="18" aria-hidden="true" />
              Choose file
            </button>
          </div>
        </div>
      </section>
    </main>

    <footer>
      <div class="shell footer-row">
        <p>chleb.app</p>
        <p>Private conversion with no uploads.</p>
      </div>
    </footer>
  </div>
</template>
