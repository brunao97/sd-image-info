<script setup>
import { ref } from "vue";
import exifr from "exifr";
import { nanoid } from "nanoid";
import ComfyPrompt from "../components/ComfyPrompt.vue";
import DefaultPrompt from "../components/DefaultPrompt.vue";
import IconCheck from "../components/icons/IconCheck.vue";
import IconExclamation from "../components/icons/IconExclamation.vue";
import FileUploader from "../components/FileUploader.vue";
import Card from "../components/Card.vue";
import Modal from "../components/Modal.vue";
import ImageSelector from "../components/ImageSelector.vue";
import IconCancel from "../components/icons/IconCancel.vue";
import IconDownload from "../components/icons/IconDownload.vue";

import {
  saveFile,
  getFileName,
  removeExif,
  saveImage,
} from "../components/helpers";

let rShowMessage = ref(null);

let rImages = ref(null);
let rSelected = ref(null);
let rShowSavePictureModal = ref(false);

let rPictureFilename = ref(null);

function getStringFromBytes(uint8Array) {
  const decoder = new TextDecoder("utf-8");
  const filteredUint8Array = Uint8Array.from(
    uint8Array.filter((byte) => byte !== 0)
  );

  return decoder.decode(filteredUint8Array);
}
async function getImageData(file) {
  let params = null;
  let type = null;
  let id = nanoid();

  const filename = getFileName(file.name);

  const imageURL = URL.createObjectURL(file);

  return new Promise((resolve) => {
    exifr.parse(file, true).then((res) => {
      if (res.userComment) {
        params = getStringFromBytes(res.userComment).replace("UNICODE", "");
        type = "default";
      } else if (res.parameters) {
        params = res.parameters;
        type = "default";
      } else if (res.workflow) {
        // comfyui
        params = res;
        type = "comfyui";
      } else {
        type = "notFound";
      }

      resolve({ params, type, imageURL, id, filename, file });
    });
  });
}

function handleFile(fileList) {
  let files = Array.from(fileList);

  const data = files.map((file) => getImageData(file));

  Promise.all(data).then((d) => {
    if (d.length === 1) {
      rSelected.value = d[0];
    } else {
      rImages.value = d;
    }
  });
}

function showCopyMessage() {
  rShowMessage.value = "Copied to clipboard!";
  setTimeout(() => {
    rShowMessage.value = null;
  }, 1000);
}
function handleCopy() {
  showCopyMessage();
}

function handleSave() {
  const { params, type, filename } = rSelected.value;
  if (type === "comfyui") {
    saveFile(params.workflow, filename, "json");
  } else if (type === "default") {
    saveFile(params, filename, "txt");
  }
}

async function handleSavePicture() {
  rPictureFilename.value = rSelected.value.filename;
  rShowSavePictureModal.value = true;
}

async function handleDownloadPicture() {
  rShowSavePictureModal.value = false;
  const blob = await removeExif(rSelected.value.file);
  saveImage(blob, rPictureFilename.value);
}

async function handleDownloadAll() {
  if (rImages.value && rImages.value.length > 1) {
    for (const image of rImages.value) {
      const blob = await removeExif(image.file);
      saveImage(blob, image.filename + "_no_meta_data");
    }
  }
}
</script>

