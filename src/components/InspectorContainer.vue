<template>
  <div id="controls-container">
    <RouterLink :to="{ name: 'home' }" style="margin-right: 40px;">Home</RouterLink>
    <span v-if="widgets.savedData.size === 0">&lt;No Entries&gt;</span>
    <template v-else>
      <label for="entry-select">Entry</label>
      <select id="entry-select" v-model.number="selectedIdx">
        <option v-for="[i, name] of entries.entries()" :key="i" :value="i">{{ name }}</option>
      </select>
      <button @click="deleteData">Delete</button>
      <button @click="downloadData">Download</button>
      <button @click="clearData">Clear All</button>
      <button @click="generateQRCode">Generate QR Code for latest round</button>
    </template>
  </div>
  <div>
    <div class="w-full h-full" id="qrcode"></div>
  </div>
  <div class="table-container">
    <span v-if="selectedEntry === undefined">No Data</span>
    <InspectorTable v-else v-model="selectedRecords" :data="selectedEntry" />
  </div>
  <a :hidden="true" :download="entries[selectedIdx]" ref="downloadLink"></a>
</template>

<script setup lang="ts">
import QRcode from 'easyqrcodejs';
import InspectorTable from "./InspectorTable.vue";
import { useWidgetsStore } from "@/common/stores.js";

const widgets = useWidgetsStore();
let selectedIdx = $ref(0); // The index of the entry selected in the combobox

import lzma from 'lzma-js-simple'
const downloadLink = $ref<HTMLAnchorElement>();
const selectedRecords = $ref(new Set<number>());
const hasSelectedRecords = $computed(() => selectedRecords.size > 0);

const entries = $computed(() => [...widgets.savedData.keys()]); // The entries in local storage
const selectedEntry = $computed(() => widgets.savedData.get(entries[selectedIdx])); // The selected entry
const entries_2 = $computed(() => widgets.savedData.get(entries[entries.length - 1])); // The entries in local storage
const res = JSON.stringify(entries_2?.values);
const generateQRCode = () => {
  console.log(res)
  const qrCodeOptions = {
    text: res, // Change this text to the data you want in the QR code
    width: 400,
    height: 400,
    colorDark: '#000000',
    colorLight: '#ffffff',
    correctLevel: QRcode.CorrectLevel.L,
  };

  const qrcode = new QRcode(document.getElementById('qrcode'), qrCodeOptions);
  qrcode.makeCode(res);
};

// Filters records in the selected entry based on the user selection.
// If there are no records selected, the filter directly uses the given state, returning either all or no records.
const filterRecords = (state: boolean) => (selectedEntry === undefined)
  ? []
  : selectedEntry.values.filter((_v, i) => hasSelectedRecords ? (selectedRecords.has(i) === state) : state);

function deleteData() {
  if (selectedEntry === undefined) return;

  if (!confirm(`Delete ${hasSelectedRecords ? "the selected" : "all"} records in this entry permanently?`)) return;

  // Discard out the selected records
  // If there are none selected, they are all deleted
  selectedEntry.values = filterRecords(false);

  selectedRecords.clear();
}

function downloadData() {
  if (selectedEntry == undefined) return;

  if (downloadLink === undefined) return; // Make sure the link exists

  // Generate the download link for the selected records, then trigger the download
  // If there are no records selected, they will all be included in the generated file
  downloadLink.href = widgets.makeDownloadLink({ header: selectedEntry.header, values: filterRecords(true) });
  downloadLink.click();
}

function clearData() {
  if (!confirm("Clear all saved entries in local storage permanently?")) return;

  widgets.savedData.clear();
  selectedIdx = 0; // Reset selected index
}


</script>

<style>
#qrcode > :not(canvas:nth-last-child(1)){
  display: none;
}
.table-container {
  overflow: auto;
}

#controls-container>* {
  margin: 4px;
}
</style>