<template>
  <div
    v-if="rShowMessage"
    class="fixed flex z-50 items-center w-full max-w-xs p-4 space-x-4 text-gray-500 bg-white divide-x divide-gray-200 rounded-lg shadow top-5 right-5 dark:text-gray-400 dark:divide-gray-700 space-x dark:bg-gray-800"
  >
    <IconCheck />
    {{ rShowMessage }}
  </div>

  <div v-if="rSelected" class="flex flex-col h-full items-center">
    <div
      v-if="rSelected.type === 'notFound'"
      class="w-full h-full flex items-center"
    >
      <div
        class="flex flex-col items-center justify-center w-full h-64 border-2 border-slate-300 rounded-lg bg-slate-50 dark:bg-slate-700 dark:border-slate-600"
      >
        <IconExclamation
          class="!w-10 !h-10 text-slate-500 dark:text-slate-400"
        />
        <span class="font-bold p-4 text-slate-500 dark:text-slate-400">
          Prompt not found
        </span>
        <button
          @click="rSelected = null"
          class="inline-flex items-center py-2.5 px-5 sm:mr-2 mb-2 text-sm font-medium text-slate-900 focus:outline-none bg-white rounded-lg border border-slate-200 hover:bg-slate-100 hover:text-blue-700 focus:z-10 focus:ring-4 focus:ring-slate-200 dark:focus:ring-slate-700 dark:bg-slate-800 dark:text-slate-400 dark:border-slate-600 dark:hover:text-white dark:hover:bg-slate-700"
        >
          Return
        </button>
      </div>
    </div>

    <DefaultPrompt
      v-if="rSelected.type === 'default'"
      :params="rSelected.params"
      v-model:filename="rSelected.filename"
      @save="handleSave"
      @copy="handleCopy"
      @clear="rSelected = null"
      @save-picture="handleSavePicture"
    />
    <ComfyPrompt
      v-if="rSelected.type === 'comfyui'"
      :params="rSelected.params"
      v-model:filename="rSelected.filename"
      @save="handleSave"
      @copy="handleCopy"
      @clear="rSelected = null"
      @save-picture="handleSavePicture"
    />

    <div
      class="fixed z-50 w-full bottom-24 left-0 flex justify-center items-center"
    >
      <ImageSelector
        v-if="rImages?.length > 1"
        @select="(rImage) => (rSelected = rImage)"
        :rImages="rImages"
        :rSelected="rSelected"
      />
    </div>
  </div>

  <div v-else-if="rImages?.length">
    <button
      @click="rImages = null"
      class="inline-flex items-center py-2.5 px-5 sm:mr-2 mb-2 text-sm font-medium text-gray-900 focus:outline-none bg-white rounded-lg border border-gray-200 hover:bg-gray-100 hover:text-blue-700 focus:z-10 focus:ring-4 focus:ring-gray-200 dark:focus:ring-gray-700 dark:bg-gray-800 dark:text-gray-400 dark:border-gray-600 dark:hover:text-white dark:hover:bg-gray-700"
    >
      <IconCancel />

      Clear
    </button>
    <div v-if="rImages?.length > 1">
      <button
        @click="handleDownloadAll"
        class="inline-flex items-center text-white bg-blue-700 hover:bg-blue-800 focus:outline-none focus:ring-4 focus:ring-blue-300 font-medium rounded-lg text-sm px-5 py-2.5 text-center sm:mr-2 mb-2 dark:bg-blue-600 dark:hover:bg-blue-700 dark:focus:ring-blue-800"
      >
        <IconDownload />
        Download All
      </button>
    </div>
    <div class="columns-2 md:columns-3 lg:columns-4 mt-1">
      <div v-for="rImage in rImages" :key="rImage.id" class="inline-block mb-2">
        <Card :url="rImage.imageURL" @click="() => (rSelected = rImage)" />
      </div>
    </div>
  </div>

  <div v-else class="w-full h-full flex items-center">
    <FileUploader @files-change="handleFile" />
  </div>

  <Modal
    :show="rShowSavePictureModal"
    @close="rShowSavePictureModal = false"
    @save="handleDownloadPicture"
  >
    <label
      for="prompt"
      class="block mb-2 text-sm font-medium text-gray-900 dark:text-gray-400"
      >Filename</label
    >
    <textarea
      class="w-full p-3 dark:bg-black dark:text-gray-400"
      :value="rPictureFilename"
      @input="rPictureFilename = $event.target.value"
    >
    </textarea>
  </Modal>
</template>
